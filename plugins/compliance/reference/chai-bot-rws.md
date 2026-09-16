# Running `analyze-cve` on a Coordinator/Worker Host (e.g. Chai Bot)

Guidance for running the [`analyze-cve`](../skills/analyze-cve/SKILL.md) skill on a
host that splits execution across a **coordinator** (holds credentials, talks to
Slack/Jira/GitHub) and an **isolated worker** (runs the actual Claude Code session
with only build/git tooling — no Jira access). [Chai Bot](https://github.com/openshift-eng/ship-help-bot)
Remote Workspace (RWS) pods are the motivating example, but nothing below is
specific to its internals — apply the same pattern to any host shaped this way.

## The problem

This skill's sub-skills (`jira-cve-extraction`, `report-to-jira`) assume Jira
access is available in the same session that runs the rest of the pipeline —
true for a plain Claude Code CLI with the `jira` plugin's Atlassian MCP, or for
Ambient Code. It is **not** true on a coordinator/worker host: the worker
container typically has no Jira credentials or MCP server of its own, only the
coordinator does. Calling into Phase 0.3 (JQL search), Phase 0.5 (ticket
extraction), or `report-to-jira`'s posting steps from inside such a worker will
simply fail — there's no Jira tool there to call.

## The adapter pattern

1. **Do Jira I/O on the coordinator, not the worker.** Before dispatching to the
   worker, the coordinator itself performs whatever Phase 0.3/0.5 would have
   done: run the JQL search or fetch the ticket, extract `CVE_ID`, `IMAGE_NAME`,
   `BRANCH`, and the full `jira_context` object, and check `embargo_status`.
2. **Hand the worker pre-resolved context, not a ticket key.** Pass `CVE_ID`,
   `IMAGE_NAME`, `BRANCH`, and `jira_context` into the worker's prompt so it can
   skip straight to Phase 0.7 (repository resolution) instead of attempting
   Phase 0.5 itself.
3. **Let the worker do everything that needs its own toolchain**: clone the
   repo, run `govulncheck`, call-graph analysis, apply fixes, `gh pr create`.
4. **Route the worker's output back through the coordinator.** If the worker
   session has no Jira access (the normal case here), it should return
   structured results — the report markdown, risk level, PR URL — rather than
   attempt to post them itself. The coordinator then posts the analysis
   comment, adds the `ai-cve-analyzed` label, and posts the PR-URL follow-up
   comment, using whatever Jira integration *it* has.
5. **The sub-skills' Jira calls are prose, not literal API bindings.** They're
   written against generic Atlassian MCP tool names for portability. When the
   coordinator (an LLM) reads "fetch the issue" or "add a label," it maps that
   onto whatever real Jira tool it has available — there is no naming
   convention to keep in sync with this plugin.
6. **If your host gates PR creation behind a review/approval flow**, run that
   flow on the coordinator before the worker's local commit becomes an actual
   pull request. The worker may commit locally; opening the PR is the
   coordinator's job whenever policy requires a check first.
7. **For scheduled/unattended runs**, use `--auto-approve=yes` and have the
   coordinator honor whatever completion contract your host uses for scheduled
   tasks (e.g. posting a final report) once the worker returns.

The exact tool names for any of the above are host-specific and documented by
that host, not here.

## Plugin install layout

Hosts that pre-install plugin content onto the worker's filesystem typically
write skills to the standard Claude Code project-level discovery path:

```
<workspace-root>/.claude/skills/analyze-cve/SKILL.md
```

so the worker discovers `analyze-cve` natively via project-level skill
scanning — the coordinator does not need to paste skill bodies into the
prompt, and generally does not need to name the skill explicitly either
(though it may invoke it precisely with `/compliance:analyze-cve ...` when
exact arguments matter, e.g. after prefetching Jira context per the pattern
above).

## Workspace paths

Ephemeral/remote worker hosts usually mount a persistent workspace at a fixed
path — Chai Bot RWS pods use `/workspace`. Set `AI_HELPERS_WORKSPACE` to match
your host's convention:

```bash
export AI_HELPERS_WORKSPACE="${AI_HELPERS_WORKSPACE:-/workspace}"
export REPOS_BASE="${AI_HELPERS_WORKSPACE}/.work/compliance/analyze-cve/repos"
export WORK_CVE="${AI_HELPERS_WORKSPACE}/.work/compliance/analyze-cve/${CVE_ID}"
```

Reports: `${AI_HELPERS_WORKSPACE}/.work/compliance/analyze-cve/{CVE-ID}/report.md`
