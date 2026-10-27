# Android Memory Management and Optimization

> Beginner-friendly study notes based on the Android Developers documentation.  
> Last reviewed: 27 October 2026.

## Learning goals

By the end of this note, you should be able to:

- Explain how Android uses RAM, zRAM, storage, and memory pages.
- Explain managed memory and garbage collection (GC) in ART.
- Recognize common Android memory leaks and understand why GC cannot fix them.
- Explain what happens when Android is under memory pressure.
- Reduce memory use from images, libraries, services, and data structures.
- Use Android Studio Memory Profiler, heap dumps, LeakCanary, and ADB to investigate problems.
- Explain the difference between an Android process and a thread.
- Describe Android app memory budgets and their platform requirements.

---

## 1. What does memory management mean?

An Android app needs memory for its Activity and Fragment objects, ViewModels, UI views, Kotlin/Java objects, images, database results, media buffers, libraries, and background work.

**Memory management** means using only the memory the app needs, releasing references and resources when they are no longer needed, and being prepared for Android to reclaim or terminate a background process when the device needs RAM.

A useful distinction:

- **Memory leak:** An object is no longer useful, but something still references it, so it cannot be garbage-collected.
- **High memory usage:** The app is using a lot of memory. This may be intentional, such as a large image cache, and is not automatically a leak.
- **OutOfMemoryError (OOM):** The app cannot satisfy a memory allocation, commonly because its managed heap has reached its allowed capacity. Native or other allocation failures can also occur in their respective memory areas.

Android performs automatic garbage collection for managed objects, but developers still need to manage object lifetimes and non-memory resources.

## 2. RAM, zRAM, storage, and memory pages

Android documentation describes three broad memory types:

| Type | Simple explanation | Example |
|---|---|---|
| RAM | Fast physical memory used by the CPU and GPU while the device is running | Objects, decoded images, active app data |
| zRAM | A portion of RAM used as compressed swap space | Some inactive memory pages are compressed to make physical RAM available |
| Storage | Persistent, larger-capacity space | App files, Room database, DataStore files, photos |

Storage is not used as ordinary swap space on Android in the way it is on some other Linux systems; frequent writes can wear storage. zRAM is compressed RAM, not extra physical RAM.

### Memory pages

The operating system manages RAM in small blocks called **pages**. Android documentation says a page is typically 4 KB, although the actual page size is device-dependent.

- A **clean file-backed page** matches data already stored in a file. The system can discard it and read it again later.
- A **dirty page** has been modified. It cannot simply be discarded if doing so would lose data.
- Some dirty pages can be compressed and moved to zRAM.

**Remember:** Android may use most available RAM for useful caching. A low amount of “free RAM” by itself does not prove that an app has a memory problem.

## 3. Managed heap and Garbage Collection (GC)

Kotlin/JVM objects are generally allocated in the managed heap. Android Runtime (ART) tracks managed allocations and can reclaim objects that are no longer reachable from the app's live references.

### Reachability: the key idea

An object is eligible for GC when it can no longer be reached from live references (for example, references held by active threads, static fields, or other reachable objects).

```kotlin
class User(val name: String)

fun example() {
    var user: User? = User("Anand")

    // The local variable currently references the User object.
    user = null

    // If no other live reference points to this User object,
    // it is now eligible for garbage collection.
    // This does NOT mean GC runs immediately.
}
```

Assigning `null` only removes that particular reference. If another object still references the `User`, it remains reachable.

### How GC works (high-level)

1. ART tracks managed object allocations.
2. The runtime determines which objects are still reachable.
3. Objects that cannot be reached are considered garbage.
4. GC reclaims the memory occupied by those objects for future allocations.

Android's documentation describes its managed heap as generational: objects are tracked according to expected lifetime, and objects that remain active can move into older generations. Exact GC implementation details can vary by Android Runtime version.

### Why can GC affect performance?

GC consumes CPU time. If the app creates many short-lived objects in a hot path—such as repeatedly allocating objects during every animation frame—GC may run more often. This can contribute to missed frames or jank.

**Good practice:** Avoid unnecessary allocations in frequently executed code, especially drawing, scrolling, and animation paths. Measure before optimizing.

## 4. Memory leaks in Android

A memory leak occurs when an object that should be released remains reachable through an unnecessary reference.

### Example: a singleton holding an Activity

**Problematic code:**

```kotlin
object ActivityHolder {
    var activity: Activity? = null
}

class LoginActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        ActivityHolder.activity = this
    }
}
```

The singleton can live as long as the process. It holds the `LoginActivity`, so the Activity may remain reachable even after it is destroyed.

**Better approach:** Do not store Activity or View references in long-lived singletons. Pass only the data or behavior needed. When a long-lived object genuinely needs a Context, use the application context only when its behavior is appropriate for application scope.

### Common leak patterns

| Cause | Why it can leak | Better practice |
|---|---|---|
| Singleton/static field stores Activity or View | Long-lived owner keeps short-lived UI alive | Avoid storing UI references in global objects |
| Listener or callback not unregistered | The listener owner retains the screen | Register and unregister using the correct lifecycle |
| Handler/Runnable with delayed work | Pending callback may retain an Activity | Cancel callbacks when the work is no longer needed |
| Fragment ViewBinding retained after view destruction | Fragment can outlive its view | Clear binding in `onDestroyView()` |
| Coroutine launched in an inappropriate scope | Work can outlive the screen that started it | Use lifecycle-aware scopes or cancel the job |
| Large image/media cache never cleared | Unneeded image or media memory stays retained | Bound caches and release resources when appropriate |

### Fragment ViewBinding example

A Fragment can remain alive after its View is destroyed. Therefore, clear the binding when the view lifecycle ends:

```kotlin
private var _binding: FragmentHomeBinding? = null
private val binding get() = requireNotNull(_binding)

override fun onCreateView(
    inflater: LayoutInflater,
    container: ViewGroup?,
    savedInstanceState: Bundle?
): View {
    _binding = FragmentHomeBinding.inflate(inflater, container, false)
    return binding.root
}

override fun onDestroyView() {
    super.onDestroyView()
    _binding = null
}
```

## 5. What happens when Android is low on memory?

Android manages memory for the entire device, not just one application. The system tries to keep useful apps and cached data in RAM so users can return to apps quickly.

Two mechanisms described in Android's memory documentation are:

### `kswapd` — kernel swap daemon

- It is part of the Linux kernel.
- It becomes active when free RAM falls below a system threshold.
- It can reclaim clean file-backed pages because those can be loaded again from storage.
- It can move certain dirty or anonymous pages to zRAM, where they are compressed.
- If a process needs a compressed page again, the page can be decompressed and returned to RAM.

### Low-memory killing

If reclaiming and compression are not enough, Android may terminate processes to free memory. The system considers process importance, including whether an app has a visible Activity or is doing user-perceptible work. Cached/background processes are generally more likely to be killed than a foreground app.

Modern Android implementations use the low-memory-killer infrastructure (commonly referred to as `lmkd`); Android's developer page also explains the low-memory killer concept and OOM adjustment priority.

### Developer responsibility

Do not assume that a background process will remain alive forever. Persist important application data and make screens recoverable after process recreation.

- Use Room or files for persistent application data.
- Use DataStore for persistent preferences/settings.
- Use `SavedStateHandle` or saved instance state for small pieces of UI state that should survive process recreation.
- Do not use a ViewModel as permanent storage; its lifetime does not survive process death.

## 6. `onTrimMemory()` and releasing resources

`ComponentCallbacks2.onTrimMemory()` gives an app a chance to reduce memory use in response to lifecycle or memory-related conditions.

Current Android guidance focuses on these callbacks:

- `TRIM_MEMORY_UI_HIDDEN`: The app's UI is no longer visible. Release heavy UI-only resources that can be recreated, such as custom bitmap caches, video buffers, or animation caches.
- `TRIM_MEMORY_BACKGROUND`: The process is in the background and is a candidate for termination. Release additional reconstructible resources where appropriate.

**Compatibility note:** Beginning with Android 14, the system no longer delivers the other legacy trim-level notifications; those older constants were formally deprecated in Android 15. Do not build new logic around the old trim-level values.

Example:

```kotlin
class MainActivity : AppCompatActivity(), ComponentCallbacks2 {

    override fun onTrimMemory(level: Int) {
        super.onTrimMemory(level)

        when {
            level >= ComponentCallbacks2.TRIM_MEMORY_BACKGROUND -> {
                // Release reconstructible background caches/resources.
            }
            level >= ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN -> {
                // Release heavy resources used only by the hidden UI.
            }
        }
    }
}
```

The example shows where cleanup can happen; actual cleanup depends on your app. Do not clear essential user data or resources that are actively in use. Avoid holding Activity/Context references in long-lived managers just to perform cleanup.

## 7. Practical ways to reduce memory usage

### 7.1 Load images at the size you need

A compressed image file can expand significantly when decoded into a bitmap. For a typical `ARGB_8888` bitmap, each pixel uses 4 bytes (red, green, blue, alpha).

Example for a 1000 × 1000 image:

```text
1000 × 1000 pixels × 4 bytes = 4,000,000 bytes ≈ 4 MB
```

A 100 KB JPEG can therefore require about 4 MB as a decoded 1000 × 1000 `ARGB_8888` bitmap. The compressed file size is not the same as decoded in-memory size.

Good practices:

- Decode/load at a size close to the displayed size.
- Use an image-loading library such as Coil or Glide rather than manually retaining many decoded bitmaps.
- Use bounded memory caches.
- Release custom bitmap references and media buffers when they are no longer needed.
- Avoid loading a very large list of full-resolution images at once.

### 7.2 Use memory-efficient data structures where appropriate

For integer-keyed mappings, Android offers `SparseArray`, `SparseBooleanArray`, and `LongSparseArray`. They can reduce object overhead compared with a regular `HashMap<Integer, ...>` in suitable cases by avoiding some key/value boxing and per-entry objects.

```kotlin
val enabledById = SparseBooleanArray()
enabledById.put(10, true)
enabledById.put(20, false)
```

Do not blindly replace every `HashMap`. Choose a structure based on the access pattern, collection size, and measurements.

### 7.3 Review libraries and dependency injection

- Remove unused dependencies and duplicate libraries.
- Check whether a library has a large runtime footprint or retains objects longer than expected.
- Use correct dependency-injection scopes; an object with a longer lifetime must not retain short-lived Activities or Views.
- Google's current memory guide recommends Hilt for Android dependency injection; its compile-time approach avoids runtime reflection-based class scanning.

### 7.4 Use R8 in release builds

R8 can remove unused code, optimize bytecode, and shrink/minify identifiers. Android's current guidance notes that reducing code loaded at runtime can reduce baseline memory footprint as well as app size.

Example Gradle configuration (Groovy DSL):

```groovy
android {
    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'),
                    'proguard-rules.pro'
        }
    }
}
```

Use precise keep rules. Broad rules such as keeping every class in a whole package can prevent R8 from optimizing code. Verify release behavior because reflection, serialization, and JNI may require specific keep rules.

### 7.5 Avoid unnecessary long-running services

Do not keep a Service running when no work needs it. Long-lived services can keep a process important and reduce the memory available for other cached apps. For deferrable, guaranteed background work, consider WorkManager. Use a foreground service only for work that meets the platform's user-visible and foreground-service requirements.

### 7.6 Keep background work off the main thread

A slow operation on the main thread can block drawing and touch handling, causing jank or an ANR. Use an appropriate coroutine dispatcher for blocking I/O or CPU work.

```kotlin
lifecycleScope.launch {
    val users = withContext(Dispatchers.IO) {
        database.userDao().getAllUsers()
    }

    // Returns to the lifecycleScope's Main dispatcher.
    adapter.submitList(users)
}
```

This example assumes `lifecycleScope` is using its normal Main dispatcher and the DAO call is a blocking I/O operation. Prefer Room suspend APIs where possible.

## 8. Processes and threads

### Process

A process is an operating-system execution environment with its own virtual address space and process-level resources. Android starts a Linux process for an app when needed. By default, an app's Activities, Services, BroadcastReceivers, and ContentProviders run in the same process.

### Thread

A thread is a path of execution inside a process. Threads in the same process share the process's memory space, although each thread has its own execution stack and thread-specific state.

### Main thread

Android creates a main thread for an app process. It dispatches UI events and drawing work and is commonly called the UI thread. Long-running work on this thread can block UI rendering.

### Worker threads and coroutines

Use background execution for I/O and CPU-intensive work. Coroutines are a structured-concurrency tool; they are not themselves threads. A coroutine runs on a dispatcher, which determines the thread or threads used for execution.

### Multiple processes

You can configure a component to run in another process using `android:process` in the manifest. Separate processes have separate process memory and add overhead. Do not use multiple processes as a generic performance optimization; use them only for a justified architecture or isolation requirement.

Example:

```xml
<service
    android:name=".SyncService"
    android:process=":sync" />
```

The `:sync` name creates a private process for this app component. Components in different processes do not share ordinary in-process objects; communication requires IPC mechanisms such as Binder.

### Process death and restoration

Android can kill a process when system resources are needed. Components are later recreated when the user or system needs them. Persist durable data and restore transient UI state rather than relying on static variables or in-memory objects to survive process death.

## 9. Tools for memory investigation

### Android Studio Memory Profiler

Use it to:

- See memory usage over time.
- Inspect managed object counts and GC events.
- Record allocations and investigate where objects were allocated.
- Capture a heap dump and inspect retained objects and references.

A heap dump is a snapshot of objects in the managed heap. Use it to investigate why an Activity, Fragment, View, or other object remains reachable.

### LeakCanary

LeakCanary is a development-time library that helps detect retained objects and produces leak traces. It is useful for identifying objects that remain reachable after their expected lifecycle ends. It complements, rather than replaces, heap-dump analysis.

### ADB commands

Replace `com.example.myapp` with your package name.

```bash
# Detailed process memory breakdown
adb shell dumpsys meminfo com.example.myapp

# Show app process and component state
adb shell dumpsys activity processes com.example.myapp

# List matching processes
adb shell ps -A | grep com.example.myapp
```

If you know the process ID (`PID`), you can inspect Linux process metrics:

```bash
# Replace <PID> with the app process ID
adb shell cat /proc/<PID>/status | grep -E 'VmRSS|RssAnon|VmSwap'
adb shell cat /proc/<PID>/oom_score_adj
```

### Important memory metrics

- **Java/Kotlin heap:** Memory used by managed objects.
- **Native heap:** Memory allocated outside the managed heap, often by native libraries, graphics, or media components.
- **RSS (Resident Set Size):** Pages currently resident in physical memory for a process; shared pages can be counted in more than one process's RSS.
- **PSS (Proportional Set Size):** Shared pages are apportioned across processes. PSS is often more useful than RSS for estimating a process's proportional share of RAM.
- **Swap/zRAM:** Pages moved to compressed swap.

Do not judge a leak from a single memory number. Capture a repeatable workflow, compare before and after, inspect retained objects, and check whether memory returns to a stable level after screens/resources are released.

## 10. A practical memory-leak investigation workflow

Example: memory appears to increase every time a user opens and closes the Profile screen.

1. Reproduce the issue on a representative device and build.
2. Open Android Studio Memory Profiler and record a baseline.
3. Open the Profile screen, perform the same actions, then navigate away.
4. Repeat the cycle several times using the same steps.
5. Force/observe GC only as a diagnostic aid, not as a production fix.
6. Capture a heap dump after the screen should have been released.
7. Search for old Profile Fragment, View, Activity, Bitmap, or related objects.
8. Inspect the reference path that keeps the object reachable.
9. Fix the owner: unregister a listener, cancel a job, clear Fragment binding, remove a callback, or bound a cache.
10. Repeat the exact test and confirm that retained instances no longer accumulate.

**Important:** The memory graph may not immediately fall after GC. The runtime may keep reclaimed heap space for reuse. A high graph alone does not prove a leak; retained objects and reference paths provide stronger evidence.

## 11. App memory budgets (newer Android feature)

An app memory budget lets an app declare a resident-memory budget. When memory exceeds the budget, Android can reclaim inactive pages—first clean file-backed pages, then dirty file-backed pages, and then compress/swap anonymous pages to zRAM. This helps constrain the app's resident working set; it is not the same as setting the Java heap limit.

### Platform support

According to the current Android Developers documentation, `<memory-budget>` declarations are supported starting with **Android 17 QPR2 (minor SDK release, API level 37.2)**. The documentation recommends testing on Android 17 QPR2 system images/emulator and applying declarations only to optimized release builds. Older platform manifest parsers ignore the unrecognized element.

Example baseline declaration:

```xml
<application
    android:label="@string/app_name">

    <memory-budget android:maxMb="256" />

</application>
```

The `256` MB value is only an example from the documentation, not a recommended budget for every app. Measure the app's working set on target devices before choosing a value. An overly small budget can cause active pages to be reclaimed and later restored, hurting performance.

The documentation also describes different budgets for foreground, perceptible, and background process states. Read the current platform guide before using state-specific or per-process configuration because the feature is new and subject to platform-version requirements.

## 12. Top 10 Android memory interview questions and answers

### 1. What is memory management in Android?

**Answer:** Memory management is the process of using RAM efficiently, releasing resources and object references when they are no longer needed, and handling system memory pressure. Android Runtime manages garbage collection for managed objects, while the app developer must avoid leaks, control large allocations, and handle lifecycle changes correctly.

### 2. What is Garbage Collection, and how does it work?

**Answer:** Garbage Collection is the runtime process that identifies managed objects that are no longer reachable and reclaims their heap memory. ART tracks allocations and object references. When an object is no longer reachable from live references, it becomes eligible for collection. GC timing is controlled by the runtime, so setting a reference to `null` does not cause immediate collection.

### 3. What is the difference between a memory leak and an OutOfMemoryError?

**Answer:** A memory leak happens when an object is unintentionally retained even though the app no longer needs it. An OutOfMemoryError happens when an allocation cannot be satisfied in the relevant memory area. A leak can gradually contribute to an OOM, but high memory usage or an OOM can also happen without a leak, for example when loading an image that is too large.

### 4. Why can't Garbage Collection remove an Activity referenced by a singleton?

**Answer:** GC removes only objects that are unreachable. A singleton is long-lived, and if it holds an Activity reference, the Activity remains reachable through that reference. The fix is to avoid retaining Activity/View references in long-lived objects, or to remove the reference at the correct lifecycle point. Using `applicationContext` is appropriate only when application-level Context behavior is sufficient.

### 5. Why can a small JPEG use several megabytes of memory?

**Answer:** A JPEG is compressed on disk. To display it, Android decodes it into pixels. A typical `ARGB_8888` bitmap uses four bytes per pixel. A 1000 × 1000 bitmap therefore needs about 4 MB, even if its compressed JPEG file is only 100 KB. Load images near their display dimensions and use an image-loading library with sensible caching.

### 6. What is `onTrimMemory()` and when should it be used?

**Answer:** `onTrimMemory()` is a callback from `ComponentCallbacks2` that gives the app an opportunity to release reconstructible resources when lifecycle or memory conditions change. Current Android guidance focuses on `TRIM_MEMORY_UI_HIDDEN` and `TRIM_MEMORY_BACKGROUND`. Use it to release nonessential UI caches, bitmap caches, media buffers, or other resources that can be rebuilt. Do not use it to delete user data.

### 7. What happens when Android runs low on RAM?

**Answer:** The kernel first tries to reclaim memory. `kswapd` can discard clean file-backed pages and move/compress certain dirty or anonymous pages into zRAM. If that is not enough, Android can terminate processes according to their importance to the user. Cached/background processes are generally more likely to be killed than a process hosting visible UI. Apps must persist important data and restore their state after process recreation.

### 8. What is the difference between a process and a thread in Android?

**Answer:** A process is an operating-system execution environment with its own virtual address space and resources. A thread is a path of execution within a process. Threads in the same process share the process memory, although each has its own stack. By default, an Android app's components run in one process, and UI work is dispatched on the main thread. Extra processes have separate memory and communication overhead.

### 9. How do you detect and fix a memory leak in an Android app?

**Answer:** First reproduce the same screen/navigation workflow repeatedly. Use Android Studio Memory Profiler to observe allocations and capture a heap dump. Inspect retained Activities, Fragments, Views, or other objects and follow the reference path that keeps them alive. LeakCanary can help identify retained objects. Fix the owner—for example, clear Fragment binding in `onDestroyView()`, unregister listeners, cancel lifecycle-inappropriate work, remove callbacks, or bound caches. Repeat the test to verify the fix.

### 10. What is an app memory budget?

**Answer:** An app memory budget is a declared resident-memory target that allows Android to reclaim inactive pages when the app exceeds its budget. It is not the same as the Java heap limit. Current Android documentation says manifest memory budgets are supported from Android 17 QPR2 (API 37.2). Developers should measure the app's working set and test carefully; a budget that is too low can cause extra page reclaim/swap and hurt performance.

---

## 13. Quick revision checklist

- [ ] I can explain RAM, zRAM, storage, and memory pages.
- [ ] I understand object reachability and why GC cannot fix reachable leaks.
- [ ] I can explain Activity, Fragment, Context, listener, and coroutine leak patterns.
- [ ] I know the purpose of `kswapd`, zRAM, and low-memory process killing.
- [ ] I can explain the current `onTrimMemory()` guidance.
- [ ] I understand why decoded bitmaps can be much larger than compressed image files.
- [ ] I know when `SparseArray` may be useful and why measurement matters.
- [ ] I understand process vs thread and why background work should not block the main thread.
- [ ] I can use Memory Profiler, heap dumps, LeakCanary, and ADB commands.
- [ ] I understand the new Android app memory-budget feature and its platform requirement.

## 14. Official references

1. [Android memory — overview and learning path](https://developer.android.com/topic/performance/memory)
2. [Overview of memory management](https://developer.android.com/topic/performance/memory-overview)
3. [Memory allocation among processes](https://developer.android.com/topic/performance/memory-management)
4. [Manage your app's memory](https://developer.android.com/topic/performance/memory/manage-app-memory)
5. [Processes and threads overview](https://developer.android.com/guide/components/processes-and-threads)
6. [Set app memory budgets](https://developer.android.com/topic/performance/memory/app-memory-budgets)
7. [Android Studio Memory Profiler](https://developer.android.com/studio/profile/memory-profiler)
8. [LeakCanary](https://github.com/square/leakcanary)

---

## Suggested next study topic

Study **Garbage Collection in ART** in more depth: heap allocation, object reachability, GC roots, generational collection, allocation churn, and how to read a heap dump. Then practice finding a Fragment or Activity leak with LeakCanary and Android Studio Memory Profiler.
