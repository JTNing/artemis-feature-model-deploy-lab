# Observation: exercise-creation page freeze on `develop` (upstream, not lab)

- Date: 2026-08-15.
- Scope verdict: **excluded from lab scope** — reproduced identically on an
  official TUM test server running the same-era `develop` build.

## Symptom

On the lab instance (`https://artemis.192.168.252.2.nip.io`, `develop`):
navigating to create a programming exercise freezes the current page
(unresponsive; the create/edit page never appears). No related backend log
entries; no directly related browser-console errors. Entering the base URL in
the *same* tab loads forever; a *new* tab works normally until the same route
is visited.

## Control experiment

The identical behavior occurs on an official Artemis test server 3 (TUM-operated
deployment, TUM values, same-era `develop` build). Same app build + two
independent deployments + same failure ⇒ the deployment layer (lab values,
ansible path, VM) is exonerated; the suspect is the `develop` client build
itself.

- TODO for the record: capture the exact build identity on both sides (UI
  footer or `/management/info` shows version + git hash):
  - lab instance commit: `1e9d816bfec51932224b340672daf110491e990b`
  - test server used + commit: `d73c4173808fd7ba74722416b9f4f922542363d4`

## Mechanism hypothesis (consistent with every symptom)

A blocked browser main thread on the exercise-creation route (e.g. an
infinite loop in component initialization / change detection or editor
setup): a hung main thread logs nothing to the console, sends nothing to the
backend, blocks same-tab navigation, and leaves fresh tabs (new renderer)
working until they hit the same route.

## Disposition

- Not a lab defect; positively, reproducing upstream behavior bug-for-bug is
  a **fidelity result** for the lab.
- Does not block the L-track: L4 acceptance (boot, health, profiles, login)
  is met; L5/L6 proceed.
- Re-test after a future `develop` redeploy
  (`sudo ./artemis-docker.sh restart develop develop` pulls the newest build);
  optionally report upstream with this reproduction if it persists.
- Reinforces the evaluation-run option recorded in the incident report: pin a
  release tag for demo stability; keep `develop` for parity/drift testing.
