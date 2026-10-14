# Build Cloudroom from source

You need macOS on Apple Silicon, Git, Node.js 22.19+, pnpm 9.15.0, Python 3.11+, and Xcode Command Line Tools. Building needs no Cloudroom account.

Run these from this folder:

```sh
pnpm install --frozen-lockfile
pnpm exec turbo run build typecheck --filter=@cloudroom/app --filter=@cloudroom/server --filter=@cloudroom/desktop --concurrency=2
```

Keep dependency install scripts on. SQLite, file watching, and terminals use native modules. If your pnpm version differs, use `npx --yes pnpm@9.15.0`.

## Run in dev mode

Start the backend and browser UI, then Electron in a second terminal:

```sh
pnpm dev
pnpm exec turbo run dev --filter=@cloudroom/desktop
```

Dev mode uses its own ports and profile. Local agents need their own installed, signed-in CLIs. To use your own core for Cloud threads, see [connect the desktop app](https://github.com/davidondrej/cloudroom-core/blob/main/docs/setup.md#connect-the-desktop-app).

## Checks

```sh
pnpm exec turbo run test --filter=@cloudroom/scripts --filter=@cloudroom/config --concurrency=2
```

Desktop integration tests open real apps, so run them only in a test environment. The sync and preview helpers in `apps/server/src/assets/` are copies from the core, so change them there.

The build makes runnable bundles, not a signed installer.

---

# extending

Build a Windows desktop client for Cloudroom that can run **natively on Windows without WSL** and consume the **same backend capabilities** currently used by `cloudroom-gui` and `cloudroom-core`.

Use the existing `cloudroom-gui` repository as the base. Reuse everything that is already cross-platform and keep the current product behavior wherever it already works. Only replace or isolate the parts that are macOS- or Unix-specific.

## Goal

Deliver a Windows client that:
- runs on Windows 10/11
- connects to the same Cloudroom backend and remote services that the current GUI uses
- preserves Cloudroom’s thread, project, environment, terminal, file, account, and settings workflows
- reuses the existing renderer/UI/API contracts where possible
- does not require WSL, a local Linux agent runtime, or macOS-only helper behavior
- degrades gracefully for features that are inherently unavailable on Windows

## Hard constraints

- Do **not** redesign the app or invent a new product flow.
- Do **not** tell the implementation model to “figure out the spec.”
- Do **not** introduce a separate replacement UI unless absolutely necessary.
- Do **not** require WSL.
- Do **not** depend on Unix-only process, PTY, socket, or filesystem behavior on Windows.
- Do **not** place tokens in URLs or logs.
- Do **not** silently claim support for features that are actually Mac-only or local-agent-only.

## Reuse-first strategy

Reuse the following existing layers wherever possible:
- renderer/UI components
- thread, project, environment, and settings state management
- existing API client patterns
- streaming/reconnect logic
- authentication/session concepts
- server contract types and route shapes
- any platform-neutral business logic

Only replace or isolate:
- macOS-specific startup and helper integration
- Unix socket / PTY / `/proc` / `/dev/*` assumptions
- process ancestry assumptions
- credential storage that relies on macOS or Linux shell semantics
- local command execution paths that cannot exist on Windows

## What the Windows client must do

### Core product behavior
The Windows client must support, at minimum:
- signing in and maintaining session state
- opening the main Cloudroom app shell
- listing and opening threads/sessions
- viewing live conversation output and history
- sending prompts and follow-up messages
- retrying/resuming supported actions
- browsing projects, workspaces, and files
- managing settings
- showing account and provider state
- handling streaming updates and reconnects
- displaying errors clearly and safely
- respecting permissions, auth boundaries, and sandbox boundaries

### Data flow expectations
The Windows client should consume the same backend shape the existing GUI expects:
- HTTP APIs for CRUD/state
- streaming transport for live updates
- authenticated requests for user/account/session state
- separate handling for local-only vs cloud-hosted features
- stable request identifiers for retries and idempotent actions where applicable

## Backend/API surface the Windows client must support

Implement the Windows client so it can consume the backend capabilities below. If a feature is unavailable on Windows, clearly disable it in the UI and document the reason.

### 1) Projects
Must support:
- list projects
- create project
- open project
- update project
- delete project
- reorder projects
- read project execution defaults
- read prompt history
- create/update/delete project sources
- browse project files
- read project file contents
- list project paths
- list project commands
- list project skills
- read/update skill content
- list skill files
- manage project branches and branch options
- upload/copy attachments
- read attachment content

### 2) Files
Must support:
- read files
- write files
- list files
- list paths
- make directories
- move paths
- remove paths
- create file previews

### 3) Threads
Must support:
- list threads
- count threads
- list currently running threads
- search threads
- resolve mentions
- create thread
- fork thread
- get thread details
- update thread metadata
- get/update thread plugin metadata
- delete thread
- get child summary
- send message
- edit message
- retry turn
- list queued messages
- create queued message
- update queued message
- send queued message
- reorder queued message
- prompt history
- delete queued message
- stop thread
- compact thread
- clear context
- cancel plan
- clear goal
- set goal
- open thread
- pane actions
- tabs read/update
- pin/unpin/reorder
- interactions: list/get/resolve/respond/cancel
- archive all
- unarchive
- read/unread
- timeline
- context
- conversation outline
- timeline turn summary details
- output
- events
- wait for next event
- default execution options
- thread storage file list / location / content / paths
- host file content for a thread
- worktree file access
- raw file access

### 4) Terminals
Must support:
- list terminals
- create terminal
- get terminal
- update terminal
- restart terminal
- close terminal
- send input to terminal
- resize terminal
- read terminal output

### 5) Environments
Must support:
- list environments
- get environment
- update environment
- delete environment
- status
- pull request view
- diff view
- diff files
- diff file
- diff branches
- paths
- environment actions
- archive threads

### 6) System / settings
Must support:
- machine environment
- replace machine environment
- system attention
- system config
- general settings
- keyboard settings
- experiments
- appearance
- UI preferences get/update/reset
- themes and theme resolution
- reload config
- CLI skills status/install
- execution options
- restart model discovery
- environment providers
- machine providers
- providers
- installable providers
- provider logos
- provider states
- usage limits
- voice transcription
- version

### 7) Hosts
Must support:
- create host
- create join code
- list hosts
- get host
- enrollment command
- update host
- update permission ceiling
- retry update
- reconcile
- suspend
- resume
- retry cleanup
- delete host
- directory listing
- clone default path
- path existence checks
- pick folder
- provider CLI status
- provider CLI install

### 8) Desktop browser features
Must support:
- list browser instances
- list tabs
- create tab
- acquire control
- open connection
- release control
- close tab
- reveal tab
- capture tab
- list import sources
- import cookies

### 9) Queue
Must support:
- list queued messages across the workspace

### 10) Account / auth / local integrations
Must support:
- account status
- sign-in flow
- logout/cancel
- project account association
- GitHub login/account/disconnect
- Connect relay registration
- bug reports
- copy logins / cloud logins where relevant
- Cloud environment and sandbox configuration
- auto-delete settings
- share links
- MCP sign-in and server propagation
- Claude / Codex / Cursor / Pi / OpenCode / Hermes account integrations where supported by the backend

## Windows-specific feature handling

### Must implement natively
- secure token storage
- file dialogs
- clipboard integration
- notifications
- app updates / installer flow
- window lifecycle
- deep linking / protocol handling if needed
- Windows file-path handling
- Windows-friendly account/session persistence

### Must replace or isolate
- Unix sockets
- PTYs and `/dev/ptmx`
- `/proc` process ancestry
- Unix file permission assumptions
- macOS helper app flows
- Mac-only command forwarding
- Mac-specific credential/keychain behavior
- shell integration that depends on Unix login shell behavior

### If not feasible
If a feature cannot be made native on Windows, then:
- disable it explicitly
- surface a clear explanation in the UI
- keep the rest of the client functional

## Repository reuse guidance

Use the existing repository structure as the starting point:
- preserve the existing app shell, routes, and service layers where possible
- keep current API contract types and generated clients intact
- add platform abstractions rather than duplicating logic
- isolate platform-specific modules behind interfaces
- prefer small adapters over rewrites

## Existing code areas to preserve or adapt

Treat the following as reusable or partially reusable:
- the existing GUI app shell and renderer
- server contract definitions
- cloudroom service routes and clients
- thread/project/environment state machinery
- streaming/event handling
- account and settings screens
- update and telemetry plumbing where cross-platform
- validation schemas and API typing

Treat the following as Windows blockers that need replacement/abstraction:
- Mac access helper logic
- Unix socket access
- PTY/terminal internals
- Unix-specific secrets flow
- shell/login assumptions tied to macOS or Linux
- process ancestry code based on `/proc`
- command forwarding that assumes the paired Mac model

## Feature parity checklist

Document whether each area is:
- fully supported on Windows
- partially supported with degradation
- intentionally disabled

At minimum, produce a parity matrix for:
- auth/sign-in
- threads/messages
- prompts/retries
- streaming/reconnect
- projects/files
- environments/diffs
- terminals
- hosts
- settings
- browser features
- queue
- account integrations
- share links
- bug reports
- MCP
- local command execution
- local helper/control features
- file dialogs
- desktop notifications
- updates

## Deliverables

1. A Windows client implementation in the existing repository.
2. A clearly documented Windows compatibility layer or adapter boundary.
3. A build/install path for Windows.
4. A test plan and automated tests for the Windows-specific flows that can be tested in CI.
5. A feature parity matrix showing supported / partial / disabled features.
6. A list of any features that remain blocked by platform limitations.

## Acceptance criteria

The Windows client is acceptable only if it can:
- start on Windows without WSL
- authenticate successfully
- connect to the same backend model the existing GUI uses
- list and operate on threads
- send messages and observe streamed updates
- browse and manage projects/files/settings
- clearly disable unsupported Mac/local-only behavior
- preserve security boundaries and not leak tokens

## Important implementation note

Focus on **consuming the same backend as `cloudroom-gui`**, not on inventing a new architecture. The Windows client should be the same product, just with Windows-native wrappers and platform-specific replacements where required.
