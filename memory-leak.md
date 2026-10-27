# Android Memory Leaks — Beginner-to-Interview Study Guide

> **Goal:** Understand what an Android memory leak is, why it happens, how to prevent common leaks in Kotlin/Android code, and how to prove that a fix worked using Android Studio Memory Profiler and LeakCanary.
>
> **Learning order:** Fundamentals → object references → common leak patterns → lifecycle-safe fixes → debugging workflow → interview questions.

---

## 1. What is a memory leak?

A **memory leak** happens when an object is no longer useful, but the application still holds a reference to it. Because the object is still reachable, the garbage collector (GC) cannot reclaim it.

Think of a hotel room that a guest has checked out of, but the room is still marked as occupied in the booking system. The room cannot be assigned to another guest even though nobody needs it. A leaked object is similar: memory remains occupied unnecessarily.

### Android example

1. `ProfileActivity` is created and displays a screen.
2. The user leaves the screen and the Activity is destroyed.
3. A long-lived singleton still holds a reference to that Activity.
4. The Activity and possibly its entire View hierarchy remain reachable.
5. GC cannot reclaim those objects.

A single small leak may not be noticeable. Repeating the same navigation many times can retain many Activities, Fragments, Views, Bitmaps, or other objects and gradually increase memory pressure.

### Memory leak vs high memory usage vs memory churn

| Term | Meaning | Example |
|---|---|---|
| Memory leak | Unneeded objects remain reachable for longer than their intended lifetime | A singleton retains a destroyed Activity |
| High memory usage | The app legitimately needs a large amount of memory at that moment | Editing a large photo |
| Memory churn | The app repeatedly creates many short-lived objects | Allocating temporary objects on every animation frame |
| OutOfMemoryError (OOM) | An allocation cannot be satisfied from the available memory for that allocation | Trying to allocate a very large Bitmap when the heap has insufficient room |

**Important:** A high memory graph by itself does not prove a leak. The runtime may retain heap space for future allocations even after objects become collectible. Look for objects that should have been destroyed but remain reachable, and compare repeated user journeys.

---

## 2. Why can't Garbage Collection remove a leaked object?

The ART garbage collector identifies objects that can still be reached from **GC roots**. Objects reachable from a root are considered live. Objects that are no longer reachable are eligible for collection.

Common GC roots include thread stacks/local references, active threads, system classes and references held by native code or the runtime.

### Reachability example

```kotlin
class User(val name: String)

var currentUser: User? = User("Anand")

fun clearUser() {
    currentUser = null
}
```

After `currentUser = null`, the `User` object is eligible for collection only if no other strong reference points to it. Setting one variable to `null` does not force an immediate GC run.

### A leak is a reference path

A leak trace can be read as a chain:

```text
Long-lived GC root
    ↓
Singleton / static field
    ↓
Collection or callback
    ↓
View
    ↓
Destroyed Activity
```

The key debugging question is:

**Which long-lived object still points to the object that should have been released?**

A `GC root` at the top is usually not the bug. Find the reference in the chain whose lifetime is longer than the target object's intended lifetime.

---

## 3. The most common Android memory leaks

### 3.1 Singleton or static object holding an Activity

A singleton normally lives as long as the app process. An Activity normally lives only for a screen instance. Storing an Activity in a singleton can therefore keep the Activity alive after it is destroyed.

**Incorrect:**

```kotlin
object ScreenStore {
    var activity: Activity? = null
}

class ProfileActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        ScreenStore.activity = this // Dangerous long-lived reference
    }
}
```

**Better:** Do not store an Activity in a singleton. Pass short-lived UI dependencies to the code that needs them, and keep app-wide data in repositories or application-scoped objects that do not retain UI objects.

If a long-lived component genuinely needs a `Context` for an app-wide operation, an application context may be appropriate:

```kotlin
class AnalyticsStore(context: Context) {
    private val appContext = context.applicationContext
}
```

`applicationContext` does not replace an Activity context when a themed UI, window, or Activity-specific operation is required. Avoid keeping a UI context beyond that UI's lifetime.

### 3.2 Fragment ViewBinding leak

A Fragment and its View have different lifecycles. A Fragment can remain on the back stack after its View is destroyed. If the Fragment keeps its old binding, it also keeps references to the old View hierarchy.

**Correct ViewBinding pattern:**

```kotlin
class DetailsFragment : Fragment(R.layout.fragment_details) {

    private var _binding: FragmentDetailsBinding? = null
    private val binding get() = requireNotNull(_binding)

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        _binding = FragmentDetailsBinding.bind(view)
        binding.title.text = "Details"
    }

    override fun onDestroyView() {
        _binding = null
        super.onDestroyView()
    }
}
```

**Why clear it in `onDestroyView()`?** The binding owns references to Views. The View lifecycle ends at `onDestroyView()`, even if the Fragment instance itself is retained.

Do not keep View, binding, Activity, or other UI references in a ViewModel. A ViewModel can outlive a particular Activity instance during a configuration change.

### 3.3 Handler and delayed Runnable leak

A `Handler` can post a `Runnable` for later execution. If that Runnable is an inner class or lambda that captures an Activity, the message queue can keep the Activity reachable until the callback runs or is removed.

**Risky pattern:**

```kotlin
class SearchActivity : AppCompatActivity() {
    private val handler = Handler(Looper.getMainLooper())

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        handler.postDelayed({
            // This lambda can capture the Activity.
            updateSearchUi()
        }, 60_000)
    }
}
```

**Fix when the callback is only needed while the Activity is active:** keep a reference to the callback and remove it at the matching lifecycle boundary.

```kotlin
class SearchActivity : AppCompatActivity() {
    private val handler = Handler(Looper.getMainLooper())

    private val updateRunnable = Runnable {
        updateSearchUi()
    }

    override fun onStart() {
        super.onStart()
        handler.postDelayed(updateRunnable, 60_000)
    }

    override fun onStop() {
        handler.removeCallbacks(updateRunnable)
        super.onStop()
    }
}
```

Choose the lifecycle boundary based on the actual requirement. If the callback must continue while the screen is stopped, it should not directly capture or update the Activity UI; move the work to a lifecycle-independent owner and deliver results to the UI safely.

### 3.4 Listener, observer, receiver, or callback not unregistered

A long-lived publisher can retain a listener. If the listener is an Activity or captures one, the Activity may stay in memory.

Examples include:

- Location updates or sensor listeners.
- Event-bus subscriptions.
- Custom listener interfaces.
- Broadcast receivers registered dynamically.
- Observers attached to non-lifecycle-aware sources.
- SDK callbacks that are never unregistered.

**Pattern:** Register and unregister using matching lifecycle methods.

```kotlin
class CompassActivity : AppCompatActivity() {
    private val sensorListener = object : SensorEventListener {
        override fun onSensorChanged(event: SensorEvent?) {
            // Update UI while this screen is active.
        }

        override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) = Unit
    }

    override fun onResume() {
        super.onResume()
        // Register the listener with SensorManager here.
    }

    override fun onPause() {
        // Unregister the same listener here.
        super.onPause()
    }
}
```

The example shows the lifecycle pairing; add the actual `SensorManager` registration for the sensor your app uses.

For lifecycle-aware observable data, prefer APIs such as `LiveData` observers with a `LifecycleOwner`, or collect Flows with `repeatOnLifecycle` / `collectAsStateWithLifecycle` rather than collecting forever from a UI owner.

### 3.5 Coroutine outliving the screen

A coroutine that keeps running after its UI owner is gone can retain captured objects, perform unnecessary work, and attempt to update an obsolete screen.

**Avoid for screen work:**

```kotlin
// Risky for Activity-specific work: no automatic screen lifecycle cancellation.
GlobalScope.launch {
    val result = repository.loadProfile()
    // May capture or try to update a screen that no longer exists.
}
```

**Use an appropriate lifecycle scope:**

```kotlin
class ProfileActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        lifecycleScope.launch {
            val profile = withContext(Dispatchers.IO) {
                repository.loadProfile()
            }
            // Update this Activity's UI on the Main dispatcher.
            render(profile)
        }
    }
}
```

For screen state and work that should survive Activity recreation, use a `ViewModel` and `viewModelScope`. The work is cancelled when the ViewModel is cleared. For UI collection that should only run while visible, use `repeatOnLifecycle`.

**Cancellation is cooperative:** long-running work must suspend or check cancellation. Avoid swallowing `CancellationException`.

### 3.6 ViewModel holding Activity, View, or binding

A ViewModel commonly survives configuration changes. If it stores an Activity, Fragment, View, or ViewBinding, it can keep the old UI instance alive.

**Incorrect:**

```kotlin
class ProfileViewModel(activity: Activity) : ViewModel() {
    private val screen = activity // Do not keep UI objects here.
}
```

**Better:** Keep data and UI state in the ViewModel. Let the Activity or Fragment observe the state and render the UI.

```kotlin
class ProfileViewModel(
    private val repository: ProfileRepository
) : ViewModel() {
    val profile = repository.profileFlow
        .stateIn(
            viewModelScope,
            SharingStarted.WhileSubscribed(5_000),
            null
        )
}
```

An application context is not the same as an Activity context and can be used in specific app-wide cases, but prefer passing the required dependency (such as a repository) instead of passing Context everywhere.

### 3.7 Jetpack Compose effects without cleanup

Compose UI can create subscriptions, listeners, and other side effects. When the composable leaves composition, those resources must be cleaned up.

Use `DisposableEffect` when registering something that has a matching cleanup action:

```kotlin
@Composable
fun ObserveConnection(connectivityManager: ConnectivityManager) {
    DisposableEffect(connectivityManager) {
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) {
                // React to connectivity changes.
            }
        }

        val request = NetworkRequest.Builder().build()
        connectivityManager.registerNetworkCallback(request, callback)

        onDispose {
            connectivityManager.unregisterNetworkCallback(callback)
        }
    }
}
```

Use `LaunchedEffect` for coroutine work tied to a composable's keys and composition lifetime. Use `rememberCoroutineScope()` for event-triggered coroutines tied to that composition. Do not launch unbounded work or retain Activity/View references in long-lived state holders.

For Flow collection in a screen, prefer lifecycle-aware collection such as:

```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

### 3.8 Unbounded caches and collections

A cache is not automatically a leak. It becomes a problem when it grows without a limit or retains objects longer than their intended lifetime.

Examples:

- A `MutableList` that stores every screen visited.
- A singleton cache that stores Views or Activities.
- An image cache with no size policy.
- A map that keeps every API response indefinitely.

**Better practices:**

- Set maximum size or use an LRU policy.
- Cache data, not Activity/View objects.
- Expire entries when appropriate.
- Clear screen-specific entries when their owner is finished.
- Use image-loading libraries with appropriate memory and disk cache policies.

### 3.9 WebView not cleaned up correctly

A WebView can use significant Java and native memory. When a WebView is permanently finished, remove it from its parent and destroy it according to the WebView lifecycle. Do not keep a reference to the WebView or its Activity in a singleton.

A simplified cleanup pattern is:

```kotlin
fun disposeWebView(webView: WebView) {
    (webView.parent as? ViewGroup)?.removeView(webView)
    webView.stopLoading()
    webView.webChromeClient = null
    webView.webViewClient = WebViewClient()
    webView.destroy()
}
```

Call cleanup only when the WebView is truly no longer needed; do not destroy it while the UI still uses it. Review the current WebView guidance for the full set of app-specific cleanup requirements.

### 3.10 Unnecessary Service or background component lifetime

Keeping a Service running when it has no work to do consumes process resources and can make the system keep the process at a higher importance. Stop services when their work is complete and use the correct background-work API for the use case. Do not use a Service simply as a permanent holder for application state.

---

## 4. Lifecycle: where should cleanup happen?

There is no universal cleanup method. Pick the callback that matches the resource's intended lifetime.

| Resource / work | Common cleanup point | Reason |
|---|---|---|
| Fragment ViewBinding | `onDestroyView()` | The View is destroyed before the Fragment may be destroyed |
| UI listener needed only while visible | Matching unregister in `onStop()` or `onPause()` | Stop receiving events when the UI is not active |
| Activity-specific coroutine | `lifecycleScope` cancellation | Coroutine is cancelled when the Activity lifecycle is destroyed |
| ViewModel-owned work | `viewModelScope` cancellation / `onCleared()` for custom cleanup | Work follows the ViewModel lifetime |
| Compose subscription | `DisposableEffect` → `onDispose` | Unregister when the effect leaves composition or its key changes |
| Delayed Handler callback | `removeCallbacks()` at matching lifecycle boundary | Remove queued work that should no longer run |
| Service work | Stop the Service when finished | Avoid keeping an unnecessary process component alive |
| Large UI cache | When UI becomes hidden or the owning screen is disposed | Release resources that can be recreated later |

### `onDestroy()` is not a guaranteed process-shutdown callback

Do not depend on `Activity.onDestroy()` or `Application.onTerminate()` to save all important data or clean up every resource before Android kills a process. Android may terminate a process without calling those methods. Persist important data as the user works, and scope resources so they do not need a last-minute global cleanup callback.

---

## 5. WeakReference: should we use it to fix leaks?

A `WeakReference` does not keep its target alive solely because of that weak reference. It can be useful for specific designs where a non-owning reference is intentional.

However, **do not use WeakReference as a quick fix for every memory leak**.

Why?

- It may hide the real lifecycle or ownership bug.
- The target can disappear at any time after becoming weakly reachable.
- Code must handle a `null` target safely.
- It can make behaviour unpredictable if the object was actually required.

The better fix is usually to remove or shorten the strong reference, unregister a callback, cancel work, or use a lifecycle-aware owner.

---

## 6. How to detect memory leaks

Use more than one signal. A graph showing that memory stays high is not enough to diagnose a leak.

### 6.1 LeakCanary

LeakCanary is a popular Android library for detecting retained objects in debug builds. It watches objects that should become collectible after lifecycle events, waits to see whether they remain retained, captures a heap dump when appropriate, analyses reference paths, and displays a leak trace.

Typical setup in the app module's Gradle dependencies:

```kotlin
dependencies {
    // Replace <latest-version> with the current stable version from LeakCanary docs.
    debugImplementation("com.squareup.leakcanary:leakcanary-android:<latest-version>")
}
```

Use the current official setup instructions because dependency versions and Gradle configuration can change. In a standard setup, LeakCanary is intended for debug/testing builds, not as a feature that users interact with in production.

### How to read a LeakCanary leak trace

Example (simplified):

```text
GC Root: System class
    ↓
Application singleton
    ↓
MutableList
    ↓
TextView
    ↓
Destroyed ProfileActivity
```

Interpretation:

1. The system class is a long-lived root.
2. It references an application-level singleton.
3. The singleton contains a list.
4. The list contains a TextView.
5. The TextView holds its Context, which is the destroyed Activity.

The important suspect is often the application singleton's list entry. The TextView-to-Activity reference is normal; the problem is that the singleton kept the View alive after the Activity should have gone away.

### 6.2 Android Studio Memory Profiler

Use the Memory Profiler to:

- Observe memory usage and allocations over time.
- See when GC occurs.
- Record allocation activity and inspect allocation call stacks.
- Capture a Java heap dump.
- Inspect object counts, fields, references, and retained size.
- Compare snapshots before and after repeating a user journey.

A heap dump is a snapshot of objects in the managed heap. It helps identify objects that remain in memory and the reference paths holding them.

### 6.3 ADB memory information

```bash
adb shell dumpsys meminfo com.example.myapp
```

Replace `com.example.myapp` with your app's package name. This gives a process-level memory breakdown. It is useful for triage, but it does not by itself prove which object is leaking.

### 6.4 Android Vitals and production monitoring

For production apps, monitor low-memory process kills, OOM events, and memory anomalies using Android's available quality and profiling tools. A device-side heap dump from a debug session and aggregate production metrics answer different questions; use both where appropriate.

---

## 7. Practical debugging workflow: prove the leak and prove the fix

Use this repeatable process for an Activity or Fragment leak.

### Step 1 — Create a reproducible journey

Example: open `ProfileActivity`, open a details screen, press Back, and repeat 5–10 times. Also test device rotation, app background/foreground transitions, and navigation away from a Fragment.

### Step 2 — Establish a baseline

Launch the app, let startup settle, and inspect memory. Note the number of instances of the Activity, Fragment, View, and any large object types that matter.

### Step 3 — Repeat the journey

Open and close the same screen several times. Allow transitions and short-lived callbacks to finish.

### Step 4 — Capture a heap dump

Use Android Studio Memory Profiler or LeakCanary. Capture after the repeated journey and, where appropriate, after GC has had an opportunity to run.

### Step 5 — Find retained objects

Look for destroyed Activities, Fragments, Fragment Views, old ViewBindings, duplicate ViewModels, large Bitmaps, or growing collections that should not remain.

### Step 6 — Follow references back to the owner

Inspect the reference path from a GC root. Ask:

- Which object owns this reference?
- Is that owner supposed to outlive the Activity or Fragment?
- Why does it still have this callback, View, Context, or collection entry?
- Which lifecycle event should remove the reference or cancel the work?

### Step 7 — Fix the ownership/lifecycle problem

Examples: clear Fragment binding in `onDestroyView()`, unregister a listener, remove a Handler callback, cancel an obsolete job, bound a cache, or stop storing an Activity in a singleton.

### Step 8 — Repeat the exact test

Run the same navigation and rotation sequence again. Confirm that old instances are no longer retained and object counts return close to the expected baseline. Check that the fix did not break the feature.

**Good debugging practice:** Compare heap dumps and object counts. A lower memory number alone is not proof; the important result is that the objects whose lifetimes ended are no longer retained.

---

## 8. Special considerations for Jetpack Compose

Compose does not eliminate memory leaks. A composable can still register callbacks, capture a Context, retain state, or launch work that outlives the intended UI.

Checklist:

- Use `DisposableEffect` for subscriptions that need explicit unregistration.
- Use `LaunchedEffect` for coroutine work tied to composition and keyed inputs.
- Use `rememberCoroutineScope()` for event-driven work tied to that composition.
- Use lifecycle-aware Flow collection, such as `collectAsStateWithLifecycle()`.
- Do not store Activity, View, or Activity Context in long-lived state holders or ViewModels.
- Bound caches and avoid retaining old composable lambdas or UI objects in singleton objects.
- When investigating a leak, inspect the Activity hosting the Compose tree as well as state holders and captured references.

---

## 9. Memory leak prevention checklist

Use this checklist during code review and testing.

- [ ] No singleton/static field stores an Activity, Fragment, View, binding, or short-lived Context.
- [ ] Fragment ViewBinding is cleared in `onDestroyView()`.
- [ ] ViewModels hold UI state and business data, not Views or Activity references.
- [ ] Every listener, observer, receiver, or callback has a clear registration/unregistration pair.
- [ ] Delayed Handler callbacks are removed when they are no longer needed.
- [ ] Activity-specific coroutines use lifecycle-aware scopes; obsolete jobs are cancelled.
- [ ] Long-lived Flows are collected with lifecycle-aware APIs from UI owners.
- [ ] Compose effects clean up subscriptions in `onDispose` where necessary.
- [ ] Caches and collections have deliberate size and lifetime policies.
- [ ] Services stop when their work is complete.
- [ ] Large images, Bitmaps, media buffers, and WebViews are managed according to their lifecycle.
- [ ] The same navigation/rotation test has been repeated after the fix.
- [ ] Heap dumps or LeakCanary traces have been checked instead of relying only on the memory graph.

---

## 10. Top 10 Android memory leak interview questions and answers

### Q1. What is a memory leak in Android?

**Answer:** A memory leak happens when an object that is no longer needed remains strongly reachable through another object. Because it is still reachable from a GC root, ART cannot reclaim it. For example, a singleton retaining a destroyed Activity can keep the Activity and its View hierarchy in memory. Repeated leaks can increase memory pressure, cause more GC work, jank, and eventually contribute to an OutOfMemoryError.

### Q2. Why can't Garbage Collector remove a leaked Activity?

**Answer:** GC does not decide that an object is unused based only on whether the screen is visible. It checks reachability from GC roots. If a static field, singleton, thread, callback, or another long-lived object still references the Activity, the Activity is reachable and cannot be collected. I would inspect the heap dump's reference path to find and remove the unnecessary reference.

### Q3. What are the common causes of memory leaks in Android?

**Answer:** Common causes include storing Activity or View references in singletons, keeping a Fragment's ViewBinding after `onDestroyView()`, forgetting to unregister listeners or receivers, delayed Handler callbacks that capture an Activity, coroutines that outlive their UI owner, ViewModels holding Views or Activity Context, unbounded caches, and incorrectly managed WebViews or services. I prevent them by aligning each resource's lifetime with its owner and cleaning it up at the correct lifecycle boundary.

### Q4. Why should Fragment ViewBinding be set to null in `onDestroyView()`?

**Answer:** A Fragment instance can remain alive after its View is destroyed, for example while it is on the back stack. ViewBinding holds references to the View hierarchy. If the Fragment keeps the binding, it can retain the old Views and their Activity Context. Clearing the binding in `onDestroyView()` releases those references when the View lifecycle ends.

### Q5. How can a Handler cause a memory leak?

**Answer:** A delayed message or Runnable can remain in the main thread's message queue. If the Runnable is an inner class or lambda that captures an Activity, the queue can keep that Activity reachable until the callback executes or is removed. I keep the Runnable when needed and call `removeCallbacks()` at the appropriate lifecycle boundary. For longer work, I use a suitable lifecycle-aware coroutine or background-work API.

### Q6. Can a ViewModel cause a memory leak?

**Answer:** Yes, if it holds an Activity, Fragment, View, or ViewBinding. A ViewModel may survive Activity recreation, so it can retain an obsolete UI instance. I keep ViewModels focused on data and UI state, use repositories for data access, and expose observable state for the Activity or Fragment to render. An application context can be appropriate for some app-wide operations, but a ViewModel should not keep a UI context.

### Q7. How do you detect a memory leak in Android?

**Answer:** I first reproduce the issue with a repeatable user journey, such as opening and closing a screen several times or rotating the device. I use LeakCanary to detect retained lifecycle objects and inspect its leak trace. I also use Android Studio Memory Profiler to record allocations, capture heap dumps, inspect object counts and references, and compare snapshots. For process-level memory triage, I can use `adb shell dumpsys meminfo <package>`. I verify the fix by repeating the same test.

### Q8. What is LeakCanary, and how does it work?

**Answer:** LeakCanary is a library used during Android development to detect objects that remain retained after their expected lifecycle ends. It watches objects such as destroyed Activities, Fragments, Fragment Views, and cleared ViewModels. If an object remains retained, LeakCanary can capture a heap dump, analyse the reference path, and report a leak trace that helps identify the reference preventing collection. It identifies likely leaks; the developer still needs to understand and fix the ownership problem.

### Q9. Does using WeakReference solve memory leaks?

**Answer:** Not generally. WeakReference can be appropriate when a reference is intentionally non-owning, but replacing a strong reference with a weak one can hide the real lifecycle bug and may allow an object to disappear while the app still needs it. I first identify why the strong reference outlives the object. I then unregister callbacks, cancel work, remove references, or correct object ownership. I use WeakReference only when the design specifically calls for it.

### Q10. How do you confirm that a memory leak is fixed?

**Answer:** I reproduce the same user journey before and after the fix. I capture a baseline and a heap dump after repeating the journey, inspect instances of the destroyed Activity, Fragment, View, or other target object, and follow their reference paths. After the fix, those obsolete instances should no longer remain retained, and their counts should return close to the expected baseline. I also confirm that the feature still behaves correctly and monitor production memory metrics where available. A memory graph staying high by itself does not mean the leak remains, because the runtime can retain heap capacity for future allocations.

---

## 11. A strong interview answer structure

When asked, **“How did you identify and fix a memory leak in your Android application?”**, answer in this order:

1. **Problem:** Explain which screen or object was retained and how you noticed it.
2. **Reproduction:** Describe the repeatable action, such as opening and closing a screen several times.
3. **Investigation:** Mention Android Studio Memory Profiler, heap dump, LeakCanary, and the reference path you inspected.
4. **Root cause:** Identify the actual strong reference, such as a singleton, callback, listener, Handler, or Fragment binding.
5. **Fix:** Explain the lifecycle-correct cleanup or ownership change.
6. **Verification:** Repeat the same scenario and confirm the obsolete objects are no longer retained.

Do not claim a percentage improvement unless you measured it. A clear root cause, fix, and verification are stronger than an unsupported performance number.

---

## 12. Quick revision notes

- A memory leak is usually an **object lifetime / reference ownership** problem.
- GC collects unreachable objects; it cannot collect an object that is still strongly reachable.
- A destroyed Activity retained by a singleton is a classic Android leak.
- A Fragment can outlive its View; clear ViewBinding in `onDestroyView()`.
- Pair every listener/observer registration with the correct unregistration.
- Remove delayed Handler callbacks when they are no longer needed.
- Use `lifecycleScope` for Activity lifecycle work and `viewModelScope` for ViewModel-owned work.
- Do not store Activity, View, or binding references in a ViewModel.
- Use `DisposableEffect` to clean up Compose subscriptions.
- Do not use WeakReference as a blanket leak fix.
- LeakCanary gives leak traces; Android Studio heap dumps help inspect object counts and references.
- Reproduce, inspect, fix, and repeat the same test to verify.

---

## 13. Official documentation and further reading

1. [Android — Manage your app's memory](https://developer.android.com/topic/performance/memory/manage-app-memory)
2. [Android — Capture a heap dump](https://developer.android.com/studio/profile/capture-heap-dump)
3. [Android — Understanding and troubleshooting memory](https://developer.android.com/topic/performance/memory/guide)
4. [Android — Better performance through threading](https://developer.android.com/topic/performance/threads)
5. [Android — Lifecycle-aware components](https://developer.android.com/topic/libraries/architecture/lifecycle)
6. [Android — WebView memory management](https://developer.android.com/develop/ui/views/layout/webapps/manage-webview-memory)
7. [LeakCanary — How LeakCanary works](https://github.com/square/leakcanary/blob/main/docs/fundamentals-how-leakcanary-works.md)
8. [LeakCanary — Fixing a memory leak](https://github.com/square/leakcanary/blob/main/docs/fundamentals-fixing-a-memory-leak.md)
9. [LeakCanary — Official project and setup](https://github.com/square/leakcanary)

> **Study recommendation:** First understand reachability and reference paths. Then learn the Activity, Fragment, Handler, listener, coroutine, and Compose examples. Finally, reproduce a leak in a small sample app and practise reading a LeakCanary trace. This sequence builds practical skill rather than memorising definitions.
