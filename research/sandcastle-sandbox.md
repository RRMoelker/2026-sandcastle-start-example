# What can an agent run inside the Sandcastle sandbox?

Research for issue #2. Source: the published npm package `@ai-hero/sandcastle@0.12.0`
(`npm pack`, read only, not executed). File references below are paths inside that tarball.

## Base image and installed tooling

The Dockerfile that `sandcastle init` writes to `.sandcastle/Dockerfile` for the Claude Code agent
(`dist/main.js`, `CLAUDE_CODE_DOCKERFILE`):

- `FROM node:22-bookworm` (Debian 12, Node 22 + npm; corepack only enabled for the Beads tracker).
- `apt-get install git curl jq`.
- With the GitHub Issues tracker, the `gh` CLI is added (`GITHUB_CLI_TOOLS`).
- Claude Code CLI via `curl -fsSL https://claude.ai/install.sh | bash`, on PATH at `/home/agent/.local/bin`.
- Runs as non-root user `agent`, UID/GID aligned with the host user (build args `AGENT_UID`/`AGENT_GID`).
- `ENTRYPOINT ["sleep", "infinity"]`: the container idles; Sandcastle `docker exec`s commands into it.
- **No browsers, no Playwright, no Chromium system libraries.** No sudo/root for the agent at runtime.

The Dockerfile is meant to be edited ("Sandbox environment (customize as needed)", README `init` section),
and rebuilt with `sandcastle docker build-image`.

## Network and package install

- The container is started with plain `docker run -d` (`dist/chunk-CP3TYXZA.js`, `startContainer`):
  no `--network none`, so it gets Docker's default bridge network with **full outbound internet**.
  `npm install`, `gh`, and the Claude API all work.
- Optional `docker({ network: ... })` attaches named Docker networks (e.g. to reach a DB container).
- **No `-p` port publishing**: a dev server started inside the sandbox is reachable only from inside
  that container (localhost), not from the host. Fine for `curl`/test runners in the same container.
- The template runs `npm install` as an `onSandboxReady` hook in every sandbox, and copies the host's
  `node_modules` into the worktree first (`copyToWorktree: ["node_modules"]`) to speed it up; the
  hook fixes platform-specific binaries (host is macOS, sandbox is Linux).

## How the agent is invoked

`claudeCode(model)` (`dist/index.js`) runs, per iteration:
`claude --print --verbose --dangerously-skip-permissions --output-format stream-json --model <model> -p -`
with the prompt on stdin. So it is headless, non-interactive, all tools allowed. Auth via
`.sandcastle/.env`: `CLAUDE_CODE_OAUTH_TOKEN` (or `ANTHROPIC_API_KEY`) and `GH_TOKEN`
(fine-grained PAT, Issues read/write + Metadata read). Defaults: idle timeout 600 s (resets on output).
Prompts can embed shell output with `` !`command` `` which is executed before the agent starts.

## `parallel-planner-with-review` template

From `dist/templates/parallel-planner-with-review/main.mts` and prompts. Outer loop up to 10 iterations:

1. **Plan** (`claude-opus-4-8`, 1 iteration, docker sandbox on the host working dir). Issue list =
   `gh issue list --state open --label Sandcastle --limit 100 ...` (`LIST_TASKS_COMMAND`, `dist/main.js`).
   **Only open issues with the label `Sandcastle` are picked up** (init offers to create the label).
   The planner builds a dependency graph (incl. "touches overlapping files") and emits a `<plan>` JSON
   of unblocked issues, branch name `sandcastle/issue-{id}`.
2. **Execute + review**, all issues concurrently (`Promise.allSettled`). Per issue `createSandbox({ branch })`:
   a git worktree on branch `sandcastle/issue-{id}` bind-mounted into its own container.
   - Implementer (`claude-sonnet-4-6`, up to 100 iterations): reads the issue via `gh issue view`, TDD
     (red-green-refactor), **must run `npm run typecheck` and `npm run test` before committing**, commit
     prefix `RALPH:`; comments on the issue if unfinished; does not close.
   - Reviewer (sonnet, 1 iteration, same sandbox/branch), only if commits were made: refactors for
     clarity against `.sandcastle/CODING_STANDARDS.md`, reruns tests, commits.
3. **Merge** (sonnet, 1 iteration): `git merge <branch> --no-edit` for each completed branch into the
   **current local branch**, resolves conflicts, runs `npm run typecheck` + `npm run test`, then
   `gh issue close <ID> --comment "Completed by Sandcastle"`. No push and no PRs: everything is local.

Implications baked into the template: the project **must** expose `npm run typecheck` and `npm run test`
scripts, and they are the only verification feedback loop the prompts mention.

## Headless browser / Playwright: practical?

Possible but not out of the box, and costly:

- The image has no browser or its system deps. `npx playwright install --with-deps` needs root (apt),
  which the `agent` user does not have at runtime; deps would have to be added to `.sandcastle/Dockerfile`
  (or base on `mcr.microsoft.com/playwright`), then the image rebuilt.
- Browser download inside each `onSandboxReady` hook would repeat per sandbox per iteration (slow);
  it belongs in the image.
- Parallel sandboxes each run their own dev server + Chromium: CPU/RAM pressure on a laptop (Docker
  Desktop VM), and arm64-vs-amd64 browser builds are a common snag.
- No port publishing, so a human cannot watch the dev server; e2e would have to run fully inside.
- Visual/UI checks are not in the prompts; the reviewer only sees the diff.

## Implications for Urenbrug

- Stack: Node 22 + npm, TypeScript, nothing native that differs between macOS host and Linux sandbox.
- Provide `npm run typecheck` (`tsc --noEmit`) and `npm run test` (`vitest run`) from day one; that is
  the contract the implementer, reviewer and merger all rely on.
- **Recommend unit + integration tests with Vitest** as the agents' verification loop: domain logic as
  pure TS, HTTP/API handlers tested in-process (e.g. via the framework's request injection or
  supertest), DOM-level component tests with `jsdom`/`happy-dom` if needed. All run in the stock image.
- **Skip browser e2e (Playwright) for the agents**; at most a small human-run smoke test on the host.
  Only add it to the sandbox if the exercise specifically needs it, by extending the Dockerfile with
  Playwright deps and browsers, and keeping it out of `npm run test` to keep the loop fast.
- The template only picks up issues labelled `Sandcastle`, while this repo's triage vocabulary uses
  `ready-for-agent`. Either add the `Sandcastle` label to agent-ready issues, or change `--label Sandcastle`
  to `--label ready-for-agent` in `.sandcastle/plan-prompt.md` after init. Keep issues small and
  file-disjoint so the planner can parallelise them.
