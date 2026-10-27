# 📱 Android Memory Management — Quick Reference Notes

> **💡 The Golden Rule:** *"Free memory is wasted memory."*  
> Android keeps closed apps in RAM so they reopen instantly. Devices almost always run with very little free RAM, making smart memory management critical.

---

## 1. The 3 Types of Memory (The Desk Analogy)

```text
+-------------------------------------------------------------------+
| 1. RAM (Your Desk Top)                                            |
|    - Super fast, used by CPU & GPU, limited in space.             |
|                                                                   |
|    +---------------------------------------------------------+    |
|    | 2. zRAM (Vacuum-Seal Zip-Bag on the Corner of Desk)     |    |
|    |    - Partition inside RAM used for swap space.          |    |
|    |    - Compresses inactive data to save space;            |    |
|    |      decompresses when needed again.                    |    |
|    +---------------------------------------------------------+    |
+-------------------------------------------------------------------+
                                 ^
                                 | (Demand Paging: Reads files/code)
                                 v
+-------------------------------------------------------------------+
| 3. Storage (Your Filing Cabinet)                                  |
|    - Permanent data (apps, photos, files).                        |
|    - Android NEVER swaps temporary RAM to Storage because         |
|      constant writing wears out flash storage chips!              |
+-------------------------------------------------------------------+
```

---

## 2. Memory Pages (4KB Building Blocks)

RAM is divided into **4KB chunks** called **Pages**. Used pages fall into two categories:

| Page Type | Backed by Storage File? | Clean vs. Dirty | How Android Reclaims It |
| :--- | :--- | :--- | :--- |
| **Cached Pages** *(Code, mapped files)* | **Yes** *(Private or Shared)* | **Clean:** Unmodified copy of storage file.<br>**Dirty:** Modified copy in RAM. | **Clean:** Deleted anytime (re-read from disk via *Demand Paging*).<br>**Dirty:** Written to storage or compressed into **zRAM**. |
| **Anonymous Pages** *(App heap, variables)* | **No** *(Created from scratch)* | Always **Dirty** | Cannot be deleted (no disk backup). Must be compressed into **zRAM**. |

---

## 3. How Android Cleans Up Memory

### A. Inside Your App: Garbage Collection (ART)
The Android Runtime (ART) tracks object allocations across generational buckets. When a bucket hits its limit, **Garbage Collection (GC)** sweeps away unused objects:

```text
[New Objects] ──> [Young Generation] ──> [Older Generation] ──> [Permanent Generation]
```

### B. Across the OS: `kswapd` & `LMK`
When system-wide free RAM drops below threshold:

```text
                  [ Device Free RAM Runs Low! ]
                                |
                                v
       +-------------------------------------------------+
       | Step 1: kswapd (Kernel Swap Daemon)             |
       |  - Deletes "Clean" pages (can reload from disk) |
       |  - Compresses "Dirty" pages into zRAM           |
       +-------------------------------------------------+
                                |
           (Still below minimum free memory threshold?)
                                |
                                v
       +-------------------------------------------------+
       | Step 2: LMK (Low-Memory Killer)                 |
       |  - Force-closes processes to rescue the system, |
       |    starting with least important cached apps.   |
       +-------------------------------------------------+
```

---

## 4. Sharing & Measuring Memory (USS vs. PSS vs. RSS)

### How Android Shares Memory
* **Zygote:** Master process booted at startup with common framework code/themes. Every app is forked from Zygote to share framework RAM.
* **`mmap` / `ashmem` / `gralloc`:** Static files (`.odex`, `.so`, resources) are mapped via `mmap`. Dynamic graphics buffers use `ashmem` and `gralloc`.

### Memory Metrics Example
Suppose **App A** uses **10 MB** of private memory and shares a **6 MB** library with **2 other apps** (3 apps total):

```text
+------------------------------------+----------------------------------+
| App A Private Memory: 10 MB        | Shared Library (3 Apps): 6 MB    |
+------------------------------------+----------------------------------+
```

* **USS (Unique Set Size) = $10\text{ MB}$:** Only App A's private memory.
* **PSS (Proportional Set Size) = $12\text{ MB}$:** Private ($10\text{ MB}$) + fair share of shared ($\frac{6\text{ MB}}{3} = 2\text{ MB}$). *Best for OS total RAM accounting; slow to calculate.*
* **RSS (Resident Set Size) = $16\text{ MB}$:** Private ($10\text{ MB}$) + 100% of shared ($6\text{ MB}$). *Fastest to calculate.*

---

## 5. App Memory Budgets & Process States (Android 17 QPR2+)

Starting in **Android 17 QPR2 (API 37.2)**, apps can declare memory budgets in `AndroidManifest.xml` (release builds only) or dynamically via `MemoryBudgetManager` / NDK `<android/memory_budget_manager.h>`.

### What Happens When an App Exceeds Its Budget?
Instead of immediately killing the app, the OS trims memory in 3 steps:
1. **Evicts Clean file-backed pages** (inactive code/assets).
2. **Writes Dirty file-backed pages** to storage and evicts them.
3. **Compresses Anonymous pages** (heap data) into zRAM.

### The 4 Process States
```text
[1. FOREGROUND]  ──> App open on screen (e.g., 180 MB budget)
       │
       ▼ (User presses Home, audio keeps playing)
[2. PERCEPTIBLE] ──> No UI, but user hears audio / uses navigation (e.g., 64 MB budget)
       │
       ▼ (Playback stops, background ":sync" runs)
[3. BACKGROUND]  ──> Invisible work: sync jobs, alarms (e.g., 32 MB budget)
       │
       ▼ (All work finished, app idle)
[4. CACHED]      ──> Frozen in RAM for fast relaunch; prime target for LMK.
```

### Sample `AndroidManifest.xml` Configuration
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.radio">

    <application android:label="@string/app_name">
        <!-- Baseline budget (Phone/Tablet) -->
        <memory-budget android:maxMb="180" />

        <!-- Tighter budget when playing audio in background -->
        <memory-budget
            android:maxMb="64"
            android:state="perceptible" />

        <!-- Wear OS hardware override -->
        <memory-budget
            android:maxMb="48"
            android:feature="watch" />

        <processes>
            <!-- Dedicated background sync process -->
            <process android:process=":sync">
                <memory-budget android:maxMb="32" />
            </process>
        </processes>
    </application>
</manifest>
```
> **📌 Resolution Rule:** If multiple `<memory-budget>` tags match, the **last applicable clause** in the manifest takes effect.

---

## 6. Developer Best Practices Checklist

- [ ] **Enable R8 (`proguard-android-optimize.txt`):** Shrinks code, metadata, and RAM footprint.
  - *Avoid* `-dontoptimize`, `-dontshrink`, `-dontobfuscate`, and broad package wildcards (`com.example.** { *; }`).
- [ ] **Prevent Memory Leaks:** Don't hold references (e.g., `static` variables, singleton listeners) to `Activity`/`View` contexts after they close. Use lifecycle-aware components.
- [ ] **Avoid Memory Churn:** Never allocate new objects inside tight `for` loops or UI drawing methods (`onDraw`, Compose `DrawScope` / `Canvas`). It causes rapid GC pauses and dropped frames.
- [ ] **Optimize Bitmaps:** Downsample images to match the exact UI container size before loading into RAM.
- [ ] **Use Lite Protobufs:** Always use Lite Protocol Buffers on Android clients to avoid verbose generated code in RAM.
- [ ] **Use Hilt & Smart Containers:** Use Hilt for dependency injection, `SparseArray`/`ArrayMap` where appropriate, and use background `Services` sparingly.
- [ ] **Manage WebViews Carefully:** `WebView` runs in separate processes and consumes heavy RAM—destroy it when no longer needed.
- [ ] **Listen to Memory Pressure:** Check `ActivityManager.getMemoryInfo()` (`lowMemory`) before heavy work and release caches on `TRIM_MEMORY` callbacks.

---

## 7. Memory Debugging & Profiling Tools

| Stage | Tool | Purpose |
| :--- | :--- | :--- |
| **Local Debug** | **Android Studio Memory Profiler** | Real-time heap inspection, allocation tracking, and leak detection. |
| **Local Debug** | **Perfetto & Heap Dump Explorer** | System traces and deep visual analysis of Java/Kotlin/Native heap dumps. |
| **Local Debug** | **ADB (`dumpsys meminfo`)** | Run `adb shell dumpsys meminfo <package>` to inspect PSS, RSS, and active Memory Budgets. |
| **Local Build** | **R8 Configuration Analyzer** | Identifies overly broad keep rules blocking code/RAM optimization. |
| **Production** | **`ProfilingManager`** | Captures field memory/performance profiles on real user devices on trigger events. |
| **Production** | **Android Vitals (Play Console)** | Tracks real-world **Memory Usage (Anon RSS + swap)**, **Bitmap memory**, and **LMK kills**. |

---

## 🔗 Official Google Documentation Links
* [Android Memory Overview](https://developer.android.com/topic/performance/memory)
* [Overview of Memory Management](https://developer.android.com/topic/performance/memory-overview)
* [Memory Allocation Among Processes](https://developer.android.com/topic/performance/memory-management)
* [Manage Your App's Memory](https://developer.android.com/topic/performance/memory/manage-app-memory)
* [Set App Memory Budgets](https://developer.android.com/topic/performance/memory/app-memory-budgets)