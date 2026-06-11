---
title: "Why an Idle Android Screen Was Burning 300% CPU"
description: "A production-style Android debugging note where an idle screen, Compose animations, and Privy's readiness loop combined into a serious CPU and heat issue."
date: "2026-06-11"
slug: "privy-android-cpu-runaway"
category: "Android Performance"
---

While investigating heat in an Android app, we found a surprising pattern: the main screen looked idle, but the device got hot quickly and `top` showed the app process using roughly 280-300% CPU.

The first suspicion was network polling or repeated API retries. That turned out not to be the main issue. After `logcat -c`, a 10-second observation window showed no repeated logs from the app PID. Protected API requests appeared around tab switches or screen recreation, but they were not continuously repeating.

The root cause had two layers:

1. The main screen card UI was continuously rendering at almost 120fps on an apparently static screen.
2. After stopping that UI rendering loop, Privy's Android SDK readiness waiting path was still burning CPU.

The measurements eventually made the priority clear: the continuous UI rendering was real and worth fixing, but Privy was the dominant heat and CPU source. After the UI frame loop was removed, the app still consumed roughly `200-242%` CPU with hot `DefaultDispatcher` workers. After preventing unnecessary Privy initialization during signed-in cold start, CPU and device temperature dropped much more significantly.

## Measurement Summary

| State | Main screen UI rendering | App CPU | Thermal | Temperature | Interpretation |
| --- | ---: | ---: | --- | --- | --- |
| Baseline build, after ~5 min on the main screen | 5s `618 frames` | `282-325%` | status `1` | AP `48.3C`, PA `49.0C`, skin `39.9C`, BAT `39.7C` | Re-measured baseline still showed both problems: continuous main screen rendering plus hot `DefaultDispatcher` workers from Privy initialization. Thermal status was only `1`, but the device was already warm by AP/PA/skin/BAT readings. |
| Experiment A: shimmer limited to loading only, after ~5 min on the main screen | 5s `618 frames` | `253-293%` | status `3` | AP `49.6C`, PA `50.3C`, skin `42.0C`, BAT `43.3C` | Limiting shimmer alone did not materially reduce the heat source; continuous rendering and Privy `DefaultDispatcher` workers remained. |
| Experiment B: shimmer limited + marquee removed, after ~10 min on the main screen | 5s `0 frames` | `200-242%` | status `2` | AP `44.5C`, PA `45.0C`, skin `39.1C`, BAT `39.5C` | The main screen UI rendering loop stayed gone, but CPU remained high in Privy `DefaultDispatcher` workers. Temperature improved from Experiment A, but the app was still not idle. |
| Experiment C: auth gate only, after ~5 min on the main screen | 5s `610 frames` | `73-92%` | status `0` | AP `38.5C`, PA `39.2C`, skin `35.0C`, BAT `35.3C` | Privy runaway disappeared. Experiment B was not included in this build, so main screen UI rendering still remained. |
| Experiment B + C combined | 5s `0 frames` | `0.0%` in repeated `top` samples | status `0` | AP `33.4C`, PA `33.8C`, skin `32.5C`, BAT `33.2C` | Both loops were gone: no main screen continuous rendering and no Privy `DefaultDispatcher` runaway. |

## First Cause: Always-On Main Screen UI Animation

Card images had an always-applied shimmer modifier. The shimmer implementation used `rememberInfiniteTransition` and `infiniteRepeatable`.

The first experiment limited shimmer to the Coil loading placeholder only. The result barely changed:

- after ~5 minutes on the main screen, the 5s frame count was still `618`
- CPU samples still stayed around `253-293%`
- thermal status reached `3`, with AP `49.6C`, PA `50.3C`, skin `42.0C`, and BAT `43.3C`

At that point, the next likely culprit was `basicMarquee(iterations = Int.MAX_VALUE)` on the card text. The visible card grid had multiple cards, and each card had marquee behavior on grade/title text. Even though the screen looked static, those text animations could keep the Compose animation clock active.

In Experiment B, we removed the card marquee and replaced it with `maxLines = 1` plus `TextOverflow.Ellipsis`.

Result:

- `dumpsys gfxinfo com.example.app reset`
- wait 5 seconds
- `Total frames rendered: 0`

That confirmed the main screen UI continuous rendering loop was gone.

But CPU was still high. Re-measuring the same build after keeping it on the main screen for roughly 10 minutes still showed `200-242%` app CPU, with hot `DefaultDispatcher` workers. The device cooled compared with Experiment A, but it was still not truly idle.

## Second Cause: Privy SDK Readiness Loop

After the UI stopped rendering, the process still showed elevated CPU. Thread-level inspection changed the direction of the investigation.

`top -H` showed that the hot threads were `DefaultDispatcher` workers, not network threads and not `RenderThread`.

We captured an ART method sampling trace:

```bash
adb shell cmd activity profile start \
  --sampling 1000 \
  --clock-type thread-cpu \
  com.example.app \
  /data/local/tmp/privy.trace

sleep 10

adb shell cmd activity profile stop com.example.app
adb pull /data/local/tmp/privy.trace
```

Trace summary:

```text
io.privy.sdk.webview.WebViewState.awaitReady
  ~1.7-1.85s thread-cpu in a ~10s trace
  ~3150+ calls

paired with:
  kotlinx.coroutines.YieldKt.yield
  DefaultIoScheduler.dispatchYield
  LimitedDispatcher.dispatchYield
```

In other words, coroutine workers were repeatedly running a `yield()` loop.

A key detail: this did not require the user to press a sign-in button or call the app's email-code API. The SDK was initialized as a side effect of briefly composing the sign-in route during cold start. Once `Privy.init()` ran, the SDK started its internal readiness work, and that work could keep running even though the user was already authenticated and looking at an unrelated screen.

## Evidence From the Source JAR

Privy's Android SDK does not appear to have a publicly accessible source repository. The Maven POM lists `https://github.com/privy-io/android-sdk` as SCM, but that repository is not publicly accessible at the time of writing.

The Maven Central artifacts do include source JARs, which were enough to inspect the relevant implementation.

App dependency:

```kotlin
implementation("io.privy:privy-core:0.12.0")
```

Maven metadata indicated that `0.12.0` was the latest release.

Relevant source JARs:

```text
~/.gradle/caches/modules-2/files-2.1/io.privy/privy-core/0.12.0/.../privy-core-0.12.0-sources.jar
~/.gradle/caches/modules-2/files-2.1/io.privy/kmp-authentication-impl-android/0.12.0/.../impl-android-sources.jar
```

The key implementation was `WebViewState.awaitReady()`:

```kotlin
while (!isReady) {
  yield()
}
```

`RealWebViewHandler.init()` also starts a coroutine that waits for the WebView ready state.

The important part is the waiting strategy:

- The WebView ready flag is false.
- A coroutine waits until it becomes true.
- The waiting loop only calls `yield()`.
- It does not use `delay`, `StateFlow`, `CompletableDeferred`, a blocking signal, or a timeout-based suspend.
- If the ready flag never flips, `DefaultDispatcher` workers can keep waking up and burning CPU.

That is risky from a CPU perspective. `yield()` means "let other coroutines run"; it does not mean "sleep". If the condition depends on an external event, the coroutine should usually suspend on an event-driven primitive.

## The App-Side Trigger

Even if the SDK behavior is problematic, the app was also waking Privy earlier than intended.

The previous `Root` logic looked like this:

```kotlin
val isAuthenticated by session.isAuthenticated.collectAsState(initial = false)

startDestination =
  if (isAuthenticated) Home else SignIn
```

This is dangerous on signed-in cold start.

Before the local session store emits the stored session, Compose sees `initial = false`. That can briefly compose the SignIn route even though the user is actually signed in.

The SignIn route creates a sign-in view model. That view model injects an auth repository, and the auth repository injects `Privy`.

So the cold-start path becomes:

```text
cold start
-> isAuthenticated initial false
-> SignIn compose
-> sign-in view model
-> auth repository
-> Privy.init
-> Privy SDK readiness work
```

The user might land on the main screen, but the app briefly created SignIn and woke Privy anyway.

In Experiment C, we changed the root navigation gate so the navigation graph does not start until the first auth state emission:

```kotlin
val isAuthenticated: Boolean? =
  session.isAuthenticated.collectAsState(initial = null)

if (isAuthenticated == null) {
  // auth boot/loading
  return
}
```

After that change, a signed-in cold start no longer logged Privy initialization, and `DefaultDispatcher` runaway no longer appeared.

## Conclusion

This was not a single-bug investigation.

The first issue was our UI. Always-on card animations kept producing frames on an idle screen. Limiting shimmer alone was not enough; removing `basicMarquee(Int.MAX_VALUE)` stopped the main screen rendering loop.

The second issue was the Privy SDK path exposed by our auth boot behavior, and it turned out to be the main driver of heat. The app briefly composed SignIn on signed-in cold start, which initialized Privy earlier than necessary. Once initialized, Privy's SDK readiness code could wait using `while (!ready) { yield() }`. If the ready flag did not flip, that could burn CPU across coroutine workers.

The strongest evidence was Experiment B: once the card marquee was removed, `gfxinfo` reported `0` rendered frames over 5 seconds, but app CPU still stayed around `200-242%`. That means the UI was no longer repainting, yet the process was still busy. Experiment C then removed the Privy runaway and brought CPU and temperature down far more sharply, even though the UI rendering loop was still present in that build.

The app-side fix we kept was:

- Do not compose SignIn while auth state is still loading.
- Avoid unnecessary Privy initialization on signed-in cold start.

The UI rendering fix is deferred for now. It clearly removed unnecessary frame production, but the measurements suggest that it was not the source of the critical heat. The device only became truly idle after preventing the Privy initialization path from starting during signed-in cold start.

One meta-lesson from this investigation is that AI agents are useful, but they do not remove the need for engineering judgment.

Codex helped with the mechanical parts: searching the codebase, preparing small experiments, running measurements, reading traces, and turning the findings into a reproducible writeup. But the investigation could still have stopped too early. Once the UI frame loop was found, it would have been easy to declare victory after removing the animation. The device still felt warm, and that human skepticism pushed the investigation deeper into thread-level CPU, ART sampling, and eventually the Privy readiness loop.

In that sense, the agent accelerated the work, but the direction still came from the engineer's judgment: knowing when a result does not explain the whole symptom, when to challenge an attractive explanation, and when to keep measuring. Human-in-the-loop still matters.
