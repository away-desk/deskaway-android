# AGENT.md — deskaway-android

The phone client. It pairs with a desktop, starts and watches a run, and is
where a human approves the steps that need approving. It is the control
surface, not the executor.

Its hard problem is connection state, not UI. A phone loses the network,
sleeps, and gets killed by the OS mid-run, and none of that may lose or
duplicate an approval.

## Folder structure

```
.github/workflows/
  ci.yml  release.yml
gradle/
  libs.versions.toml      # version catalog, single source of dep versions
config/
  detekt.yml              # static analysis rules
  proguard-rules.pro      # release shrinking
app/                      # thin shell: assembly, navigation, DI wiring
core/
  model/                  # domain types, no framework imports
  common/                 # shared utilities
  design/                 # theme and reusable composables
  testing/                # shared test fixtures and fakes
data/
  network/
    rest/                 # relay HTTP calls
    socket/               # relay WebSocket link
    reconnect/            # backoff and resume after a drop
  auth/                   # tokens, sign-in state
  local/                  # on-device persistence
  push/                   # notifications that wake the app for approvals
  repository/             # the single API the features consume
feature/
  pairing/                # discover and pair a desktop
  session/                # claim and hold an active session
  task/                   # write a task, watch its checklist
  approval/               # approve or reject a gated step
  history/                # past runs
  settings/
protocol/                 # generated or vendored deskaway-protocol types
fastlane/                 # release automation
docs/
```

Every directory above except `gradle/` and `config/` holds only a `.gitkeep`
today — they are agreed module boundaries with no Gradle module yet.

## Conventions

- Dependency direction is one way: `app` -> `feature` -> `data` -> `core`.
  A `core` module never imports a `feature`, and features never import each
  other.
- `core/model` stays free of Android and networking imports so it can be
  unit tested on the JVM.
- Features reach `data` only through `data/repository`. No feature talks to
  `network/` or `local/` directly.
- Every dependency version goes in `gradle/libs.versions.toml`. No inline
  versions in a build script.
- `protocol/` is derived from `deskaway-protocol` — never hand-edit a type in
  there, regenerate it.
- Approval actions must be idempotent and survive process death. Assume the
  app is killed between tapping approve and the relay acknowledging it.
- Never commit a keystore or signing properties; `.gitignore` blocks them.

## Rule: keep README.md current

The README is the one file a newcomer is guaranteed to read. Revisit it
whenever this repo's answer to any of the four questions below changes — not
on a schedule.

Every DeskAway README answers four things, in this order:

1. **What this one repo is**, in two lines, and where it sits in the whole
   system.
2. **Its current status**, stated honestly. Right now that is *early
   development, nothing works yet.*
3. **How to run it locally**, aiming for under ten minutes.
4. **A link back** to the org or to `deskaway-docs`, so someone landing here
   can find the rest.

How to apply it:

- Keep those four as the first four sections, in that order. Anything else
  goes after them.
- Status rots fastest. The moment the first thing in this repo actually
  runs, that line changes in the same PR. "Nothing works yet" is honest
  only until it isn't.
- If a setup step breaks, or creeps past ten minutes, fix the README in the
  PR that caused it. A stale run section is worse than no run section.
- Never write intent as if it were fact. Anything not yet true is either
  labelled as planned or left out entirely.
- Two lines means two lines. If section 1 needs a third paragraph, that
  content belongs in `deskaway-docs`.
