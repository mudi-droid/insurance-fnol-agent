# Changes from upstream

Copy of `mitchfiorini-DR/insurance-agent-template-starter` with one fix.

## `dr run dev` now works on a fresh clone

**The problem.** `dr run dev` starts five processes, one of which is `infra` — the local tracing
dashboard and the OpenTelemetry plumbing. `infra` depends on `infra:check-use-case-id`, which exits
1 when `pulumi_config.json` is absent. That file is only created by the Pulumi `deploy-dev` step,
which normally runs inside `dr start`'s first-run wizard.

So on a fresh clone, `dr run dev` either dies outright, or a coding assistant routes around it by
starting the other four processes without `infra`. The app then runs and looks healthy — but the
agent's OTel exporter has no credentials, every span export returns `401 Unauthorized`, and the
Tracing tab stays empty. The failure is silent and costs you the governance half of any demo.

**The fix, part 1 — provision once.** The root `dev` task now checks for `pulumi_config.json` and runs `task infra:deploy-dev`
once if it is missing, before `drdev` starts anything. Applied to both `Taskfile.yml` (composed) and
`.Taskfile.template` (source), so it survives `dr task compose`.

**The fix, part 2 — create the stack.** Provisioning alone is not enough: `deploy-dev` runs
`pulumi up`, which fails on a fresh clone because no Pulumi stack exists yet. `dr start` normally
creates one interactively. `infra:deploy-dev` now calls a new `infra:ensure-stack` task first, which
selects the stack named by `PULUMI_STACK_NAME` (default `agent-application-dev`) and creates it if
it is absent.

This was found by running it, not by reading it: with part 1 alone, `dr run dev` still failed on a
fresh clone before any service started.

The two misleading "Please run 'dr start' first" messages in `infra/Taskfile.yaml` now name the task
that actually fixes it.

**The fix, part 3 — log in to a backend.** A brand-new machine or codespace has no `~/.pulumi`, so
there is no backend for stack state and `ensure-stack` fails before it can create anything.
`dr start` normally handles this. `ensure-stack` now checks `pulumi whoami` and falls back to
`pulumi login --local` when there is no backend, so a first-ever checkout works without `dr start`.

**Nothing else is changed.** The agent, its tools and the hardcoded DataRobot asset IDs in
`agent/agent/myagent.py` and `fastapi_server/app/api/v1/claims.py` are untouched; they still point at
the dataset and the two deployments in the AI Training tenant.

## Verified

Run end to end in a DataRobot codespace on 2026-11-20, from a fresh clone, driven entirely through
Build Studio's Assist tab with no manual `dr start`, no manual `deploy-dev` and no hand-created
stack:

- `dev` saw `pulumi_config.json` missing and ran `infra:deploy-dev` itself
- `ensure-stack` created the Pulumi stack
- `pulumi up` created 8 resources in about 20s and wrote `pulumi_config.json` with the trace endpoint
- `drdev` brought up all five services, every port returning HTTP 200
- the agent's OTel exporter started with both auth headers; **401 count: 0** across the whole log
- one recommendation produced a 10-span trace, 0 errors, visible in Build Studio

**Known limitation.** Two checkouts of this project on one machine share a Pulumi project name and
local state folder, so the second one keeps the default stack name and reuses the first one's stack.
`ensure-stack` now warns when it selects a stack that already holds resources; set a unique
`PULUMI_STACK_NAME` in that checkout's `.env` before running.
