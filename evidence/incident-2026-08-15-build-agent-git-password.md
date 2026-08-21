# Incident: first boot refused by `BuildAgentGitPasswordValidator`

- Date: 2026-08-15 (L4, first end-to-end boot of the minimal MySQL+ICL variant).
- Status: **resolved same day**; fix verified (application started, stack healthy,
  admin login OK).
- Raw logs: `L4-run1.log` (config-time run), application failure excerpt below.

## Symptom

`artemis-app` in a restart loop; `nginx` unhealthy (downstream only — its
upstream was dead). Application log on every attempt:

```text
APPLICATION FAILED TO START
Property: artemis.version-control.build-agent-git-password
Problem:  the build-agent git password is a value published in the Artemis
          repository, and a caller presenting it can read every repository
          without any authorization check or access-log entry
```

## Root cause chain (all links verified)

1. The Artemis image's classpath config `application-localvc.yml:14` ships the
   example value `build-agent-git-password: buildjob_password`. With the
   `localvc` profile active and nothing overriding it, that value was live.
2. `BuildAgentGitPasswordValidator`
   (`src/main/java/de/tum/cit/aet/artemis/core/config/`) is **new on
   `develop`**: absent from the local Artemis checkout of 2026-08-12
   (`ff19136a81`), present on 2026-08-15. Deploy-time uses the *newest*
   `develop` (fresh clone + `:develop` image), so the lab met the guard on its
   very first boot.
3. Guard semantics (read from source): runs under the `prod` profile on core
   **or** buildagent nodes; a `null` property is exempt (neither profile
   defines it then); a **blank** value is rejected; the published set
   `{buildjob_password, buildagent_password, artemis_admin}` is rejected;
   **using SSH for build agents does not exempt** — the password shortcut in
   `LocalVCServletService` exists regardless.
4. The lab's rendered `artemis.env` contained no override because the L2
   baseline (transformed from the admin's test7 sample) had no
   `build_agent_git_credentials` — neither does TUM's own
   `artemistests_local_vc_ci.yml` (see drift snapshot).

## Three-repo drift snapshot (2026-08-15)

| Repo | State |
|---|---|
| `ls1intum/Artemis` `develop` | Guard added within the last ~3 days (absent 08-12, present 08-15) |
| `artemis-ansible-collection` | `artemis.env.j2` supported `version_control.localvc.build_agent_git_credentials` already at our pin `346e3b4` (08-03); `main` (`8977303`, 08-13) additionally **removes** the legacy `ARTEMIS_VERSIONCONTROL_USER/PASSWORD` emission (PR #234 "remove legacy localvc build agent authentication"), adds presence-gated OIDC blocks, and fixes docker dir permissions |
| `artemis-ansible` (TUM values) | `build_agent_git_credentials` added to `artemis_prod_like_common`, `…_local_vc_jenkins`, `…_local_vc_hades` — **but not to `artemistests_local_vc_ci.yml`** as of `origin/main` today. The ICL test servers (test1/2/3/4/7/9) will meet the same guard on their next `develop` deployment |

The lab therefore reproduced a *live* cross-repo consistency window: the
application added a contract requirement, the roles library had the variable,
and the values layer had it for three groups but not the fourth. This is
precisely the class of gap the thesis's three-way consistency check
(feature model ↔ Artemis config keys ↔ ops variables) targets — observed here
on the real operational chain, caught by the lab before TUM's own ICL test
servers hit it.

## Fix applied (verified in the repo)

1. Collection pin bumped `346e3b4…` → `8977303c560a91be27214509dd07bf6170c97277`
   (delta reviewed: legacy VC-credential lines removed, OIDC blocks
   presence-gated/inert for the lab, permissions fixes; no breaking change for
   our render). `ansible-galaxy collection install --force`.
2. New generated secret `lab_build_agent_git_password` in
   `artemislocal/secrets.yml` (git-ignored).
3. `artemistests_local_vc_ci.yml` localvc block extended:

   ```yaml
   build_agent_git_credentials:
     user: "artemis-build-agent"
     password: "{{ lab_build_agent_git_password }}"
   ```

4. Config-time playbook re-run; the notified handler restarted the stack.
   Verified: `ARTEMIS_VERSIONCONTROL_BUILDAGENTGITUSERNAME/-PASSWORD` present
   in `artemis.env`, legacy `ARTEMIS_VERSIONCONTROL_USER/PASSWORD` gone,
   `Started ArtemisApp`, containers healthy, browser login OK.

## Lessons recorded

- **Default provenance is a required catalog field.** Same class as the
  `push_notification_relay` hermes-prod escape found in L3: template guards
  alone do not determine output — collection defaults and (here) *image
  classpath defaults* are two further layers. The R0 variable catalog must
  record, per variable: template guard kind, collection default, and whether
  an Artemis-side shipped default exists underneath.
- **Pin lag reproduces real ops failure windows.** Deploying moving `develop`
  against a pinned collection is exactly TUM's situation; the lab makes such
  windows observable and cheap.
- **Reproducibility option.** For evaluation runs where a stable app matters,
  pin `artemis_version`/`artemis_branch` to a release tag instead of
  `develop` (record the chosen tag in evidence); keep `develop` for
  parity/drift testing.
- Optional courtesy: give the admins a heads-up that
  `artemistests_local_vc_ci.yml` likely needs `build_agent_git_credentials`
  before their next `develop` rollout.
