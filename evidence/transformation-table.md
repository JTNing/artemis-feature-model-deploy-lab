# Sample → Lab Transformation Table

- Date: 2026-08-14.
- Source being transformed: `sample-setup.txt` — admin-provided setup values for
  **artemis-test7**, verified to be a verbatim concatenation of four files of the
  public `ls1intum/artemis-ansible` repository @ `52d44e162ea4753c97ea89c7b0f7d5ec28d81645`
  (`group_vars/artemistest7.yml` + `artemistests_common_config.yml` (minus `sentry`)
  + `artemistests_local_vc_ci.yml` + `artemistests_mysql.yml`), with YAML
  indentation lost in transit. Structure was therefore restored from the public
  originals, never from the sample text.
- Consumer contract: `ls1intum/artemis-ansible-collection`, pinned via the
  `JTNing` fork. Baseline pin `346e3b4ed563b27aefe880b92e7926f24a612a12`
  (2026-08-03); bumped 2026-08-15 to
  `8977303c560a91be27214509dd07bf6170c97277` during the build-agent-password
  incident (see §10 and `incident-2026-08-15-build-agent-git-password.md`).
  Template-guard behavior referenced below was verified on the baseline pin:
  dropped blocks are presence-guarded (dropping removes the rendered keys) —
  with one verified exception, `push_notification_relay`, which carries a
  collection default and needs an explicit null (NULL-OVERRIDE); the two
  `INFO_OPERATOR*` lines are **unguarded** and therefore mandatory.
- Purpose. This table is simultaneously:
  1. evidence of the **variable-ownership classification** (which values belong
     to the variant, the host identity, the admin, or a deployment topology);
  2. the **requirements specification for the remote-ansible composer** (what it
     must generate, substitute, or leave to the admin);
  3. the reference for comparing any future chair-VM values against the lab.

## Dispositions

| Code | Meaning |
|---|---|
| KEEP | Copied verbatim from the public original |
| DUMMY | Vault lookup replaced by a locally generated random value (git-ignored `secrets.yml`) — the manual rehearsal of the composer's future vault-reference handling |
| IDENTITY | TUM-specific identity/endpoint replaced by a lab equivalent |
| DROP-MULTINODE | Removed: multinode-only; first target is single-node |
| DROP-MINIMAL | Removed: presence-gated feature/integration block not selected in the minimal variant (removal = rendered keys absent, guard verified) |
| DROP-UNUSED | Removed: not consumed by the `artemis-tests` playbook / docker path |
| NULL-OVERRIDE | Explicit null needed to defeat a collection-default fallback (the guard accepts `is not none`, but plain removal resurrects the default) |
| LAB-ADD | Added by the lab, no sample counterpart |

## 1. Per-server section (sample lines 1–19, from `artemistest7.yml`)

| Sample entry | Disposition | Lab location | Reason |
|---|---|---|---|
| `var_testserver_name: "artemis-test7"` | IDENTITY → `"artemis-local"` | `artemislocal/main.yml` | Host identity |
| `artemis_database_password: lookup(hashi_vault kv/data/artemis/test/artemistest7 # db_password)` | DUMMY | `artemislocal/secrets.yml` | Deployment-internal contract value; a self-generated random value is fully functional (both ends owned by the deployment) |
| `artemis_internal_admin_password: lookup(… # internal_admin)` | DUMMY | `artemislocal/secrets.yml` | Same class; doubles as the lab login password for `artemis_admin` |
| `artemis_jhipster_jwt: lookup(… # jhipster_jwt)` | DUMMY | `artemislocal/secrets.yml` | Same class; format constraint: long base64 (`openssl rand -base64 64`) |
| `is_multinode_install: true` | DROP-MULTINODE | — | First target is single-node docker (test2/4/6/9/10 class) |
| `artemis_node_count: 3` | DROP-MULTINODE | — | — |
| `node_id: 0` (per-server override; “prevents all nodes scheduling”, merge trick) | DROP-MULTINODE | — | Single node keeps the common `node_id: 1` (scheduling profile holder) |
| `broker:` (username / password lookup `# broker_password` / url `artemis-activemq-broker`) | DROP-MULTINODE | — | ActiveMQ broker exists only in the multinode stack |
| `artemis_jhipster_registry_usernamme: "admin"` | DROP-MULTINODE | — | Registry is multinode-only. Note: the `usernamme` typo is upstream-real (test1/3/5/7), self-consistent, and the collection defines no such variable — the eureka URL is assembled in the inventory layer |
| `artemis_jhipster_registry_password: lookup(… # registry_password)` | DROP-MULTINODE | — | `variable_checks.yml` requires it only when `is_multinode_install` |
| `artemis_eureka_urls: http://{{ …usernamme }}:{{ …password }}@artemis-jhipster-registry:8761/eureka/` | DROP-MULTINODE | — | — |

## 2. Common baseline (sample lines 21–78, from `artemistests_common_config.yml`)

| Sample entry | Disposition | Lab location | Reason |
|---|---|---|---|
| `node_id: 1` | KEEP | `artemistests_common_config.yml` | Node 1 carries the `scheduling` Spring profile |
| `is_testserver: true` | KEEP | 〃 | — |
| `artemis_server_url: "https://{{ var_testserver_name }}.artemis.cit.tum.de"` | IDENTITY → `"https://{{ var_server_hostname }}"` | 〃 | Derivation pattern kept; domain source replaced by the lab hostname variable (nip.io) |
| `artemis_email: "{{ var_testserver_name }}.aet@xcit.tum.de"` | IDENTITY → `artemis-local@thesis.invalid` | 〃 | Syntactically valid, non-routable |
| `artemis_version: develop` | KEEP | 〃 | Deploy-time image tag baseline |
| `artemis_branch: develop` | KEEP | 〃 | Compose-file source branch for the checkout |
| `artemis_internal_admin_user: artemis_admin` | KEEP | 〃 | — |
| `artemis_passkey_enabled: true` | KEEP | 〃 | Parity. Passkey UI may not function behind a self-signed cert (WebAuthn secure-context rules) — out of scope, login by password unaffected |
| `artemis_repo_basepath: "/opt/artemis/data"` | KEEP | 〃 | — |
| `artemis_tmp_directory: "/tmp"` | KEEP | 〃 | — |
| `push_notification_relay: hermes-staging…` | NULL-OVERRIDE | `artemistests_common_config.yml` (explicit null) | Guarded, but the collection **defaults** it to hermes-**prod** — plain removal resurrected the default in the L3 render; explicit null is the off-switch |
| `tum_live_base_url: https://tum.live/api/v2` | DROP-MINIMAL | — | Presence-gated TUM-Live integration (guard verified) |
| `artemis_ssh_key_password:` (null) | KEEP | `artemistests_common_config.yml` | LocalVC key-generation contract (quartet kept verbatim) |
| `artemis_ssh_key_path:` (null here; localvc group sets `/tmp`) | KEEP | 〃 | — |
| `artemis_ssh_key_name: "id_ecdsa"` | KEEP | 〃 | — |
| `artemis_ssh_priv_key_value: ""` | KEEP | 〃 | — |
| `use_docker: true` | KEEP | 〃 | Selects the docker deployment path — essential |
| `artemis_system_ram_proportion: 0.6` | DROP-UNUSED | — | Consumed only by the native `artemis.service.j2`; the docker path hardcodes `-Xmx3g` (verified 2026-08-14). Dormant on all-docker test servers — a per-template-applicability catalog lesson |
| `apollon_url: https://apollon.aet.cit.tum.de` | DROP-MINIMAL | — | Presence-gated Apollon converter feature (guard verified) |
| `artemis_operator_name: "Technical University of Munich"` | IDENTITY → `"Artemis Feature Model Thesis Lab"` | `artemistests_common_config.yml` | **Unguarded, mandatory** (`INFO_OPERATORNAME`, verified) — cannot be dropped |
| `artemis_operator_admin_name: "Stephan Krusche"` | IDENTITY → `"Junting Ning"` | 〃 | Same (mandatory) |
| `artemis_global_search_enabled: true` | KEEP | 〃 | Plain feature flag, parity |
| `lti:` (oauth_secret lookup `common/lti # oauth_secret`) | DROP-MINIMAL | — | External LTI integration + shared secret (guard verified) |
| `licenses.matlab: 28000@mlm1.rbg.tum.de` | DROP-MINIMAL | — | TUM license server (guard verified) |
| `helios:` (environment_name + 2 endpoints with lookups `common/helios`) | DROP-MINIMAL | — | Status push to TUM Helios (guard verified; also gated on `not is_multinode_install`) |
| `artemis_rate_limit:` (account_management_rpm 5 / authentication_rpm 30) | KEEP | `artemistests_common_config.yml` | Plain config, parity (presence-guarded block) |
| `deimos:` (llm base_url/model/… + api_key lookup `common/deimos`) | DROP-MINIMAL | — | External LLM service + secret (guard verified) |

## 3. Proxy section (sample lines 80–86)

| Sample entry | Disposition | Lab location | Reason |
|---|---|---|---|
| `proxy_ssl_certificate_path: /var/lib/rbg-cert/live/fullchain.pem` | IDENTITY → `/opt/lab-certs/fullchain.pem` | `artemistests_common_config.yml` | “Real object” class: nginx bind-mounts the file; lab uses a self-signed SAN cert for the nip.io hostname (rbg-cert infrastructure does not exist on the lab/bare VM) |
| `proxy_ssl_certificate_key_path: /var/lib/rbg-cert/live/privkey.pem` | IDENTITY → `/opt/lab-certs/privkey.pem` | 〃 | — |
| `firewall_hostgroup: proxy` | DROP-UNUSED | — | Consumed by the `firewall` role; `artemis-tests.yml` applies only `artemis` + `legal` |

## 4. Deployment-user section (sample lines 88–95)

| Sample entry | Disposition | Lab location | Reason |
|---|---|---|---|
| `artemis_create_deployment_user: true` | IDENTITY → `false` | `artemistests_common_config.yml` | Deploy-time automation plane not exercised yet; re-enable in phase 2 with a dedicated GitHub-Actions key |
| `artemis_deployment_user_name: deployment` | DROP-MINIMAL | — | Unused while `create` is false |
| `artemis_deployment_user_uid: 1338` | DROP-MINIMAL | — | 〃 |
| `artemis_deployment_user_public_key: lookup(kv/data/artemis/test/{name} # ssh_pub)` | DROP-MINIMAL | — | 〃 (will be the GH-Actions public key in phase 2, not a Vault ref) |
| `artemis_deployment_user_comment: "User to deploy artemis to this host"` | DROP-MINIMAL | — | 〃 |

## 5. LocalVC / LocalCI section (sample lines 97–114, from `artemistests_local_vc_ci.yml`)

| Sample entry | Disposition | Lab location | Reason |
|---|---|---|---|
| `version_control.localvc.url: "{{ artemis_server_url }}"` | KEEP | `artemistests_local_vc_ci.yml` | — |
| `version_control.localvc.ssh_key_path: /opt/artemis/data/artemis/ssh-keys` | KEEP | 〃 | Triggers the role's `generate_ssh_keys.yml` |
| `version_control.localvc.use_version_control_access_token: true` | KEEP | 〃 | — |
| `version_control.localvc.build_agent_use_ssh: true` | KEEP | 〃 | — |
| `version_control.localvc.ssh_url: ssh://git@{{ var_testserver_name }}.artemis.cit.tum.de:7921/` | IDENTITY → `ssh://git@{{ var_server_hostname }}:7921/` | 〃 | Host part only; port/scheme kept |
| `artemis_ssh_key_path: "/tmp"` | KEEP | 〃 | Verbatim TUM quirk (docker.env `ARTEMIS_SSH_KEY_PATH` mount source) |
| `continuous_integration.localci.url: "{{ artemis_server_url }}"` | KEEP | 〃 | — |
| `continuous_integration.localci.is_core_node: true` | KEEP | 〃 | Single node = core **and** agent; node-1-must-be-core check |
| `continuous_integration.localci.is_build_agent: true` | KEEP | 〃 | — |

## 6. MySQL section (sample lines 116–123, from `artemistests_mysql.yml`)

| Sample entry | Disposition | Lab location | Reason |
|---|---|---|---|
| `artemis_database_dbname: "Artemis"` | KEEP | `artemistests_mysql.yml` | Value-free config, verbatim |
| `artemis_database_host: "artemis-mysql"` | KEEP | 〃 | Compose service name |
| `artemis_database_username: "Artemis"` | KEEP | 〃 | — |
| `artemis_database_type: mysql` | KEEP | 〃 | Drives compose-file choice (`test-server-mysql-localci.yml`) |
| `artemis_database_port: 3306` | KEEP | 〃 | — |

## 7. Public-repo entries absent from the sample

| Entry | Handling | Reason |
|---|---|---|
| `sentry:` block (in public `artemistests_common_config.yml`, not in sample) | Not adopted | Admin omitted it from the sample (intent to be politely confirmed); lab does not report to TUM Sentry either way. Guard is undefined-safe (`sentry.dsn \| default('')`) |
| `weaviate_collection_prefix`, `athena:`, `theia:` (other servers' files) | Not applicable | test7 is not in those groups — its shape (MySQL + ICL, no AI) is precisely the lab's minimal variant anchor |

## 8. Lab additions (no sample counterpart)

| Entry | Location | Reason |
|---|---|---|
| `var_server_hostname: artemis.192.168.252.2.nip.io` | `artemislocal/main.yml` | Single maintenance point for the IP-derived hostname (server URL + LocalVC ssh_url) |
| `artemis_telemetry_enabled: false` | `artemistests_common_config.yml` | Collection default is `true`; the lab must not report to TUM telemetry |
| `lab_iris_dummy_secret` | `artemislocal/secrets.yml` | Pre-generated for the L7 iris cycle; referenced by the inert `artemistests_iris.yml` |
| `artemistests_postgres.yml` (copied verbatim from upstream) | group_vars | Inert until L6 membership switch (mysql → postgres) |
| `artemistests_iris.yml` (iris block only, dummy URL/secret) | group_vars | Inert until L7 membership add. Deliberately iris-only — TUM's iris group file bundles nebula + a local-LLM flag, an inventory quirk the feature model does not reproduce |

## 9. Reading this table as the composer specification

| Disposition class | Composer (R1) responsibility |
|---|---|
| KEEP | Baseline values a generated package emits for every single-node docker variant |
| DUMMY | In a real package these are **Vault references** (`vault:kv/data/artemis/test/<server>#<field>`), emitted as `lookup('hashi_vault', …)` expressions — never values; the lab's dummy substitution is the manual stand-in |
| IDENTITY | **Admin-owned inputs** the package must parameterize (hostname/URL, operator identity, certificate paths) and document in its README, never invent |
| DROP-MINIMAL | **Feature-gated blocks**: emitted if and only if the corresponding feature is selected; absence is the off-switch (presence-gating + merge semantics) |
| DROP-MULTINODE | Out of scope for the single-node mode; a future multinode mode owns them |
| DROP-UNUSED | Not consumed on this path; emitting them would be dead config (catalog records per-template applicability) |
| NULL-OVERRIDE | For deselected features whose variable carries a collection default, the composer must emit an **explicit null** — absence is not the off-switch there. Requires the catalog's default-provenance field |
| LAB-ADD | Lab-only scaffolding, except `artemis_telemetry_enabled: false`, which the composer should offer as an explicit choice |

## 10. Post-baseline deltas (2026-08-15, L4 incident)

| Entry | Disposition | Lab location | Reason |
|---|---|---|---|
| Collection pin `346e3b4` → `8977303` | PIN-BUMP | `requirements.yml` | Companion state for deploying newest `develop`: removes the legacy `ARTEMIS_VERSIONCONTROL_USER/PASSWORD` emission (PR #234), adds inert presence-gated OIDC blocks and docker permission fixes; delta reviewed and archived in the incident report |
| `version_control.localvc.build_agent_git_credentials:` (user + password) | DUMMY | `artemistests_local_vc_ci.yml` + `artemislocal/secrets.yml` (`lab_build_agent_git_password`) | Required by the new `BuildAgentGitPasswordValidator` on `develop` (prod profile; SSH does **not** exempt); overrides the image-shipped example `buildjob_password`. In a real package: Vault reference |

## Tally

KEEP 27 · DUMMY 4 · IDENTITY 9 · DROP-MULTINODE 8 · DROP-MINIMAL 11 ·
NULL-OVERRIDE 1 · DROP-UNUSED 2 · LAB-ADD 5 (incl. two inert files) ·
PIN-BUMP 1 — every line of the sample plus the post-baseline deltas is
accounted for.
