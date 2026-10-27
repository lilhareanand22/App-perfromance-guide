# 📊 Android App Performance Score — Quick Study Notes

> **Source:** [developer.android.com/topic/performance/app-score](https://developer.android.com/topic/performance/app-score)

---

## 🎯 What Is the App Performance Score?

The **App Performance Score** is a standardized framework by Google that generates a score between **0 and 100** to evaluate how well your Android app performs. A **lower score** means more room for improvement. It helps engineering and product teams assess, prioritize, and track performance improvements **without deep profiling expertise**.

---

## 🏗️ Two Pillars of Assessment

### 1. Dynamic Score (Runtime Behavior)

Measures how the app **actually performs at runtime** on a real device. Focuses on:

| Metric | What It Measures |
|--------|-----------------|
| **TTID** (Time to Initial Display) | Time from launch → first frame visible on screen |
| **TTFD** (Time to Full Display) | Time from launch → app is fully interactive (async content loaded) |
| **Slow Frames** | Frames taking > 16.67 ms (on 60 Hz) but < 700 ms — causes visible jank |
| **Frozen Frames** | Frames taking > 700 ms — causes UI to completely freeze |

#### Key Thresholds

| Metric | ✅ Healthy | ❌ Bad |
|--------|-----------|-------|
| Cold Start (TTID) | < 2 seconds | > 5 seconds |
| Slow Frames | ≤ 25% of session frames | > 50% of frames |
| Frozen Frames | ≤ 0.1% of session frames | > 0.5% |
| ANRs | ~0% | > 0.47% |

### 2. Static Score (Best Practices & Tooling)

Evaluates your project setup and build configuration **without running the app**. Checks adoption of high-impact tools and best practices.

---

## 📋 Static Score Breakdown (Scoring Criteria & Recommendations)

### A. Android Gradle Plugin (AGP) Version — Max 20 pts

| AGP Version | Points |
|-------------|--------|
| ≤ 7.2 | 0 |
| 7.3 | 1 |
| 7.4 – 8.1 | 5 |
| 8.2 – 8.5 | 15 |
| **8.6+** | **20** |

> **📌 Recommendation:** Always keep AGP updated to the **latest stable version**. Newer AGP versions include critical compilation pipeline improvements, better R8 optimizations, enhanced resource shrinking, and improved support for Baseline & Startup Profiles. This is the **easiest high-impact change** — just upgrading can gain you 20 points.

---

### B. R8 Code Shrinking & Optimization — Max 20 pts

| Configuration | Points |
|---------------|--------|
| Minification OFF | 0 |
| Compatibility mode / major exceptions (`-dontoptimize`, `-dontshrink`) | 5 |
| **R8 Full Mode + `minifyEnabled true` + `shrinkResources true`** | **20** |

```groovy
// build.gradle (release)
android {
    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}
```

```properties
# gradle.properties
android.enableR8.fullMode=true
```

> **📌 Recommendation:** Enable **R8 Full Mode** for aggressive optimizations (stronger class merging, inlining). Ensure `minifyEnabled` and `shrinkResources` are both `true` for release builds. Audit and tighten ProGuard/R8 keep rules — avoid overly broad rules like `-dontoptimize`, `-dontshrink`, or package-wide `-keep` that prevent R8 from doing its job. R8 also **rewrites Baseline Profile rules** to match obfuscated class names, so proper R8 setup is essential for profiles to work correctly in production.

---

### C. Baseline Profile for App Startup

| Status | Points |
|--------|--------|
| Not used | 0 |
| Manually/infrequently updated | 2 |
| Fully integrated & automated in CI/CD | Max |

> **📌 Recommendation:** Create Baseline Profiles using **Jetpack Macrobenchmark's `BaselineProfileRule`** to pre-compile critical startup code paths via AOT (Ahead-of-Time) compilation. This lets ART skip interpretation and JIT overhead on first launch, dramatically reducing cold start times. **Automate profile generation in your CI/CD pipeline** so profiles stay up to date with every release — stale profiles lose effectiveness as code changes.

---

### D. Baseline Profiles for User Journeys

Extends baseline profiles beyond startup to cover scrolling, navigation, animations, etc.

| Status | Points |
|--------|--------|
| None / startup-only | 0 |
| Covers single journey | Partial |
| Covers multiple major flows, continuously maintained | Max |

> **📌 Recommendation:** Don't stop at startup — extend Baseline Profiles to cover **critical user journeys (CUJs)** like scrolling through `RecyclerView`/`LazyColumn`, navigating between screens, and complex animations. Use Macrobenchmark to record user actions (`uiDevice.findObject(...).click()`, list flings) to capture rich journey traces. This reduces jank and slow frames across the entire user experience, not just the first screen.

---

### E. Startup Profiles (DEX Layout Optimization)

Places critical startup classes together in `classes.dex` for faster class loading.

| Status | Points |
|--------|--------|
| Not implemented | 0 |
| Implemented with DEX optimization | Full |

> **📌 Recommendation:** Implement **Startup Profiles** (`startup-prof.txt`) alongside Baseline Profiles. While Baseline Profiles tell ART *what to pre-compile*, Startup Profiles instruct **D8/R8 compilers** to physically **co-locate startup-critical classes** together inside the primary `classes.dex` file. This reduces page faults and disk I/O overhead during app launch. Generate them using Macrobenchmark and ensure primary DEX overflow is managed properly.

---

### F. `reportFullyDrawn()` / `FullyDrawnReporter` — 5 pts

| Status | Points |
|--------|--------|
| Not using | 0 |
| **Using `reportFullyDrawn()` or `FullyDrawnReporter`** | **5** |

> **📌 Recommendation:** Call `reportFullyDrawn()` (or use Jetpack's `FullyDrawnReporter` / Compose's `ReportDrawn()`) once your app has **finished loading all async content** and is truly fully interactive. This unlocks accurate **TTFD (Time to Full Display)** measurement, enables better Startup Profile creation by the system, and is critical for correct Macrobenchmark results. Without it, the system guesses when your app is ready — often inaccurately.

---

## 📊 Dynamic Score — Recommendations per Metric

### G. App Startup Time (TTID / TTFD)

> **📌 Recommendation:**
> - **Minimize work in `Application.onCreate()`** — avoid initializing heavy SDKs synchronously on the main thread. Use `androidx.startup` for lazy, on-demand initialization.
> - **Defer "below-the-fold" content** — lazily load UI elements not immediately visible on screen.
> - **Offload blocking operations** — move disk I/O, JSON parsing, database queries, and network calls off the main thread to background dispatchers.
> - **Use Baseline + Startup Profiles** together to ensure critical startup paths are AOT-compiled and DEX-optimized.
> - **Target:** Cold start TTID **< 2 seconds**.

---

### H. UI Rendering / Frame Rate (Slow & Frozen Frames)

> **📌 Recommendation:**
> - **Flatten layout hierarchies** — use `ConstraintLayout` or Jetpack Compose to reduce nested `onMeasure`/`onLayout` passes.
> - **Eliminate overdraw** — use Layout Inspector to find unnecessary background layers being drawn.
> - **Avoid allocations in draw loops** — custom canvas drawing or complex animations should not allocate objects inside `onDraw()`, which triggers GC pauses.
> - **Reduce recomposition (Compose)** — use `derivedStateOf`, stable types, and `key()` to isolate UI invalidations. Inspect with Compose Layout Inspector.
> - **Enable StrictMode in debug** — catch accidental disk/network operations on the UI thread early.
> - **Target:** Slow frames **≤ 25%**, Frozen frames **≤ 0.1%** of session frames.

---

## 🔧 How to Use the Framework

```
Step 1 → Run the assessment → Get your 0-100 baseline score
Step 2 → Review recommendations → Focus on low-scoring categories
Step 3 → Implement fixes → (Baseline Profiles, R8 full mode, etc.)
Step 4 → Re-run assessment → Track improvement over time
```

---

## 📈 Performance Levels (Where This Score Fits)

Google categorizes performance maturity into **5 levels**:

| Level | Focus | Tools |
|-------|-------|-------|
| **L1** — Field Monitoring | Zero-effort monitoring of real users | Android Vitals (Play Console) |
| **L2** — Score & Action Items | **← App Performance Score lives here** | Play Console + structured recommendations |
| **L3** — Local Testing | Automated regression prevention | Jetpack Macrobenchmark / Microbenchmark |
| **L4** — Advanced Tracing | Deep-dive root-cause profiling | Android Studio Profiler, Perfetto |
| **L5** — Custom Enterprise | Custom telemetry pipelines & dashboards | Bespoke monitoring infrastructure |

---

## ✅ Checklist for a High Score

- [ ] **AGP 8.6+** — Use the latest stable Android Gradle Plugin
- [ ] **R8 Full Mode** — `android.enableR8.fullMode=true` with minify + shrink enabled
- [ ] **Baseline Profiles for Startup** — Generated via Macrobenchmark, bundled in release
- [ ] **Baseline Profiles for User Journeys** — Cover scrolling, navigation, animations
- [ ] **Startup Profiles** — Optimize DEX layout for startup-critical classes
- [ ] **`reportFullyDrawn()`** — Signal when app is truly fully interactive
- [ ] **Cold Start < 2s** — Optimize TTID and TTFD
- [ ] **Slow Frames ≤ 25%** — Keep main thread lightweight
- [ ] **Frozen Frames ≤ 0.1%** — Eliminate > 700 ms frame hangs
- [ ] **ANR Rate ≈ 0%** — No main-thread blocking operations

---

## 🔑 Key Terminology Cheat Sheet

| Term | Definition |
|------|-----------|
| **TTID** | Time to Initial Display — first frame on screen |
| **TTFD** | Time to Full Display — app fully loaded & interactive |
| **Baseline Profile** | Pre-compiled code paths so ART skips JIT during startup |
| **Startup Profile** | Optimizes DEX layout to co-locate startup classes |
| **R8** | Google's code shrinker, optimizer, and obfuscator |
| **AGP** | Android Gradle Plugin — build toolchain |
| **Macrobenchmark** | Jetpack library for measuring real user interactions |
| **Android Vitals** | Google Play Console dashboard for field performance data |
| **Slow Frame** | Frame render time > 16.67 ms (60 Hz) but < 700 ms |
| **Frozen Frame** | Frame render time > 700 ms |
| **ANR** | Application Not Responding — main thread blocked > 5 seconds |
| **`reportFullyDrawn()`** | API to tell the system when the app is fully interactive |
| **Profile-Guided Optimization (PGO)** | Using runtime profiles to drive AOT compilation decisions |

---

## 🏆 Top 10 Interview Questions & Answers

### Q1: What is the Android App Performance Score and what range does it use?

**Answer:** The App Performance Score is a standardized framework by Google that evaluates an Android app's technical performance on a scale of **0 to 100**. A lower score indicates more room for improvement. It combines both **Dynamic** (runtime behavior) and **Static** (build setup & best practices) assessments to provide actionable recommendations without requiring deep profiling expertise.

---

### Q2: What is the difference between the Dynamic Score and the Static Score?

**Answer:**
- **Dynamic Score** evaluates the app's **actual runtime behavior** on a real device — startup time (TTID/TTFD), frame rendering performance (slow/frozen frames), and UI smoothness during user journeys.
- **Static Score** evaluates **build configuration and tooling adoption** without running the app — checking for R8 full mode, Baseline Profiles, Startup Profiles, AGP version, and proper use of `reportFullyDrawn()`.

---

### Q3: What are TTID and TTFD? Why do they matter?

**Answer:**
- **TTID (Time to Initial Display):** The time from when the user taps the app icon until the **first frame** appears on screen. It signals that the app is responding.
- **TTFD (Time to Full Display):** The time until the app is **fully interactive** with all async content loaded. Measured via `reportFullyDrawn()` or `FullyDrawnReporter`.

They matter because they directly affect **user perception of speed**. A cold start TTID should be **under 2 seconds** to avoid user drop-off.

---

### Q4: What are Slow Frames and Frozen Frames? What are their thresholds?

**Answer:**
- **Slow Frames:** Frames taking longer than **16.67 ms** (on a 60 Hz display) but less than 700 ms. They cause visible jank/stutter. Target: ≤ 25% of session frames.
- **Frozen Frames:** Frames taking longer than **700 ms**. The UI completely freezes, severely damaging user trust. Target: ≤ 0.1% of session frames.

On higher refresh rate displays (90 Hz / 120 Hz), the frame budget tightens to ~11.1 ms or ~8.3 ms respectively.

---

### Q5: How does the AGP version affect the App Performance Score?

**Answer:** Newer AGP versions include critical compilation and optimization improvements. The scoring is:
- AGP ≤ 7.2 → **0 points**
- AGP 7.3 → **1 point**
- AGP 7.4 – 8.1 → **5 points**
- AGP 8.2 – 8.5 → **15 points**
- AGP 8.6+ → **20 points** (maximum)

Upgrading AGP is often the **easiest high-impact change** to improve the static score.

---

### Q6: What is R8 full mode and how should it be configured for the best score?

**Answer:** R8 is Google's code shrinker, optimizer, and obfuscator. **Full mode** enables aggressive optimizations like stronger class merging and inlining. For maximum points (20):

1. Set `android.enableR8.fullMode=true` in `gradle.properties`
2. Enable `minifyEnabled true` and `shrinkResources true` in the release build type
3. Avoid overly broad keep rules like `-dontoptimize` or `-dontshrink`
4. Use `proguard-android-optimize.txt` as the default ProGuard file

---

### Q7: What are Baseline Profiles and why are they important for performance?

**Answer:** Baseline Profiles are pre-compiled code paths bundled into the APK/AAB that tell ART (Android Runtime) which methods to **AOT-compile** ahead of time. This bypasses JIT (Just-In-Time) compilation overhead during app startup and critical user journeys, resulting in:
- **Faster cold starts** (reduced interpretation time)
- **Smoother scrolling and animations** (fewer slow frames)
- **Consistent performance** from the very first app launch

They are generated using **Jetpack Macrobenchmark's `BaselineProfileRule`** and should cover both startup paths and key user journeys.

---

### Q8: What is the purpose of `reportFullyDrawn()` and `FullyDrawnReporter`?

**Answer:** `reportFullyDrawn()` is an API call that signals to the Android system when your app has **finished loading all async content** and is truly fully interactive. It enables:
- Accurate **TTFD measurement** (Time to Full Display)
- Better **Startup Profile generation** by the system
- Proper performance scoring (contributes **5 points** to the static score)

`FullyDrawnReporter` is the Jetpack wrapper that supports multiple async components reporting readiness before the final signal is sent.

---

### Q9: What are the 5 Android Performance Levels, and where does the App Performance Score fit?

**Answer:**
| Level | Description |
|-------|-------------|
| **L1** | Field Monitoring — use Android Vitals in Play Console for basic real-user data |
| **L2** | **App Performance Score** — structured assessment with actionable recommendations |
| **L3** | Local Testing — Macrobenchmark/Microbenchmark for automated regression prevention |
| **L4** | Advanced Tracing — deep-dive with Perfetto, Android Studio Profiler |
| **L5** | Custom Enterprise — bespoke telemetry pipelines and CI performance gates |

The App Performance Score sits at **Level 2**, bridging the gap between passive field monitoring (L1) and active local profiling (L3+).

---

### Q10: What practical steps would you take to improve a low App Performance Score?

**Answer:** I would follow this prioritized approach:

1. **Upgrade AGP** to 8.6+ (easy 20-point gain on static score)
2. **Enable R8 Full Mode** with `minifyEnabled`, `shrinkResources`, and audit keep rules
3. **Generate Baseline Profiles** using Macrobenchmark for startup and critical user journeys
4. **Add Startup Profiles** to optimize DEX layout for startup-critical classes
5. **Implement `reportFullyDrawn()`** to accurately measure and report TTFD
6. **Optimize the main thread** — offload heavy work (disk I/O, JSON parsing, network) to background threads
7. **Flatten view hierarchies** — use ConstraintLayout or Jetpack Compose to reduce layout passes
8. **Monitor with Android Vitals** — track slow frames, frozen frames, and ANR rates in production
9. **Set up Macrobenchmark in CI** — catch performance regressions before they ship
10. **Re-run the assessment** after each batch of fixes to track incremental improvement

---

> 📖 **Reference:** [Android App Performance Score Documentation](https://developer.android.com/topic/performance/app-score)
