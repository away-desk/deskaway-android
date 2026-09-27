# Architecture — deskaway-android

## Responsibility

The control surface. It pairs with a desktop, sends a task, shows the checklist as
it progresses, and is where a human approves the steps that need approving.

It talks only to `deskaway-relay`. Its hard problem is connection state, not
screens: a phone loses the network, sleeps, and is killed by the OS mid-run, and
none of that may lose, duplicate or silently drop an approval.

## What it deliberately does not do

- **It does not execute anything.** No commands, no shell, no automation of the
  phone. It shows what a desktop is doing and collects decisions about it.
- **It never holds a model provider key**, and never calls a model provider. If the
  phone could plan, the approval it is asking for would be its own.
- **It does not connect to a desktop directly**, even on the same Wi-Fi. One
  transport path means one place to authenticate.
- **It does not plan or classify.** Plans arrive via the relay from
  `deskaway-agent`. The phone renders reversibility tiers; it does not assign them.
- **It does not decide what is safe.** Approving a step records a human's decision;
  it does not evaluate it. Enforcement is on the desktop.
- **It does not assume it stays alive.** Nothing may depend on the process
  surviving between a tap and the relay's acknowledgement.
- **It is not the system of record.** `feature/history` reads what the relay
  recorded; the phone stores no authoritative history.

## Internal pieces, and how a message flows

**`app/`** — a thin shell: assembly, navigation, dependency wiring. No feature
logic.

**`core/`** — `model` (domain types, no Android or networking imports), `common`
(utilities), `design` (theme, reusable composables), `testing` (fixtures and fakes).

**`data/`** — `network/rest` for request-response, `network/socket` for the live
link, `network/reconnect` for backoff and resumption; `auth` for tokens; `local`
for on-device persistence; `push` for notifications that wake the app; `repository`
as the single API features consume.

**`feature/`** — one module per screen area: `pairing`, `session`, `task`,
`approval`, `history`, `settings`.

**`protocol/`** — types derived from `deskaway-protocol`, regenerated not
hand-edited.

### Flow of one approval

1. The relay sends an `approval-request`. If the app is foregrounded it arrives on
   the socket in `data/network/socket`; if backgrounded, `data/push` wakes the app
   and the socket reconnects and resumes.
2. The frame is decoded into a `protocol/` type, then mapped to a `core/model`
   type. Wire types stop at the `data` boundary — nothing in `feature/` sees a
   protocol class.
3. `data/repository` persists the pending approval through `data/local` **before**
   showing it. This is what makes the next steps survivable: from here on, the
   request exists even if the process dies.
4. `feature/approval` observes the repository and renders the request with its
   reversibility tier.
5. The human taps approve or reject. `feature/approval` calls
   `data/repository`, which records the decision locally first, then sends it.
6. `data/network/socket` transmits the response. On acknowledgement, the local
   record is marked settled.
7. If the process dies between 5 and 6, the pending decision is still in
   `data/local` and is resent on next start. **Resending must not execute the step
   twice** — that guarantee is the protocol's idempotency key, not a UI concern.

The shape is **socket or push → data → repository → feature**, with every state
change written locally before it is sent.

## Layering rules

**Dependencies point one way: `app` → `feature` → `data` → `core`.** A `core`
module never imports a feature, and **features never import each other.**

The reason for the feature rule: `pairing`, `session`, `task`, `approval`,
`history` and `settings` are the parts most likely to be worked on in parallel and
rewritten. The moment `approval` imports something from `session`, a change to one
screen breaks another, and the modules stop being independently buildable — which
is most of what they were for. Shared UI belongs in `core/design`, shared types in
`core/model`.

**`core/model` contains no Android and no networking imports.** It must be unit
testable on the JVM in seconds. A domain type needing an emulator to test is a
domain type in the wrong module.

**Features reach data only through `data/repository`.** Nothing in `feature/`
imports `data/network` or `data/local` directly, so there is exactly one place that
knows whether an approval came from the socket, from push, or from local storage
after a restart. That single place is where the app's hardest correctness property
lives, and it cannot be enforced if six screens each fetch their own way.

## What it talks to, and in which direction

| Direction | Peer | Over |
| --- | --- | --- |
| **outbound** | `deskaway-relay` | WebSocket + REST, dialled out |
| **inbound** | push provider | notifications that wake the app |
| **local** | device storage | via `data/local` |
| — | `deskaway-protocol` | build-time only, via `protocol/` |

It never connects to `deskaway-desktop`, never to `deskaway-agent`, and never to a
model provider. The relay is its only peer, and the phone always dials out.
