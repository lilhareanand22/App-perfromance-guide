# Garbage Collection (GC) in Android ART

A beginner-friendly study guide with Kotlin examples, debugging steps, and interview questions.

> **Big idea:** Garbage Collection (GC) is the Android Runtime's automatic way of reclaiming memory used by managed objects that the application can no longer reach. GC does not fix memory leaks caused by references that are still being held.

---

## Table of contents

1. [What is Garbage Collection?](#1-what-is-garbage-collection)
2. [Where do Kotlin and Java objects live?](#2-where-do-kotlin-and-java-objects-live)
3. [How does GC know an object is no longer needed?](#3-how-does-gc-know-an-object-is-no-longer-needed)
4. [GC roots and object reachability](#4-gc-roots-and-object-reachability)
5. [How ART Garbage Collection works](#5-how-art-garbage-collection-works)
6. [Generational Garbage Collection](#6-generational-garbage-collection)
7. [When does GC run?](#7-when-does-gc-run)
8. [GC pauses, allocation churn, and UI performance](#8-gc-pauses-allocation-churn-and-ui-performance)
9. [Garbage Collection vs. Memory Leak vs. OutOfMemoryError](#9-garbage-collection-vs-memory-leak-vs-outofmemoryerror)
10. [Kotlin and Android examples](#10-kotlin-and-android-examples)
11. [How to investigate GC and heap problems](#11-how-to-investigate-gc-and-heap-problems)
12. [Practical optimization checklist](#12-practical-optimization-checklist)
13. [Top 10 interview questions and answers](#13-top-10-interview-questions-and-answers)
14. [Quick revision sheet](#14-quick-revision-sheet)
15. [Official references](#15-official-references)

---

## 1. What is Garbage Collection?

Kotlin and Java applications create objects while they run. For example, an app can create a `User`, a list of transactions, a screen's view objects, or an image-related object.

These objects require memory. When an object is no longer needed, its memory should be made available for future work.

In a managed runtime such as Android Runtime (ART), the developer normally does not free each object manually. ART tracks managed object references and runs **Garbage Collection** to reclaim objects that are no longer reachable.

### Simple example

```kotlin
class User(val name: String)

fun example() {
    var user: User? = User("Anand")

    // The variable refers to a User object.
    println(user?.name)

    // Remove this particular reference.
    user = null

    // If no other live reference points to that User,
    // the object is eligible for garbage collection.
}
```

Important points:

- Setting a reference to `null` does **not** immediately run GC.
- The object is eligible for collection only if no other live reference can reach it.
- ART decides when and how to perform collection.
- The memory may be reused by the app after collection; process memory shown by system tools does not have to drop immediately.

Think of GC as a cleanup service. It can remove objects that nobody can reach, but it cannot decide that an object is unnecessary while your code still holds a reference to it.

---

## 2. Where do Kotlin and Java objects live?

At a high level, an Android process uses several kinds of memory:

| Memory area | What it commonly contains | Is it managed by Java/Kotlin GC? |
|---|---|---|
| Managed heap | Kotlin/Java objects such as `User`, collections, Activities, and many framework objects | Yes |
| Native memory | Native library allocations, some graphics/media resources, and other C/C++ allocations | Not directly as ordinary managed objects |
| Thread stacks | Method-call frames, local primitive values, and references used by threads | Stack lifetime is managed as calls return; references on stacks are relevant to GC reachability |
| Mapped/file-backed memory | App code, libraries, and mapped files | Not ordinary Java heap objects |

A Kotlin variable that refers to an object is not the object itself. It is a reference to an object. The object usually occupies the managed heap; the reference may be held in a stack frame, a field of another object, or a static/class-level field.

### Important distinction: Java heap is not total app memory

The Memory Profiler's Java/Kotlin heap view is not the same as the entire process memory footprint. An app can use significant native, graphics, or mapped memory too. Investigate the correct memory category instead of assuming every memory increase is a Java heap leak.

---

## 3. How does GC know an object is no longer needed?

GC uses **reachability**.

In simple terms, ART starts from a set of known live references (GC roots) and follows references from one object to another. Objects that can be reached are treated as live. Objects that cannot be reached are candidates for collection.

### Example

```kotlin
class Address(val city: String)
class Customer(val address: Address)

fun example() {
    val address = Address("Pune")
    val customer = Customer(address)

    println(customer.address.city)
}
```

While `customer` is reachable, its `address` field can lead GC to the `Address` object too. The fact that there is no separate local variable being used for `address` does not make the Address object collectible: it is still reachable through `customer`.

### Object graph

```text
Live GC root
    |
    v
 Customer object
    |
    | address field
    v
 Address object
```

Both objects are reachable in this example.

If the root reference to `Customer` disappears and no other root can reach either object, both objects can become eligible for collection.

---

## 4. GC roots and object reachability

A **GC root** is a starting point from which the runtime checks which objects are still reachable.

Examples of roots or root-like references include:

- References held by active thread stacks and CPU registers.
- Static or class-level fields that are still loaded and reachable.
- References maintained by the runtime, such as certain JNI global references.
- Other runtime-managed references that must remain alive.

The exact implementation details are runtime-specific, but the practical lesson is simple: an object can stay alive when a long-lived object, active thread, callback, or framework component still refers to it.

### Example: a memory leak caused by a singleton

```kotlin
object ScreenStore {
    var activity: Activity? = null
}

class ProfileActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        ScreenStore.activity = this
    }
}
```

The singleton can live for the lifetime of the process. It holds the Activity reference. Even after the Activity is destroyed, the reference remains unless it is cleared. Therefore, the Activity can remain reachable and cannot be collected.

**Better design:** Avoid storing Activity, Fragment, View, or Activity-context references in process-long singletons. Use application context only when the operation genuinely needs a long-lived Context, and clear lifecycle-bound references at the correct lifecycle event.

### Strong and weak references

- A normal Kotlin/Java reference is strong. If a live object strongly refers to another object, that reference can keep the target reachable.
- A `WeakReference` does not by itself keep its target alive. The target may be collected when there are no strong references to it.
- Weak references are useful for some caches or observer designs, but they are not a universal memory-leak fix. Correct lifecycle ownership is usually better.

---

## 5. How ART Garbage Collection works

ART is the Android Runtime that executes Android app code. It manages the managed heap and performs GC.

GC implementations can differ across Android releases and device builds. The Android Open Source Project documents the following important ART history and design points:

- Starting with Android 8.0 (Oreo), the default ART GC plan is Concurrent Copying (CC), although device manufacturers can change runtime configuration.
- Concurrent Copying is designed to do much of its work while app threads continue running.
- It uses a read barrier and can compact the heap by copying live objects, helping reduce fragmentation.
- Android 10 and later support generational mode for Concurrent Copying, which can collect young objects efficiently.
- ART also has support for other collector plans, so do not assume every device and release uses exactly the same collector configuration.

### Conceptual GC phases

The exact phases depend on the collector. A simplified way to understand tracing GC is:

1. **Identify roots:** Find the starting references that must be treated as live.
2. **Trace references:** Follow references from roots to discover reachable objects.
3. **Determine garbage:** Objects that are not reachable are candidates for reclamation.
4. **Reclaim or reuse memory:** The runtime makes space available for future allocations.
5. **Optional copying/compaction:** Some collectors move live objects to reduce fragmentation and create usable free space.

This is a learning model, not a promise that ART performs one identical stop-the-world sequence for every GC event. Modern ART collection is largely concurrent, with short pauses still possible for coordination.

### What does “concurrent” mean?

Concurrent means GC work can happen while application threads (often called mutator threads) continue executing for much of the collection. It does **not** mean there are never pauses. The runtime may briefly pause threads at specific points to get a consistent view or coordinate work.

### What does “compaction” mean?

Imagine free memory split into many small gaps between live objects. A large object might not fit into any individual gap even though the total free space is sufficient. Compaction moves live objects closer together so free space becomes less fragmented. Concurrent Copying can perform this kind of work while reducing the time app threads are stopped.

---

## 6. Generational Garbage Collection

Many objects live for a very short time. For example:

- Temporary strings created while formatting a label.
- Short-lived API mapping objects.
- Temporary lists built during a calculation.
- Objects created while processing one UI event.

Other objects may live much longer:

- A repository scoped to the application.
- A cache intentionally shared across screens.
- A ViewModel while its owner remains alive.

Generational GC uses the observation that many newly created objects become unreachable quickly.

### Young and old objects

A simplified model:

```text
New allocation
      |
      v
 Young generation
      |
      | survives collections
      v
 Older generation
      |
      v
 Full-heap collection may be needed
```

- **Young collection:** Focuses on recently allocated objects. It can be efficient because many young objects are already unused.
- **Full-heap collection:** Examines a broader portion of the heap, including longer-lived objects. It can cost more because more live objects may need to be considered.

ART's exact generation organization and promotion policy are implementation details and can evolve. Use “young” and “full-heap” as the useful ART concepts rather than assuming a traditional JVM Permanent Generation. **Android ART does not use the old JVM PermGen model as an app-object generation.**

### Why is generational GC useful?

Suppose a screen creates 10,000 temporary objects while rendering a list, but only 100 objects remain in use. A young collection can focus on the temporary objects without needing to do a full-heap collection every time.

Generational GC improves efficiency; it does not excuse excessive allocations. Large amounts of short-lived object creation can still create GC work and performance overhead.

---

## 7. When does GC run?

There is no reliable rule such as “GC runs every 60 seconds” or “GC runs immediately after an Activity closes.” ART makes the decision based on runtime conditions.

GC may be triggered or scheduled when, for example:

- The heap has reached a threshold or is under allocation pressure.
- The app allocates objects quickly and free space is becoming limited.
- The runtime determines that collection is useful for heap management.
- A diagnostic or developer tool explicitly requests collection.

The exact thresholds and scheduling policy are implementation details. Application code should not depend on a fixed GC schedule.

### Should we call `System.gc()` manually?

Usually, no. Calling `System.gc()` is only a request/hint to the runtime; it does not guarantee that collection happens immediately or that a particular object is collected. Calling it in normal app flows can add unnecessary work and cause unpredictable performance.

Use profiling tools to investigate memory. Fix object ownership and allocation patterns instead of trying to force GC.

---

## 8. GC pauses, allocation churn, and UI performance

GC is automatic, but it is not free.

When an app creates many temporary objects, ART must spend CPU time managing and collecting them. Some GC coordination phases can pause app threads briefly. If that work happens during a busy UI period, it may contribute to dropped frames or jank.

### Frame budget example

At 60 frames per second, one frame has about 16.7 ms to complete. If UI work, rendering, scheduling, and GC-related pauses push a frame beyond its deadline, the user may see a stutter.

### Allocation churn example

```kotlin
fun updateLabels(values: List<Int>) {
    for (value in values) {
        // Creates a new String on each iteration.
        val label = "Value: $value"
        render(label)
    }
}
```

This code may be perfectly fine for a small list or an occasional operation. But if similar work happens repeatedly for thousands of items during every frame, it can create unnecessary allocations.

### What should we do?

- Measure first; do not optimize harmless allocations based on guesses.
- Avoid creating large temporary collections in hot loops.
- Reuse buffers when it clearly reduces repeated allocation and keeps code maintainable.
- Do not repeatedly decode the same large images.
- Page or stream large datasets instead of loading everything at once.
- Keep expensive calculations and I/O off the Main Thread.
- Profile scrolling and animations under realistic conditions.

**Important:** The goal is not “zero allocations.” The goal is to avoid wasteful allocation patterns that measurably hurt memory or frame performance.

---

## 9. Garbage Collection vs. Memory Leak vs. OutOfMemoryError

These terms are related but different.

| Term | Meaning | Example |
|---|---|---|
| Garbage Collection | Runtime reclaims managed objects that are no longer reachable | A temporary object has no remaining references |
| Memory leak | An object that is no longer logically needed remains reachable | A singleton still holds a destroyed Activity |
| OutOfMemoryError (OOM) | An allocation cannot be satisfied under the runtime/system's memory constraints | The app tries to allocate a very large array or Bitmap |
| Allocation churn | App creates many short-lived objects in a short period | Temporary objects created on every animation frame |

### Does GC prevent memory leaks?

No. GC can collect unreachable objects. A leaked object is often still reachable through an unintended reference chain, so GC correctly keeps it alive.

### Does high memory usage always mean a leak?

No. An app may intentionally cache data, the runtime may retain heap capacity for later allocations, or memory may be in native/graphics areas rather than the managed heap. Compare repeated snapshots, inspect retained objects and reference paths, and check the relevant process memory categories.

### Why might memory not immediately drop after GC?

GC can reclaim objects for reuse by the app without returning all heap pages to the operating system. The runtime may keep reserved/committed memory available to avoid expensive future allocations. Process RSS/PSS therefore may not fall in direct proportion to the number of objects collected.

---

## 10. Kotlin and Android examples

### Example A: Object becomes eligible for GC

```kotlin
class Report(val title: String)

fun createReport() {
    var report: Report? = Report("Monthly report")
    println(report?.title)

    report = null
    // If this was the last strong reference, the object is eligible for GC.
    // GC timing is decided by ART.
}
```

### Example B: A singleton accidentally retains an Activity

**Problem:**

```kotlin
object AppCache {
    var currentActivity: Activity? = null
}

class DetailsActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        AppCache.currentActivity = this
    }
}
```

**Better:** Do not put an Activity reference in a process-wide singleton. Store only the data the singleton needs. If a long-lived object genuinely needs a Context for an application-wide operation, use `applicationContext` where appropriate—not as a blanket replacement for lifecycle-aware design.

### Example C: Clear Fragment ViewBinding when the view is destroyed

A Fragment can outlive its view. Clear the binding in `onDestroyView()` so the Fragment does not keep the old view hierarchy reachable.

```kotlin
class HomeFragment : Fragment() {

    private var _binding: FragmentHomeBinding? = null
    private val binding get() = requireNotNull(_binding)

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = FragmentHomeBinding.inflate(
            inflater, container, false
        )
        return binding.root
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

### Example D: Cancel lifecycle-bound work

Prefer lifecycle-aware scopes for UI work. A coroutine launched in `viewLifecycleOwner.lifecycleScope` is cancelled when the Fragment's view lifecycle is destroyed.

```kotlin
viewLifecycleOwner.lifecycleScope.launch {
    viewModel.uiState.collect { state ->
        // Update the view while its lifecycle is active.
    }
}
```

A lifecycle scope helps prevent work from outliving the owner, but it does not automatically fix every leak. A singleton, callback, or another object can still retain a screen if it is designed incorrectly.

### Example E: Avoid keeping an unnecessary large list

```kotlin
// Risky for a very large dataset if the entire result is retained.
val allTransactions: List<Transaction> = repository.loadAll()

// Better design for large datasets:
// use paging or load only the records needed by the current screen.
```

For a large RecyclerView or Compose list, consider Paging 3, database paging, and bounded caches rather than loading an unlimited dataset into memory.

---

## 11. How to investigate GC and heap problems

### Tool 1: Android Studio Memory Profiler

Use it to:

- View memory usage over time.
- See Java/Kotlin object counts and allocations.
- Observe GC events on the timeline.
- Record allocation activity and inspect where objects were created.
- Trigger a diagnostic GC and capture a heap dump.

Official guide: [Profile your app's memory usage](https://developer.android.com/studio/profile/memory-profiler)

### Tool 2: Heap Dump

A heap dump is a snapshot of objects in the managed heap at a particular time. Use it to answer:

- Which classes have many live instances?
- Are Activity or Fragment instances remaining after navigation?
- What is keeping a particular object alive?
- Which reference path leads from a GC root to the object?
- What is the object's shallow size and retained size?

Key terms:

- **Shallow size:** Memory occupied by the object itself.
- **Retained size:** Approximate memory that could become reclaimable if that object and the objects it exclusively keeps alive were removed.
- **Reference path:** The chain of references that keeps an object reachable.
- **GC root:** A starting point for reachability analysis.

Official guide: [Capture a heap dump](https://developer.android.com/studio/profile/capture-heap-dump)

### Tool 3: LeakCanary

LeakCanary is a development library that watches objects expected to be garbage collected (such as destroyed Activities and Fragments). If they remain retained, it can capture a heap dump and produce a leak trace.

Use its leak trace to find the reference path. Fix the owner or lifecycle problem; do not blindly add `WeakReference` everywhere.

Project: [LeakCanary on GitHub](https://github.com/square/leakcanary)

### Tool 4: Perfetto / system tracing

Use Perfetto or system tracing when you suspect GC activity contributes to jank. Inspect timing tracks and correlate GC events with frame timing and app work.

ART GC debugging guide: [Debug ART garbage collection](https://source.android.com/docs/core/runtime/gc-debug)

### Tool 5: ADB memory information

```bash
adb shell dumpsys meminfo com.example.myapp
```

This gives a process-level memory breakdown. It is useful alongside the Java heap profiler, but it is not a substitute for a heap dump when diagnosing object references.

### A practical leak investigation workflow

1. Reproduce the issue on a debug build and a representative device.
2. Record a baseline memory profile.
3. Open the screen suspected of leaking.
4. Navigate away so its lifecycle should finish.
5. Repeat the open/close cycle several times.
6. Request GC using the profiler for diagnostic purposes, then capture a heap dump.
7. Look for old Activity, Fragment, View, Adapter, or large model instances that should no longer exist.
8. Inspect their reference paths back to a GC root.
9. Fix the reference owner, callback, listener, coroutine, cache, or lifecycle handling.
10. Repeat the exact test and compare retained instances and memory trends.

One heap dump alone may not prove a leak. Look for objects that remain unexpectedly after their expected lifecycle and verify the result across repeated runs.

---

## 12. Practical optimization checklist

### Object lifecycle

- [ ] Do not store Activity, Fragment, or View references in long-lived singletons.
- [ ] Clear Fragment ViewBinding in `onDestroyView()`.
- [ ] Unregister listeners and callbacks when their owner is finished.
- [ ] Cancel lifecycle-bound coroutines and tasks when no longer needed.
- [ ] Keep caches bounded and provide a clear eviction strategy.

### Allocation behaviour

- [ ] Avoid unnecessary temporary objects in hot loops and per-frame code.
- [ ] Avoid loading huge collections when paging or streaming is suitable.
- [ ] Reuse expensive resources only when ownership and invalidation are clear.
- [ ] Avoid decoding full-size images for small UI thumbnails.
- [ ] Measure before applying micro-optimizations.

### Profiling

- [ ] Profile realistic user flows, not only app launch.
- [ ] Compare memory before and after repeated navigation.
- [ ] Inspect heap dumps and reference paths.
- [ ] Check native/graphics memory if the managed heap looks normal.
- [ ] Check GC timing and frame timing together when investigating jank.

---

## 13. Top 10 interview questions and answers

### Q1. What is Garbage Collection in Android?

**Answer:** Garbage Collection is the automatic memory-reclamation process performed by Android Runtime (ART) for managed objects that are no longer reachable by the application. ART identifies live objects through references and reclaims memory occupied by objects that cannot be reached. Developers should still manage object ownership and lifecycles correctly.

### Q2. How does Garbage Collection know whether an object is unused?

**Answer:** GC uses reachability analysis. It starts from GC roots, such as references associated with active threads, static fields, and runtime-managed references. It follows object references from those roots. Objects that cannot be reached are eligible for collection. An object is not garbage merely because the developer no longer wants it; if a live reference still points to it, it remains reachable.

### Q3. Does setting an object reference to `null` immediately free its memory?

**Answer:** No. Setting a reference to `null` removes that one reference. If no other strong or live reference can reach the object, it becomes eligible for GC. ART chooses when to run GC, so memory is not necessarily reclaimed immediately.

### Q4. What is the difference between a memory leak and Garbage Collection?

**Answer:** GC can reclaim objects that are unreachable. A memory leak occurs when an object that is no longer needed remains reachable through an unintended reference, such as a singleton retaining a destroyed Activity. GC cannot collect that object until the reference chain is removed.

### Q5. What is generational Garbage Collection, and why is it useful?

**Answer:** Generational GC takes advantage of the observation that many newly created objects become unused quickly. It can collect young objects more frequently and efficiently, while a full-heap collection handles a broader part of the heap when needed. ART's generational Concurrent Copying support is documented for Android 10 and later. It reduces some GC work but does not remove the cost of excessive allocations.

### Q6. What is Concurrent Copying GC in ART?

**Answer:** Concurrent Copying is an ART garbage collector designed to perform much of its work concurrently with application threads. It can copy live objects to compact the heap and uses read barriers to support concurrent operation. It still has short coordination pauses. It is the default ART GC plan documented from Android 8.0 onward, though device manufacturers can change runtime configuration.

### Q7. Can GC prevent an OutOfMemoryError?

**Answer:** Not always. GC can reclaim unreachable managed objects, but it cannot reclaim live objects that are still referenced. An app can still run out of memory if it retains too much data, requests a very large allocation, uses significant native/graphics memory, or exceeds runtime/system constraints. The solution is to find and fix the actual memory pressure—not to repeatedly request GC.

### Q8. Does calling `System.gc()` solve memory leaks?

**Answer:** No. `System.gc()` is only a request or hint to the runtime and does not guarantee immediate collection. It cannot collect an object that is still reachable. Calling it during normal app operation can also add unnecessary work. Use profiling tools to find retained objects and fix their references.

### Q9. How do you investigate a memory leak in an Android app?

**Answer:** I reproduce the suspected screen flow, take a baseline, open and close the screen repeatedly, and inspect memory with Android Studio Memory Profiler. I capture a heap dump after the screen should have been destroyed, identify unexpected retained instances, and inspect the reference path to a GC root. LeakCanary can help report the leak trace. After fixing the lifecycle or reference issue, I repeat the same test to verify that instances are no longer retained.

### Q10. Can frequent Garbage Collection cause UI jank?

**Answer:** It can contribute to jank. Excessive short-lived object allocation increases GC work, and some GC coordination phases can pause app threads. If those pauses or the associated CPU work interfere with a frame, the UI can miss its frame deadline. I would profile allocations and frame timing, identify hot paths, reduce unnecessary allocation, and verify the improvement rather than trying to disable GC.

---

## 14. Quick revision sheet

| Concept | Remember this |
|---|---|
| GC | Reclaims managed objects that are no longer reachable |
| Reachability | Whether an object can be reached from a GC root |
| GC root | A starting reference used by reachability analysis |
| Strong reference | Can keep an object alive |
| Weak reference | Does not by itself keep the target alive |
| Generational GC | Collects young objects efficiently because many die young |
| Concurrent GC | Performs much of its work while app threads continue, but short pauses can still occur |
| Compaction | Moves live objects to reduce fragmentation |
| Memory leak | Unneeded object remains reachable |
| OOM | An allocation cannot be satisfied under current memory constraints |
| Allocation churn | Many short-lived objects are created rapidly |
| Heap dump | Snapshot of managed objects and their references |
| Shallow size | Size of an object itself |
| Retained size | Memory kept alive exclusively through an object/reference relationship |
| Main lesson | Fix unnecessary references and allocations; do not depend on manual GC calls |

### One-minute interview summary

> Android uses ART to manage Kotlin and Java object memory. ART identifies live objects by tracing references from GC roots and can reclaim objects that are no longer reachable. Modern ART uses mostly concurrent collection techniques, and generational collection helps deal efficiently with short-lived objects. GC does not fix memory leaks because leaked objects are still reachable. Excessive allocation can increase GC work and contribute to UI jank. I use Android Studio Memory Profiler, heap dumps, LeakCanary, and Perfetto to investigate memory behaviour, then fix object ownership, lifecycle, and allocation problems.

---

## 15. Official references

1. [Overview of memory management – Android Developers](https://developer.android.com/topic/performance/memory-overview)
2. [Manage your app's memory – Android Developers](https://developer.android.com/topic/performance/memory/manage-app-memory)
3. [Debug ART garbage collection – Android Open Source Project](https://source.android.com/docs/core/runtime/gc-debug)
4. [Capture a heap dump – Android Studio](https://developer.android.com/studio/profile/capture-heap-dump)
5. [Memory Profiler – Android Studio](https://developer.android.com/studio/profile/memory-profiler)
6. [Fundamental memory concepts – Android Developers](https://developer.android.com/topic/performance/memory/guide/concepts)

> **Study tip:** First understand reachability and GC roots. Then learn the singleton/Activity leak example and Fragment ViewBinding lifecycle. Finally, practise the heap-dump workflow. These ideas help connect ART internals to real Android debugging and interview questions.
