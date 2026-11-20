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

**The fix.** The root `dev` task now checks for `pulumi_config.json` and runs `task infra:deploy-dev`
once if it is missing, before `drdev` starts anything. Applied to both `Taskfile.yml` (composed) and
`.Taskfile.template` (source), so it survives `dr task compose`.

The two misleading "Please run 'dr start' first" messages in `infra/Taskfile.yaml` now name the task
that actually fixes it.

**Nothing else is changed.** The agent, its tools and the hardcoded DataRobot asset IDs in
`agent/agent/myagent.py` and `fastapi_server/app/api/v1/claims.py` are untouched; they still point at
the dataset and the two deployments in the AI Training tenant.
