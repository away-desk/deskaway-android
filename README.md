# deskaway-android

The phone client: pair with a desktop, start a task, watch its checklist, and
approve the steps that need a human.

It talks only to `deskaway-relay`, never to a desktop directly. This is the
control surface for a run happening on another machine.

## Status

**Early development, nothing works yet.**

The module boundaries are agreed and the directory tree reflects them, but
there is no Gradle module yet — no `settings.gradle.kts`, no root build
script, no `AndroidManifest.xml`, and `gradle/libs.versions.toml` is empty.
There is no app to install. Only `config/detekt.yml` and
`config/proguard-rules.pro` exist as files, and both are empty too.

## Running locally

Not yet possible: there is no Gradle build to invoke.

You will need Android Studio, or a JDK plus the Android SDK with
`ANDROID_HOME` set. The intended path, once the build files exist:

```sh
git clone https://github.com/away-desk/deskaway-android.git
cd deskaway-android

./gradlew assembleDebug
./gradlew installDebug        # onto a device or running emulator

./gradlew testDebugUnitTest
./gradlew detekt
```

Target is under ten minutes on a cold clone, though the first Gradle sync
and SDK download will dominate that and may well exceed it on a machine
without the Android SDK already present — worth stating plainly here rather
than pretending otherwise.

Pairing needs a reachable `deskaway-relay`; unit tests must not.

## The rest of DeskAway

Cross-repo docs and architecture decisions live in
**[deskaway-docs](https://github.com/away-desk/deskaway-docs)**. All
components are under the **[away-desk](https://github.com/away-desk)** org.

The wire protocol this client speaks is defined in
[deskaway-protocol](https://github.com/away-desk/deskaway-protocol).
Contributor guidance, including the rule for maintaining this README, is in
[AGENT.md](./AGENT.md).
