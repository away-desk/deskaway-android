# Local setup — deskaway-android

**Nothing builds yet.** There is no `settings.gradle.kts`, no root build script
and no module, so there is no Gradle build to invoke. This page describes the
setup as it is intended to work. Correct it in the same pull request that makes
it true.

## What you need

- Android Studio, or a JDK 17 plus the Android SDK with `ANDROID_HOME` set.
- A device or emulator to install onto.

## Steps

```sh
git clone https://github.com/away-desk/deskaway-android.git
cd deskaway-android

./gradlew assembleDebug
./gradlew installDebug        # onto a connected device or running emulator
```

Opening the project in Android Studio and letting it sync does the same thing
with more feedback, and is the easier first run.

**Be honest about the ten-minute target here:** on a machine that already has the
Android SDK and a warm Gradle cache, this is a couple of minutes. On a fresh
machine, the SDK download and first Gradle sync will take considerably longer,
and no amount of documentation changes that. The target applies to the second
clone, not the first.

## Tests and static analysis

```sh
./gradlew testDebugUnitTest    # JVM unit tests, no device needed
./gradlew detekt               # rules in config/detekt.yml
```

`core/model` is deliberately free of Android imports so its tests run on the JVM
in seconds. Keep it that way; a domain type that needs an emulator to test is a
domain type in the wrong module.

A formatter still needs choosing and wiring — ktlint or spotless. Until then,
`detekt` is the only automated style gate, and `CONTRIBUTING.md` should be
updated when that changes.

## Module layout when adding code

Dependency direction is one way: `app` → `feature` → `data` → `core`.

- A new screen area is a new module under `feature/`. Features must not import
  each other; shared UI goes in `core/design` and shared types in `core/model`.
- Features reach data only through `data/repository`. Nothing in `feature/`
  should import `data/network` or `data/local` directly.
- Every dependency version goes in `gradle/libs.versions.toml`. No inline
  versions in a build script.
- Types in `protocol/` are derived from `deskaway-protocol`. Regenerate them;
  do not hand-edit.

## Running against a relay

Pairing needs a reachable `deskaway-relay`. Point a debug build at a local one
rather than the deployed environment.

Unit tests must not need it. If a test in `core` or `data` starts requiring a
live relay, it belongs in an instrumented or integration test instead.

Two things worth exercising deliberately, because they are where this component
is actually difficult:

- **Kill the app mid-run** and reopen it. The checklist should still be right.
- **Turn off the network during an approval** and turn it back on. The approval
  must arrive once — not zero times, and not twice.

## When it will not build

- **Gradle sync fails on a missing SDK component** — let Android Studio install
  it; the command line will not offer.
- **`./gradlew` is missing** — expected today; the Gradle wrapper has not been
  committed yet.
- **Detekt fails on code you did not touch** — `config/detekt.yml` changed. Fix
  the findings rather than adding a baseline file.
