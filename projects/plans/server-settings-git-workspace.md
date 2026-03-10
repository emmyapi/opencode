# Server Settings, Workspaces & Git Integration

Move browser settings to server, rework workspace creation as git worktrees, and add VSCode-like git/GitHub integration to OpenCode Web.

---

## Goal

1. Move browser/local settings persistence to server-side persistence
2. Change workspace creation so new workspaces are git worktrees created one level above project root with a naming prompt (e.g. `/srv/appdata/dev/humlebi__workspacename`)
3. Add VSCode-like git/GitHub integration in OpenCode Web (history, commit, push/sync, merge/rebase, PR creation)

---

## Key decisions to lock first

- **Workspace naming format** — use `<repoFolder>__<promptSlug>` exactly, with collision suffixes (`-2`, `-3`, etc.)
- **Scope of "server-side settings"** — all current local persisted state including transient UI state
- **GitHub auth model** — reuse local GitHub CLI (`gh`) with server passthrough, or add OAuth/token storage in OpenCode server?

---

## Discoveries

### Monorepo layout

| Area                     | Package             |
| ------------------------ | ------------------- |
| Backend / API            | `packages/opencode` |
| Web app UI               | `packages/app`      |
| Docs site                | `packages/web`      |
| SDK / OpenAPI generation | `packages/sdk/js`   |
| UI theme system          | `packages/ui`       |

### OpenAPI generation chain

- Routes defined in `packages/opencode/src/server/server.ts` (`Server.openapi()`)
- Spec output by `packages/opencode/src/cli/cmd/generate.ts`
- SDK build pulls OpenAPI via `bun dev generate` in `packages/sdk/js/script/build.ts`

### Existing server config endpoints

- Global config: `/global/config` in `packages/opencode/src/server/routes/global.ts`
- Per-instance config: `/config` in `packages/opencode/src/server/routes/config.ts`

### Workspace / worktree state

- Control-plane workspace API: `packages/opencode/src/server/routes/workspace.ts`
- Experimental worktree API: `packages/opencode/src/server/routes/experimental.ts` (`/experimental/worktree*`)
- Core worktree logic: `packages/opencode/src/worktree/index.ts`
- Current worktree location is under app data, **not** near project root — `Worktree.makeWorktreeInfo()` uses `path.join(Global.Path.data, "worktree", Instance.project.id)`
- Names are auto-generated (`adjective-noun`), optional name slugged, branch `opencode/<name>`

### App persistence (localStorage)

- Central utility: `packages/app/src/utils/persist.ts`
- Settings: `packages/app/src/context/settings.tsx` (`persisted("settings.v3", ...)`)
- Server list, projects, layout, language, keybinds all use `Persist.global/workspace/session`
- Additional direct localStorage usage in `packages/app/src/entry.tsx`, `packages/ui/src/theme/context.tsx`, and `packages/app/public/oc-theme-preload.js`
- Desktop app already has async storage abstraction (Tauri store) in `packages/desktop/src/index.tsx`

### Git API coverage (current)

- Available now: `project.initGit`, `vcs.get` (branch), `file.status` (changed files)
- Missing: log/history, add/reset/stage, commit, push/pull/sync, merge/rebase, PR CRUD
- No dedicated git panel in the app UI yet — only scattered git-aware behavior

---

## Deployment

The official setup for OpenCode Web is through a Docker Compose file. The current approach requires a custom multi-stage build that layers OpenCode onto a base image (e.g. Playwright) and manually copies musl libs. This works but is fragile and limits composability.

### Current pain points

- Users must build a custom image to get tools like Playwright, Vitest, or other test runners
- Alpine/musl binaries need manual lib copying (`ld-musl`, `libstdc++`, `libgcc_s`) into Debian-based images
- Git safe directory config, user setup, and `doas` shim are boilerplate repeated per deployment
- No first-party image that bundles common dev tooling out of the box

### Reference docker-compose.yml

```yaml
services:
  opencode-web:
    image: opencode-web:1.2.19-local
    build:
      context: .
      dockerfile_inline: |
        FROM ghcr.io/anomalyco/opencode:1.2.19 AS opencode
        FROM mcr.microsoft.com/playwright:v1.58.2-jammy

        COPY --from=opencode /usr/local/bin/opencode /usr/local/bin/opencode
        COPY --from=opencode /lib/ld-musl-x86_64.so.1 /lib/ld-musl-x86_64.so.1
        COPY --from=opencode /usr/lib/libstdc++.so.6 /usr/lib/libstdc++.so.6
        COPY --from=opencode /usr/lib/libgcc_s.so.1 /usr/lib/libgcc_s.so.1

        ENV PLAYWRIGHT_BROWSERS_PATH=/ms-playwright

        RUN apt-get update \
            && apt-get install -y --no-install-recommends \
              bash git sudo openssh-client curl jq ripgrep fd-find \
              python3 python3-pip build-essential coreutils findutils \
              grep sed gawk util-linux procps ca-certificates \
            && npm install -g pnpm \
            && ln -sf /usr/bin/fdfind /usr/local/bin/fd \
            && rm -rf /var/lib/apt/lists/*
        RUN git config --system --add safe.directory '*'
        RUN if ! getent group opencode >/dev/null; then groupadd opencode; fi \
            && if ! id -u opencode >/dev/null 2>&1; then \
              useradd -m -d /home/opencode -s /bin/bash -g opencode opencode; fi

        RUN cat >/usr/local/bin/doas <<'SCRIPT'
        #!/bin/sh
        exec "$$@"
        SCRIPT
        RUN chmod +x /usr/local/bin/doas

        ENTRYPOINT ["/usr/local/bin/opencode"]
    init: true
    container_name: opencode-web
    command: ["web", "--hostname", "0.0.0.0", "--port", "4096"]
    restart: unless-stopped
    user: "0:0"
    working_dir: /appforge
    ports:
      - "4096:4096"
    environment:
      HOME: /home/opencode
      XDG_DATA_HOME: /home/opencode/.local/share
      XDG_CONFIG_HOME: /home/opencode/.config
      XDG_STATE_HOME: /home/opencode/.local/state
      XDG_CACHE_HOME: /home/opencode/.cache
      GIT_SSH_COMMAND: >-
        ssh -i /home/opencode/.ssh/id_ed25519
        -o IdentitiesOnly=yes
        -o UserKnownHostsFile=/home/opencode/.ssh/known_hosts
        -o StrictHostKeyChecking=yes
      PLAYWRIGHT_BROWSERS_PATH: /ms-playwright
      OPENCODE_CONFIG_CONTENT: '{"share":"disabled","permission":"allow"}'
    volumes:
      - /:/appforge
      - ./data/.config:/home/opencode/.config
      - ./data/.local:/home/opencode/.local
      - ./data/.cache:/home/opencode/.cache
      - ./data/.pnpm-store:/home/opencode/.pnpm-store
      - /home/kenneth/.ssh:/home/opencode/.ssh:ro
    shm_size: 1gb
    pids_limit: 512
    mem_limit: 8g
```

### Improvements to pursue

- **Publish a Debian-based "full" image** alongside the current Alpine image so musl lib copying is unnecessary. This image should include git, ripgrep, fd, curl, jq, and common build tools pre-installed.
- **Add optional tool layers** (Playwright, Vitest/Node test runners) as official companion images or documented build stages, eliminating the need for users to hand-roll multi-stage builds.
- **Ship a reference `docker-compose.yml`** in the repo under `docker/` with sensible defaults for volume mounts, XDG env vars, git safe directory config, user/group setup, and the `doas` shim.
- **Support `OPENCODE_CONFIG_CONTENT` env var** as a first-class config injection path so headless/server deployments don't need a mounted config file.
- **Document container resource limits** (shm_size, pids_limit, mem_limit) with recommended values for different workload sizes.

---

## Implementation plan

### Phase 1 — Server-backed settings

- Add a canonical server settings contract (global + per-project/workspace namespaces)
- Expose OpenAPI endpoints for read/write/patch (versioned payload, validation, migration hooks)
- Regenerate SDK and switch app settings to SDK calls
- Replace `persisted(... localStorage ...)` in `packages/app` with server sync store
- One-time migration: read localStorage, upload to server, stop writing locally
- Preserve theme preload UX via server-bootstrapped theme data (or minimal local cache for first paint)

### Phase 2 — Workspace creation as git worktrees

- Update worktree creation so target base dir is parent of project root (not `Global.Path.data/worktree/...`)
- Add prompt step in app for worktree name — slugify, validate, show final path preview
- Create worktree at `<projectParent>/<repoName>__<name>`
- Keep branch strategy (`opencode/<name>`) unless changed
- Handle conflicts: existing dir, existing branch, detached/dirty repo, friendly remediation
- Update workspace APIs and experimental routes to return new absolute path + metadata
- Add/adjust tests in `packages/opencode` and app e2e

### Phase 3 — Git core APIs

Introduce backend VCS routes (OpenAPI first) for:

- Status (staged / unstaged / untracked / conflicted)
- Stage / unstage / discard (file-level first, hunk-level later)
- Commit (message + optional amend)
- Fetch / pull / push / sync
- Branch create / switch / delete
- History / log + commit details + file diff

Keep implementation in `project/vcs.ts` + `util/git.ts` with a thin route layer. Normalize error model (auth, merge conflict, non-fast-forward, no upstream) for predictable UI handling. Regenerate SDK and wire app data layer.

### Phase 4 — GitHub PR integration

- Add server endpoints wrapping `gh` for auth status, repo remotes/default base, PR create/list/open, optional checks/status
- Add "Publish branch / Create PR" flow in app, integrated with branch/upstream state
- Show actionable auth/setup hints when `gh` is missing or not logged in

### Phase 5 — Source control UI

Build a dedicated SCM panel in the app:

- Grouped file changes (staged / unstaged / untracked)
- Quick actions (stage, unstage, discard)
- Commit box + commit action
- Sync / push / pull with status badges
- Branch indicator + switch / create

Add a history view (commit list, details, changed files, per-file diff). Integrate merge/rebase UX as guided commands with clear conflict states. Add PR prompt after push when branch is ahead and no PR exists.

### Phase 6 — Docs, migration & hardening

- Update docs that describe local browser persistence
- Add migration notes: what moved server-side, fallback behavior, reset/recovery commands
- Add logging for settings sync errors and git command failures
- Verify: web + desktop parity, multi-project isolation, offline handling, OpenAPI/SDK compat

---

## Relevant files

<details>
<summary>Server / API / OpenAPI</summary>

- `packages/opencode/src/server/server.ts`
- `packages/opencode/src/server/routes/global.ts`
- `packages/opencode/src/server/routes/config.ts`
- `packages/opencode/src/server/routes/experimental.ts`
- `packages/opencode/src/server/routes/workspace.ts`
- `packages/opencode/src/server/routes/project.ts`
- `packages/opencode/src/server/routes/file.ts`
- `packages/opencode/src/server/routes/tui.ts`
- `packages/opencode/src/cli/cmd/generate.ts`
- `packages/opencode/src/config/config.ts`
</details>

<details>
<summary>Workspaces / worktrees</summary>

- `packages/opencode/src/worktree/index.ts`
- `packages/opencode/src/control-plane/workspace.ts`
- `packages/opencode/src/control-plane/workspace.sql.ts`
- `packages/opencode/src/control-plane/types.ts`
- `packages/opencode/src/control-plane/adaptors/index.ts`
- `packages/opencode/src/control-plane/adaptors/worktree.ts`
- `packages/opencode/src/project/project.ts`
- `packages/opencode/src/project/project.sql.ts`
- `packages/opencode/src/project/vcs.ts`
- `packages/opencode/src/util/git.ts`
</details>

<details>
<summary>App persistence + settings</summary>

- `packages/app/src/utils/persist.ts`
- `packages/app/src/context/settings.tsx`
- `packages/app/src/context/platform.tsx`
- `packages/app/src/entry.tsx`
- `packages/ui/src/theme/context.tsx`
- `packages/app/public/oc-theme-preload.js`
- `packages/app/src/context/server.tsx`
- `packages/app/src/context/global-sync.tsx`
- `packages/app/src/context/global-sync/child-store.ts`
- `packages/app/src/context/global-sync/bootstrap.ts`
- `packages/app/src/context/global-sdk.tsx`
- `packages/app/src/context/sdk.tsx`
- `packages/app/src/utils/server.ts`
- `packages/app/src/context/layout.tsx`
- `packages/app/src/context/language.tsx`
- `packages/app/src/context/command.tsx`
</details>

<details>
<summary>App workspace / session UX</summary>

- `packages/app/src/pages/layout.tsx`
- `packages/app/src/pages/layout/helpers.ts`
- `packages/app/src/components/session/session-new-view.tsx`
- `packages/app/src/components/prompt-input/submit.ts`
- `packages/app/src/pages/session.tsx`
- `packages/app/src/pages/session/session-side-panel.tsx`
- `packages/app/src/components/session/session-header.tsx`
- `packages/app/src/components/status-popover.tsx`
- `packages/app/src/components/dialog-settings.tsx`
- `packages/app/src/components/dialog-fork.tsx`
</details>

<details>
<summary>SDK / OpenAPI artifacts</summary>

- `packages/sdk/js/script/build.ts`
- `packages/sdk/js/src/v2/client.ts`
- `packages/sdk/js/src/client.ts`
- `packages/sdk/js/src/v2/gen/sdk.gen.ts`
- `packages/sdk/js/src/v2/gen/types.gen.ts`
- `packages/sdk/openapi.json`
- `packages/console/app/src/routes/openapi.json.ts`
</details>

<details>
<summary>Docker / deployment</summary>

- `Dockerfile` (current Alpine-based image)
- `docker/docker-compose.yml` (to be created — reference compose file)
- `packages/opencode/src/config/config.ts` (OPENCODE_CONFIG_CONTENT handling)
</details>

<details>
<summary>Tests / docs</summary>

- `packages/app/e2e/settings/settings.spec.ts`
- `packages/app/e2e/fixtures.ts`
- `packages/app/e2e/actions.ts`
- `packages/app/e2e/projects/workspace-new-session.spec.ts`
- `packages/opencode/test/project/worktree-remove.test.ts`
- `packages/web/src/content/docs/web.mdx`
- `packages/web/src/content/docs/github.mdx`
- `packages/web/src/content/docs/troubleshooting.mdx` (and locale variants)
</details>

---

## Status

- Completed architecture and code-path reconnaissance for all feature areas
- No code modified yet
- No migrations/builds/tests run yet
- **Next step**: begin Phase 1 (server-backed settings)
