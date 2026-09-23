---
type: component
title: "PDI Python Executor — Integration Test Flakiness (QUALITY-12061)"
description: "Analysis of intermittent Python Executor step failures in the PDI integration suites, covering two independent product defects that share one concurrent step-initialization window plus the test-side workaround currently in place."
tags: [architecture, pdi, python-executor, concurrency, flaky-test, timeout, quality-12061, pdi-20335]
timestamp: 2026-09-22T13:46:44Z
---

# PDI Python Executor — Integration Test Flakiness (QUALITY-12061)

<a id="summary"></a>
## 1. Summary

The PythonExecutor integration suites in `qa-automation` fail intermittently with a single, generic
assertion (see [§2 How the failure surfaces](#surfaces)). The failures are **not** a test-framework
defect.

Investigation found **two independent product defects** in the closed-source PDI Python Executor
plugin, plus one test-side workaround already in place:

| | Item | Nature | Status | Section |
|---|---|---|---|---|
| **Defect A** | Unsynchronized check-then-act in `com.pentaho.python.Python.installPyScript` | Product race condition | Fixed upstream as **PDI-20335**; ships on 11.1, **not backported** to 10.2 / 11.0 | [§5](#defect-a) |
| **Defect B** | `pdiPyServer` start-up budget hard-coded to 10 s with no override | Product configuration defect | **Unfixed.** Fix drafted and proven here; needs its own PDI ticket | [§6](#defect-b) |
| **Workaround** | Python warm-up transformation in the ACEA job | Test-asset change | Implemented and committed on the 10.2 QA branch | [§7](#workaround) |

Both defects are triggered by the same thing — a transformation initialising two or more
`PythonExecutor` steps at once — and both produce the *same* error message. That shared origin is
explained in [§4 Shared root context](#shared-context) and is the single most important thing to
understand about this ticket.

> ⚠️ **Triage before attributing.** At least five distinct root causes emit the identical assertion
> text. Always run the [§3 triage rule](#triage) first. A failure with no
> `FileAlreadyExistsException` in the Kitchen log is **not** Defect A, and a CI failure on an 11.1
> build is **not** evidence that PDI-20335 regressed.

**Where to go next:** for the product backport see [§5.3](#defect-a-fix); for the unfixed timeout see
[§6.3](#defect-b-fix); to reproduce anything here see [§8](#repro); for the current state of every
repository and the recommended actions see [§9](#state).

<a id="surfaces"></a>
## 2. How the failure surfaces

The suite
`tk421/suites/pythonexecutor-config-manual/User_Scenarios_ACEA-Ubuntu-config-manual.xml`
(executed through `com.hitachivantara.qa.automation.pdi.tests.PdiTest`) reports:

```
The following asserts failed:
  An error occurred when executing the transformation: Errors occurred during processing.
  expected [0] but found [1]
```

This message carries **no diagnostic information**. `PdiTest.executeTransformation` simply asserts
Kitchen's process exit code against the expected `0`; *any* transformation error produces
`1` and therefore this exact text. The real cause is only ever visible in the Kitchen log under
`pdi/target/output/logs/`.

Primary affected test: **ACEA Descriptive Statistics Ubuntu** (`testrail_case_id 9210065`), asset
`test_assets/User_Scenarios/ACEA/Acea_descriptive_stats/Acea_descriptive_stats_MainJob_Ubuntu.kjb`.
The sibling tests in the same suite (`ACEA Thresholds`, `ACEA Simplified Data`) contain no
`PythonExecutor` steps and are unaffected by either defect.

<a id="triage"></a>
## 3. Triage — one message, five root causes

> **Do not attribute a failure to QUALITY-12061 on the assertion text alone.**

Five distinct causes were observed producing the identical message during this investigation:

| # | Root cause | Discriminator in the Kitchen `.out` log | Which item |
|---|---|---|---|
| 1 | `installPyScript` race | `java.nio.file.FileAlreadyExistsException … pyCheck.py` | [Defect A](#defect-a) |
| 2 | `pdiPyServer` 10 s start-up budget exceeded under load | `pdiPyServer failed to initialize in time` | [Defect B](#defect-b) |
| 3 | matplotlib font-cache build on first run | same as #2, first run only | [env gap](#repro-gaps) |
| 4 | missing `scipy` | `py4j … null response`, `ModuleNotFoundError: scipy` | [env gap](#repro-gaps) |
| 5 | macOS `macosx` GUI backend | `NSWindow should only be instantiated on the main thread!` | [env gap](#repro-gaps) |

Causes #2 and #3 share a message but are different faults: #3 is a one-off cold-cache cost on the
first run in a new environment, #2 is a recurring load-sensitive cost. Both are cured by a larger
budget; only #3 is also cured by pre-building the font cache.

**Run this first, before any other analysis:**

```bash
cd /Users/andjorge/github/qa-automation
grep -l FileAlreadyExistsException      pdi/target/output/logs/*.out   # hits => Defect A
grep -l "failed to initialize in time"  pdi/target/output/logs/*.out   # hits => Defect B
```

<a id="shared-context"></a>
## 4. Shared root context — why Python Executor steps collide

Both defects live in the **same window**: PDI's concurrent step initialisation.

**1. PDI initialises every step of a transformation concurrently.**
`org.pentaho.di.trans.Trans.prepareExecution` creates one `StepInitThread` per step copy:

```java
StepInitThread[] initThreads = new StepInitThread[ steps.size() ];   // Trans.java:1201
...
initThreads[ i ] = new StepInitThread( sid, log );                   // Trans.java:1213
threads[ i ]     = new Thread( initThreads[ i ] );                   // Trans.java:1217
```

The array is sized from `steps.size()`, **not** from the hop topology. *Any* transformation with two
or more `PythonExecutor` steps is therefore exposed, even if those steps are chained sequentially.

**2. Each step installs shared script files.** `PythonExecutor.init` → `checkPythonRequirements` →
`Python.checkPyServer` → `Python.installPyScript( "pyCheck.py" )`, all racing on one static temp
directory → **[Defect A](#defect-a)**.

**3. Each step also spawns its own interpreter.** `PythonExecutor.init()` calls `initGateway()`
(`PythonExecutor.java:202`), which unconditionally constructs a **new** `PythonGateway` per step
instance (`PythonExecutor.java:388`) — no pooling, sharing or session reuse. Each gateway spawns its
own CPython process, and `pdiPyServer.py` imports `matplotlib`, `pandas` and `numpy` at **module
level**, i.e. before it reports its callback port → **[Defect B](#defect-b)**.

So a transformation with *N* `PythonExecutor` steps simultaneously performs *N* unsynchronized script
installs **and** *N* cold interpreter start-ups. Same trigger, same phase, same resulting message —
which is precisely why the two defects are so easily conflated.

<a id="exposed-assets"></a>
### 4.1 Exposed test assets

| Transformation | `PythonExecutor` steps |
|---|---|
| `Transformations/Scripting/Python_Executor/Scripting_IOVariations/Scripting_IOVariations.ktr` | 7 |
| `Transformations/Scripting/Python_Executor/Scripting_PredictPlotModel/Scripting_PredictPlotModel_trans1.ktr` | 5 |
| `User_Scenarios/ACEA/Acea_descriptive_stats/Acea_descriptive_stats_trans1_Ubuntu.ktr` | 4 |
| `User_Scenarios/ACEA/Acea_descriptive_stats/Acea_descriptive_stats_trans1_Windows.ktr` | 4 |

Only the two ACEA assets are covered by the [workaround](#workaround); the other two are not. Enumerate with:

```bash
cd /Users/andjorge/github/qa-automation/pdi/src/test/resources/test_assets
for f in $(grep -rl "PythonExecutor" --include=*.ktr .); do
  n=$(grep -c "<type>PythonExecutor</type>" "$f"); [ "$n" -ge 2 ] && echo "$n  $f"
done | sort -rn
```

In `Acea_descriptive_stats_trans1_Ubuntu.ktr` the four exposed steps are `Max_melt`, `Min_melt`,
`Mean_melt` and `STD_melt`, each on an independent parallel branch off `data` / `data 2`.

<a id="defect-a"></a>
## 5. Defect A — concurrent `installPyScript` race (PDI-20335)

<a id="defect-a-summary"></a>
### 5.1 Summary

| | |
|---|---|
| **What** | Unsynchronized check-then-act file copy in `com.pentaho.python.Python.installPyScript` |
| **Trigger** | Two or more `PythonExecutor` steps initialising concurrently ([§4](#shared-context)) |
| **Symptom** | `java.nio.file.FileAlreadyExistsException` on `pyCheck.py`, step init fails, Kitchen exits `1` |
| **Upstream** | **Already fixed** as PDI-20335, commit `525e44a7f4` (2026-09-10), `master` only |
| **Exposure** | 10.2.0.10 and 11.0.0.3 vulnerable; 11.1 fixed — matrix in [§5.3](#defect-a-fix) |
| **Action needed** | Backport to `10.2` and `11.0`. No new product ticket required |

<a id="defect-a-description"></a>
### 5.2 Description

Decompiled from `com/pentaho/python/Python.class` inside `python-executor-impl-10.2.0.10-438.jar`:

```java
static final File PY_SCRIPT_DIR;

static {
  PY_SCRIPT_DIR = com.google.common.io.Files.createTempDir();   // one shared dir per JVM
}

static File installPyScript( String name ) {
  ClassLoader cl = Python.class.getClassLoader();
  File f = new File( PY_SCRIPT_DIR, name );
  if ( f.exists() ) {                    // (1) CHECK
    return f;
  }
  URL url = cl.getResource( "py/" + name );
  Preconditions.checkNotNull( url );
  try {
    FileUtils.copyURLToFile( url, f );   // (2) ACT  <-- throws FileAlreadyExistsException
  } catch ( IOException e ) {
    throw new IllegalStateException(
      BaseMessages.getString( PKG, "PythonGateway.Error.FailedToCopyScriptFile", f ), e );
  }
  f.deleteOnExit();
  return f;
}
```

Two contributing facts:

**1. Unsynchronized check-then-act.** There is no lock between the `f.exists()` check (1) and the
`copyURLToFile` call (2). `PY_SCRIPT_DIR` is a single static temp directory shared by every Python
Executor step in the JVM.

**2. `FileUtils.copyURLToFile` is not concurrency-safe, even though it *does* request
`REPLACE_EXISTING`.** This is the counter-intuitive part. Bytecode of `commons-io-2.16.1` shows
`copyURLToFile(URL, File)` building a one-element `CopyOption[]` containing
`StandardCopyOption.REPLACE_EXISTING` and passing it to `PathUtils.copy`, which forwards to
`java.nio.file.Files.copy(InputStream, Path, CopyOption...)`:

```
27: anewarray     #259  // class java/nio/file/CopyOption     (array length 1)
32: getstatic     #261  // Field java/nio/file/StandardCopyOption.REPLACE_EXISTING
36: invokestatic  #414  // PathUtils.copy(IOSupplier, Path, CopyOption[])
```

The JDK implements that option **non-atomically**. From `java.base/java/nio/file/Files.java`
(Corretto 17.0.14, `src.zip`):

```java
if (replaceExisting) {
    try { deleteIfExists(target); } catch (SecurityException x) { se = x; }
}
...
ostream = newOutputStream(target, StandardOpenOption.CREATE_NEW,   // <-- line 3160
                                  StandardOpenOption.WRITE);
```

`deleteIfExists` and the `CREATE_NEW` open are two separate syscalls. If a competing thread creates
the file in between, the `CREATE_NEW` open fails with `FileAlreadyExistsException`. The observed
frame `java.nio.file.Files.copy(Files.java:3160)` is exactly this `newOutputStream(..., CREATE_NEW,
WRITE)` line, corroborating the mechanism precisely.

> **Consequence for any fix:** adding `REPLACE_EXISTING` is *not* a remedy — it is already there.
> Only a lock or an atomic publish will work. See [§5.3](#defect-a-fix).

The same pattern applies to the second installed resource, `py/pdiPyServer.py`.

#### 5.2.1 Observed evidence

Log: `pdi/target/output/logs/Acea_descriptive_stats_MainJob_Ubuntu.out`.

```
Min_melt.0 - ERROR : Unable to run Acea_descriptive_stats_trans1_Ubuntu.
             Failed to copy the Python script file /var/folders/.../T/13200552006143917003/pyCheck.py
Max_melt.0 - ERROR : Unable to run Acea_descriptive_stats_trans1_Ubuntu.
             Failed to copy the Python script file /var/folders/.../T/13200552006143917003/pyCheck.py

Caused by: java.nio.file.FileAlreadyExistsException: /var/folders/.../T/13200552006143917003/pyCheck.py
    at java.base/sun.nio.fs.UnixFileSystemProvider.newByteChannel(UnixFileSystemProvider.java:218)
    at java.base/java.nio.file.Files.copy(Files.java:3160)
    at org.apache.commons.io.file.PathUtils.copy(PathUtils.java:279)
    at org.apache.commons.io.FileUtils.copyURLToFile(FileUtils.java:1105)
    at com.pentaho.python.Python.installPyScript(Python.java:96)
    at com.pentaho.python.Python.checkPyServer(Python.java:64)
    at com.pentaho.di.trans.steps.pythonexecutor.PythonExecutor.checkPythonRequirements(PythonExecutor.java:260)
    at com.pentaho.di.trans.steps.pythonexecutor.PythonExecutor.init(PythonExecutor.java:184)
    at org.pentaho.di.trans.step.StepInitThread.run(StepInitThread.java:60)

Acea_descriptive_stats_trans1_Ubuntu - ERROR : Step [Max_melt.0] failed to initialize!
Acea_descriptive_stats_trans1_Ubuntu - ERROR : Step [Min_melt.0] failed to initialize!
Descriptive stats trans1 - ERROR : org.pentaho.di.core.exception.KettleException:
    We failed to initialize at least one step.  Execution can not begin!
        at org.pentaho.di.trans.Trans.prepareExecution(Trans.java:1298)
Kitchen - ERROR : Finished with errors
```

#### 5.2.2 Why it is intermittent

Each Kitchen invocation is a fresh JVM, so `PY_SCRIPT_DIR` is always empty at the start of the run —
the window is open on *every* execution. The outcome depends purely on thread scheduling:

* If the first thread completes `copyURLToFile` before the others evaluate `f.exists()`, all succeed.
* If two or more threads observe `f.exists() == false` before the winner finishes writing, the
  losers throw and the transformation aborts.

**Locally the window is not narrow at all.** On the reference machine the control experiment
reproduced the failure on **5 of 5** runs with **2** occurrences per run — effectively deterministic.
The intermittency reported in the ticket is therefore environment dependent: faster storage, fewer
cores, or a busy machine can serialize the threads enough to hide it. **Do not assume a passing run
means the defect is absent.**

#### 5.2.3 Production blast radius

* Under Kitchen (one JVM per run) `PY_SCRIPT_DIR` is empty at every execution, so the window reopens
  on every single run — which is why the QA suite is the most exposed consumer.
* On the Pentaho Server / Carte path one JVM is shared across concurrent transformations, so the
  window opens on the first Python step of the JVM's lifetime and then closes permanently. The blast
  radius is limited to a server's first concurrent Python executions after each restart — lower
  frequency, but a failed transformation at that moment is a real customer-visible error.

This severity question is now **moot for prioritisation**: the product team independently found and
fixed the defect as PDI-20335, so it is already accepted as real. The remaining work is a backport.

<a id="defect-a-fix"></a>
### 5.3 How to fix

> Proposed commit subject:
> `fix(python-executor): Backport of PDI-20335 prevent concurrent Python script installation [QUALITY-12061]`

**The fix already exists upstream.** Commit `525e44a7f4`, *"fix: prevent concurrent Python script
installation [PDI-20335] (#2606)"* (`pdi-plugins-ee`, 2026-09-10) marks the method
`static synchronized` and adds a concurrency unit test. It was never backported.

**Scope:** this class is **not** in `pentaho-kettle`. It ships from the closed plugin repository
`pdi-plugins-ee` (module `python-executor/impl`, group `com.pentaho.di.plugins`), source path
`python-executor/impl/src/main/java/com/pentaho/python/Python.java`.

#### 5.3.1 Per-version status

`installPyScript` was inspected directly in each **shipped** binary, not inferred from source:

```bash
unzip -p <install>/plugins/python-executor/python-executor-impl-*.jar \
  com/pentaho/python/Python.class > /tmp/Python.class
javap -p /tmp/Python.class | grep installPyScript
```

| PDI version | Plugin jar | `installPyScript` modifiers | Status |
|---|---|---|---|
| 10.2.0.10 | `python-executor-impl-10.2.0.10-438.jar` | `static` | **Vulnerable** |
| 11.0.0.3 | `python-executor-impl-11.0.0.3-312.jar` | `static` | **Vulnerable** |
| 11.1.0.0-SNAPSHOT | `python-executor-impl-11.1.0.0-SNAPSHOT.jar` | `static synchronized` | **Already fixed** |

Corresponding source branches:

| Branch | Contains `525e44a7f4` | `Python.java` relative to `10.2` |
|---|---|---|
| `10.2` | No | baseline |
| `11.0` | No | **byte-identical** to `10.2` |
| `master` (11.1) | Yes | differs only by the BSL header and the `synchronized` keyword |

#### 5.3.2 Backport plan

Because `10.2` and `11.0` carry byte-identical copies of `Python.java`, **the same commit applies
cleanly to both** — two cherry-picks, one per branch. Verified non-destructively:

```bash
cd ~/github/pdi-plugins-ee
git worktree add --detach /tmp/pw 10.2
cd /tmp/pw && git cherry-pick -n 525e44a7f4
```

*Result:* `Auto-merging` on both files, **no conflicts**, exit code 0. The 10.2 proprietary licence
header is preserved, `synchronized` is added at `Python.java:86`, and the upstream
`scriptsCanBeInstalledConcurrently` unit test comes along with it.

| Target | Action |
|---|---|
| `master` (11.1) | none — already contains `525e44a7f4` |
| `11.0` | `git cherry-pick 525e44a7f4` |
| `10.2` | `git cherry-pick 525e44a7f4` |

Cherry-picking the *same* commit rather than writing a per-branch fix keeps all three lines on
identical code.

#### 5.3.3 Why `synchronized` is sufficient

`PY_SCRIPT_DIR` is created by `com.google.common.io.Files.createTempDir()` in a static initialiser,
so it is unique per JVM and can never be contended by another process. The only competing callers
are threads inside the same JVM, and `installPyScript` is `static`, so `synchronized` locks on
`Python.class` and makes the check-then-act atomic for every caller. Both call sites —
`Python.checkPyServer` (`pyCheck.py`) and `PythonGateway` line 148 (`pdiPyServer.py`) — funnel
through this one method, so both are covered.

#### 5.3.4 Alternative remedies (for reference)

Any of these would also close the race; they are recorded because they were evaluated, not because
they are recommended over the upstream fix:

1. A static `ReentrantLock`, or a per-filename lock from a `ConcurrentHashMap<String, Object>`.
2. Atomic publish — copy to a unique temp file inside `PY_SCRIPT_DIR`, then `Files.move( tmp, target,
   ATOMIC_MOVE, REPLACE_EXISTING )`.
3. Memoize via `ConcurrentHashMap.computeIfAbsent( name, Python::doInstall )`.

Two traps, both analysed in [§5.6](#defect-a-review):

* ⚠️ Merely switching to a copy call that passes `REPLACE_EXISTING` is **not** a fix — see
  [§5.2](#defect-a-description).
* ⚠️ Do **not** simply catch `FileAlreadyExistsException` and return the existing `File`: the
  competing writer may have only just created the target and still be writing it, so the caller
  could receive a truncated script.

<a id="defect-a-poc"></a>
### 5.4 Local PoC implementation

The upstream fix was applied locally and injected into the installed PDI 10.2.0.10 plugin jar.

```bash
cd ~/github/pdi-plugins-ee
git cherry-pick -n 525e44a7f4          # onto branch QUALITY-12061; staged, NOT committed
cd python-executor && mvn -pl impl -o clean package
```

Module build: **70 tests, 0 failures**.

Deployment — the original filename and OSGi manifest are kept intact (both original and rebuilt
classes are Java 11, major version 55), with a pristine backup alongside:

```bash
cd python-executor/impl/target/classes
jar uf <install>/plugins/python-executor/python-executor-impl-10.2.0.10-438.jar \
       com/pentaho/python/Python.class
```

```
plugins/python-executor/python-executor-impl-10.2.0.10-438.jar                    <- patched
plugins/python-executor/python-executor-impl-10.2.0.10-438.jar.QUALITY-12061.bak  <- original
```

This install has no Karaf cache copy of the plugin (`find system/karaf/caches -iname '*python*'`
returns nothing), so replacing the jar is enough — no cache purge required.

**Regression test.** The cherry-pick brings the upstream `scriptsCanBeInstalledConcurrently` (10
threads released together from a `CountDownLatch`, asserting every caller receives the same `File`).
It currently ships on `master`/11.1 only and will accompany the proposed backports.

A locally written equivalent (`scriptInstallationIsThreadSafe`, 8 threads released from a
`CyclicBarrier`) was produced independently before the upstream commit was discovered, and proved
that such a test genuinely catches the defect. Against **unfixed** source:

```
[ERROR] scriptInstallationIsThreadSafe  <<< ERROR!
Caused by: java.nio.file.FileAlreadyExistsException: /var/folders/.../T/<random>/pyCheck.py
[ERROR] Tests run: 2, Failures: 0, Errors: 1
```

Against fixed source it passes. Since it arrived at the same design as upstream it is **redundant**,
and was discarded so the branches stay identical.

<a id="defect-a-evidence"></a>
### 5.5 Evidence

#### 5.5.1 PDI 10.2.0.10 — before / after

The [workaround](#workaround) was **removed** for these runs
(`git checkout 36ee93355 -- *_MainJob_*.kjb`) so the race was fully exposed and Defect A was measured
on its own. Two suites were used; `pythonexecutor_config_manual.xml` is the stronger evidence because
it needs **no test-asset changes at all** and exercises `Scripting_IOVariations.ktr` (7 steps):

| Suite | Arm | Runs | Result | Race hits |
|---|---|---|---|---|
| `User_Scenarios_ACEA-Ubuntu-config-manual` | control, pristine jar | 1 | `Tests run: 3, Failures: 1` | 2 |
| `User_Scenarios_ACEA-Ubuntu-config-manual` | PDI-20335 applied | 5 | 5 × `Tests run: 3, Failures: 0` | **0** |
| `pythonexecutor_config_manual` | control, pristine jar | 1 | `Tests run: 5, Failures: 2` | 4 |
| `pythonexecutor_config_manual` | PDI-20335 applied | 5 | 5 × `Tests run: 5, Failures: 0` | **0** |

Both suites were restored to a pristine jar (`cp *.bak`) before the control arm, so the comparison is
on identical assets with only the plugin class differing.

**10/10 runs green, 0 `FileAlreadyExistsException` across both suites.** Defect A's fix alone
therefore resolves the race, and the [workaround](#workaround) is not required once the product is
patched.

#### 5.5.2 PDI 11.1 — stress verification of the already-fixed baseline

11.1 ships `static synchronized` and needs no change. It was used as a living test bed to answer two
questions: does the fix hold under adversarial load, and is the harness actually *capable* of
detecting the defect if it were present?

Both suites were temporarily given `parallel="tests" thread-count="6"` and run against 12–24
busy-loop CPU load generators on a 12-core machine. These edits were reverted afterwards.

| Scenario | Result | Race hits |
|---|---|---|
| 11.1 baseline, both suites, no load | green | 0 |
| **Negative control** — `synchronized` deliberately removed, rebuilt, injected | both suites fail | **ACEA 2, python-executor 6** |
| 11.1 fixed jar restored, parallel + CPU load, ACEA ×4 | 3 pass / 1 fail | 0 |
| 11.1 fixed jar, parallel, no load, ACEA | green | 0 |

The **negative control** is the important row: it proves the amplified harness *does* catch the race
when the defect is present. Every run with the fix in place scored **0 race hits** — including the
failing one, whose cause was [Defect B](#defect-b). PDI-20335 holds.

<a id="defect-a-review"></a>
### 5.6 Design review — upstream `synchronized` vs. the local variant

Both variants were reviewed against each other. **Verdict: the upstream one-word fix is not a worse
solution — it is the better one.** Both fully close the race; the local variant over-solves it.

**Is `synchronized` sufficient?** Yes, with no hole — see [§5.3.3](#defect-a-fix). Even in the
pathological OSGi case where `Python` is loaded by two class loaders, each copy gets its own
`PY_SCRIPT_DIR`, so they still cannot collide. The theoretical concern about holding the
`Python.class` monitor across `classLoader.getResource()` in Karaf needs a reverse lock-ordering path
that does not exist in this code; the local variant does not avoid it either, since it calls
`getResource()` while a `ConcurrentHashMap` bin computation is active.

| Aspect | Upstream `synchronized` | Local `computeIfAbsent` + `ATOMIC_MOVE` |
|---|---|---|
| Closes the reported race | Yes | Yes |
| Verified on PDI 10.2.0.10 | 5/5 pass, 0 hits | 5/5 pass, 0 hits |
| Diff size | 1 keyword | ~30 lines, new static field |
| Divergence from `master` | none | permanent on 10.2 + 11.0 |
| `AtomicMoveNotSupportedException` risk | n/a | **yes** — `ATOMIC_MOVE` is not guaranteed by the `Files.move` contract, even within one directory, and the variant provides no fallback |
| Recovers if the installed file is later deleted | yes — re-installs | **no** — the map caches the `File` permanently and never re-checks `exists()` |
| Survives a mid-copy `IOException` | **no** — leaves a truncated file that later calls accept as valid | yes — staged publish never exposes a partial file |

Only one genuine advantage remains for the local variant: crash-safe publication. If `copyURLToFile`
fails part-way (disk full, source read error), JDK `Files.copy` runs
`try ( OutputStream out = ostream ) { in.transferTo( out ); }` with **no cleanup**, so a truncated
script is left on disk and the next call returns it because `exists()` is true. Technically correct
but low-materiality here: the scripts are a few KB, read from the plugin jar into a freshly created
temp directory, and a disk-full condition would be failing far more than this.

Against that, the local variant carries two concrete regressions the upstream fix does not
(`ATOMIC_MOVE` portability and permanent stale caching) plus three-branch divergence, for no measured
benefit.

**Decision: discard the local variant and cherry-pick `525e44a7f4`.** If crash-safe publication is
ever deemed a product requirement, the right move is to add the staged copy *inside* the upstream
`synchronized` method — the `ConcurrentHashMap` is unrelated to that goal — and to raise it upstream
on `master` first so all branches stay identical.

<a id="defect-b"></a>
## 6. Defect B — hard-coded `pdiPyServer` start-up budget

<a id="defect-b-summary"></a>
### 6.1 Summary

| | |
|---|---|
| **What** | `PythonGateway.MAX_PDI_PY_SERVER_INIT_WAIT` is a hard-coded 10 s with no override of any kind |
| **Trigger** | Several `PythonExecutor` steps starting interpreters at once on a loaded machine ([§4](#shared-context)) |
| **Symptom** | `pdiPyServer failed to initialize in time`, step init fails, Kitchen exits `1` — **identical assertion to [Defect A](#defect-a)** |
| **Measured** | Handshake is ~0.7 s normally, but **38–68 s** in ~1 loaded run in 5 |
| **Upstream** | **Not fixed.** Not addressed by PDI-20335; affects 11.1 too |
| **Action needed** | New PDI ticket. Fix drafted in [§6.4](#defect-b-poc); default must be raised as well |

This is the more plausible explanation for CI failures on an **already-fixed** build, because it is
purely load-sensitive and therefore worsens precisely when agents are busy.

<a id="defect-b-description"></a>
### 6.2 Description

`com.pentaho.python.PythonGateway`, before the change:

```java
private static final int MAX_PDI_PY_SERVER_INIT_WAIT = 10;          // line 54
…
if ( !latch.await( MAX_PDI_PY_SERVER_INIT_WAIT, TimeUnit.SECONDS ) ) {   // line 172
  destroyProcess( process );
  throw new IllegalStateException( getString( PKG, "PythonGateway.Error.FailedToInit", … ) );
}
```

The budget is a **hard-coded 10 seconds with no system-property or step-level override**.

Every step pays this cost independently: `initGateway()` builds a new gateway per step
(`PythonExecutor.java:202` → `:388`), each spawning its own CPython process whose `pdiPyServer.py`
imports `matplotlib`, `pandas` and `numpy` at module level before releasing the latch — see
[§4](#shared-context).

Observed failure, from `Acea_descriptive_stats_MainJob_Ubuntu.out`:

```
Max_melt.0  - java.lang.IllegalStateException: Python process started with
  [/usr/bin/python3, -u, …/pdiPyServer.py, 64759], but pdiPyServer failed to initialize in time.
Mean_melt.0 - … port 64758 … failed to initialize in time.
Min_melt.0  - … port 64760 … failed to initialize in time.
Descriptive stats trans1 - We failed to initialize at least one step.  Execution can not begin!
```

All three steps time out within the same millisecond, each having spawned its own interpreter. Full
timing data is in [§6.5](#defect-b-evidence).

<a id="defect-b-fix"></a>
### 6.3 How to fix

> Proposed commit subject:
> `fix(python-executor): make pdiPyServer init wait configurable [QUALITY-12061]`
>
> **Separate product defect** — needs its own PDI ticket. Documented here only because it
> masquerades as [Defect A](#defect-a).

Three changes, of which the first two **must ship together**:

**1. Make the budget configurable.** Follow the convention already used in this plugin
(`PythonExecutor` reads `Const.KETTLE_PYTHON_STEP_REPLACE_NULLS` via `EnvUtil.getSystemProperty`):

| Item | Value |
|---|---|
| Property | `KETTLE_PYTHON_SERVER_INIT_WAIT_SECONDS` |
| Default | `10` in the drafted change (deliberately unchanged — see point 2) |
| Set via | `kettle.properties`, or `-D` on the JVM |
| Invalid / non-positive | logged via `PythonGateway.Error.InvalidInitWait`, falls back to the default |

`EnvUtil.environmentInit()` → `applyKettleProperties()` copies `kettle.properties` entries into
system properties, so `kettle.properties` works without extra plumbing.

**2. Raise the default.** Configurability alone **cures nothing out of the box** — it only gives
operators a supported lever. A 0.7 s happy path against a measured 68 s worst case means 10 s has no
margin at all. The measurements here do **not** support 60 s either: at 60 s the harness still failed
about one loaded run in four, because the 68 s outlier exceeds it. Of the values actually tested,
only **300 s** has green evidence (19/19). A production default should sit above the observed tail
with explicit margin and deserves its own measurement on representative CI hardware, since the tail
is a property of the agent, not of PDI.

**3. Stop discarding Python output on failure.** On timeout, `destroyProcess` calls
`killPyIOThreads()`, which cancels the stdout/stderr reader futures, so any buffered Python output is
lost. It should be given a short grace period after `destroyForcibly` so the reader threads can
flush; otherwise genuine Python tracebacks are invisible — this cost real time in this very
investigation ([§6.6](#defect-b-diagnosis)).

**Worth considering beyond the timeout:** whether *N* concurrent interpreter start-ups is desirable
at all. Pooling or serialising gateway start-up would remove the contention rather than widen the
timeout around it.

<a id="defect-b-poc"></a>
### 6.4 Local PoC implementation

Implemented in `pdi-plugins-ee`, branch `master`, **uncommitted**. Files touched:
`PythonGateway.java`, `messages_en_US.properties`, `messages_zh_TW.properties`,
`PythonGatewayTest.java` (3 new tests: honoured value, unparseable, non-positive).

The constant became a default plus a resolver:

```java
private static final int DEFAULT_PDI_PY_SERVER_INIT_WAIT = 10;

@VisibleForTesting static final String PDI_PY_SERVER_INIT_WAIT_PROPERTY =
  "KETTLE_PYTHON_SERVER_INIT_WAIT_SECONDS";

@VisibleForTesting int resolveInitWaitSeconds() {
  String configured = EnvUtil.getSystemProperty( PDI_PY_SERVER_INIT_WAIT_PROPERTY );
  if ( isNullOrEmpty( configured ) ) {
    return DEFAULT_PDI_PY_SERVER_INIT_WAIT;
  }
  try {
    int seconds = Integer.parseInt( configured.trim() );
    if ( seconds <= 0 ) { /* log InvalidInitWait */ return DEFAULT_PDI_PY_SERVER_INIT_WAIT; }
    return seconds;
  } catch ( NumberFormatException e ) { /* log InvalidInitWait */ return DEFAULT_PDI_PY_SERVER_INIT_WAIT; }
}
```

The timeout message now reports the budget actually used and names the property, turning a dead-end
error into an actionable one:

```
… but pdiPyServer failed to initialize in time (waited 60 seconds). If this happens under load
or with several Python steps, increase KETTLE_PYTHON_SERVER_INIT_WAIT_SECONDS in kettle.properties.
```

Build and deploy:

```bash
cd ~/github/pdi-plugins-ee/python-executor && mvn -pl impl -o clean package
cd impl/target/classes
jar uf <install>/plugins/python-executor/python-executor-impl-11.1.0.0-SNAPSHOT.jar \
       com/pentaho/python/PythonGateway.class \
       com/pentaho/python/messages/messages_en_US.properties \
       com/pentaho/python/messages/messages_zh_TW.properties
```

Note the drafted change deliberately keeps the default at `10`, so it is only half of the remedy —
see point 2 of [§6.3](#defect-b-fix).

<a id="defect-b-evidence"></a>
### 6.5 Evidence

**Correctness**

- Module build green: **121 tests, 0 failures**; `PythonGatewayTest` 6/6.
- The two pre-existing tests that stub `latch.await( 10, SECONDS )` pass unchanged, confirming the
  default is preserved.
- End-to-end on PDI 11.1 with `KETTLE_PYTHON_SERVER_INIT_WAIT_SECONDS=60` in `kettle.properties`:
  the Kitchen log emitted **`waited 60 seconds`**, proving the property reaches the Kitchen JVM.

**Efficacy under load** (`parallel="tests" thread-count="6"`, 24 busy-loop generators on 12 cores):

| Budget | Loaded ACEA result |
|---|---|
| 10 s (current default) | ~1 in 4 runs fails |
| 60 s | ~1 in 4 runs fails — absorbs the 38 s cluster but not the 68 s outlier |
| 300 s | **19 / 19 runs green** |

**Handshake timing distribution.** Instrumenting `PythonGateway` to log the real latch wait on every
start, across six consecutive loaded runs with the budget raised to 300 s so nothing was killed:

| Run | Handshake durations (ms) |
|---|---|
| 1 | **38441, 38444, 38539, 68446** |
| 2 | 679, 736, 737, 737 |
| 3 | 690, 753, 754, 754 |
| 4 | 693, 693, 694, 701 |
| 5 | 726, 726, 726, 773 |
| 6 | 730, 731, 731, 731 |

The distribution is sharply **bimodal**: ~0.7 s normally even under full load, degrading to 38–68 s
in roughly one loaded run in five — matching the observed failure rate. The three steps that start
together cluster tightly (38.44 / 38.44 / 38.54 s), the signature of shared contention rather than a
hang. Several simultaneous cold imports of `matplotlib`/`pandas`/`numpy` on an oversubscribed machine
are simply very slow.

All diagnostic instrumentation used to obtain these numbers has been removed from the source.

<a id="defect-b-diagnosis"></a>
### 6.6 Diagnosis trail — and one retracted conclusion

An earlier revision of this analysis concluded that a **third, undiagnosed defect** sat behind the
loaded failures, on the grounds that raising the budget to 60 s did not stop them and the Python
process emitted nothing. **That conclusion was wrong and is retracted.** It is recorded here because
the two measurement faults behind it are easy to repeat:

1. *Lost diagnostics.* On timeout, `destroyProcess` calls `killPyIOThreads()`, which cancels the
   stdout/stderr reader futures. Any buffered Python output is discarded, so "no Python output" was
   an artefact of the teardown, not evidence of a silent process. (This is now recommendation 3 in
   [§6.3](#defect-b-fix).)
2. *A broken sampler.* The polling script used `ps -o etimes=`, which does not exist in the BSD `ps`
   shipped with macOS. It errored on every iteration and sampled **nothing** — including healthy
   long-lived servers. Its "no slow processes found" result was meaningless.

Direct instrumentation then explained every earlier observation exactly:

- **10 s budget** — even the fast path has no margin, and a 38 s episode fails all steps.
- **60 s budget** — absorbs the 38 s cluster but not the 68 s outlier, so a single test still fails.
  This is why raising it to 60 s appeared to change nothing.
- **300 s budget** — 19 of 19 loaded runs green, including the run whose steps took 38–68 s.

The processes were never hung: the diagnostic build confirmed them `isAlive=true` at timeout with
`exitValue=137`, i.e. killed by PDI's own `destroyForcibly`, not crashed.

**Conclusion.** The loaded failures are the `pdiPyServer` start-up budget and nothing else.

<a id="workaround"></a>
## 7. Workaround — test-side Python warm-up

<a id="workaround-summary"></a>
### 7.1 Summary

| | |
|---|---|
| **What** | A single-step Python warm-up transformation inserted before the multi-step one |
| **Fixes** | [Defect A](#defect-a) only, and only for the two ACEA assets |
| **Status** | Implemented and committed on `qa-automation` branch `QUALITY-12061` as `43a1b13c1` |
| **Verified** | 5/5 pass with the fix; 5/5 fail with 10 race hits without it |
| **Cost** | ~1.4 s per job execution |
| **Recommendation** | Retire once [Defect A](#defect-a) is backported — see [§9.2](#state-actions) |

<a id="workaround-description"></a>
### 7.2 Description

> Commit subject used:
> `test(pdi): add Python warm-up to ACEA descriptive stats job [QUALITY-12061]`

Guarantee that `pyCheck.py` / `pdiPyServer.py` already exist before the four parallel steps
initialise, by warming up the Python session with a *single* Python Executor step first. Because each
Kitchen run is a separate JVM but the warm-up runs in the **same** JVM as `trans1`, `PY_SCRIPT_DIR`
is already populated and `installPyScript` short-circuits at the `f.exists()` check.

*Pros:* no product change required, retains full test coverage, deterministic.
*Cons:* hides the defect from this particular test; the product bug must still be fixed, otherwise
real customers with multi-Python-step transformations remain exposed. It also covers only the two
ACEA assets — not `Scripting_IOVariations.ktr` (7 steps) or `Scripting_PredictPlotModel_trans1.ktr`
(5 steps), see [§4.1](#exposed-assets).

**Alternative form of the same workaround:** split the four melt branches into separate
transformations so no two `PythonExecutor` steps ever initialise concurrently. Considerably more
invasive and changes what the test actually exercises.

**Rejected alternative — retry logic.** Commit subject would have been
`test(pdi): retry PythonExecutor suites on step-init failure [QUALITY-12061]`: add a TestNG
`IRetryAnalyzer` or an internal retry loop in `PdiTest.executeTransformation`. Rejected — it masks a
genuine product defect, slows the suite, and would suppress unrelated real regressions. Acceptable
only as an explicitly time-boxed stop-gap.

<a id="workaround-apply"></a>
### 7.3 How to apply

Insert a warm-up transformation as a new job entry between `Delete file` and
`Descriptive stats trans1` in:

* `test_assets/User_Scenarios/ACEA/Acea_descriptive_stats/Acea_descriptive_stats_MainJob_Ubuntu.kjb`
* `test_assets/User_Scenarios/ACEA/Acea_descriptive_stats/Acea_descriptive_stats_MainJob_Windows.kjb`

Resulting flow:

```
Start → Set variables → Row Normaliser test → Delete file
      → Python warm-up → Descriptive stats trans1 → Descriptive Stats trans2 → Success
```

<a id="workaround-poc"></a>
### 7.4 Local PoC implementation

**Files added**

| File | Purpose |
|---|---|
| `.../ACEA/Acea_descriptive_stats/Acea_descriptive_stats_python_warmup_Ubuntu.ktr` | Warm-up transformation, `EMBED_ENV_PATH=${python_path_ubuntu}` |
| `.../ACEA/Acea_descriptive_stats/Acea_descriptive_stats_python_warmup_Windows.ktr` | Warm-up transformation, `EMBED_ENV_PATH=${python_path_win}` |

Each contains exactly three steps — `Generate row` (`RowGenerator`, 1 row, one `String` field
`warmup`) → `Python warm-up` (`PythonExecutor`, `IS_ALL_ROWS=true`, script `df = df`) →
`Dummy (do nothing)`. A single `PythonExecutor` step means there is no second thread to race with, so
`installPyScript` runs uncontended and populates `PY_SCRIPT_DIR` for the rest of the JVM's lifetime.

The header, `<log>` tables, slave server definitions and trailing `<attributes>` blocks were copied
verbatim from the corresponding `Acea_descriptive_stats_trans1_*.ktr` so the files match existing
asset conventions exactly (including the namespace-unaware `xsi:type` usage that PDI relies on).

**Files modified** — both `Acea_descriptive_stats_MainJob_*.kjb` (+48 lines each, no deletions):

1. New `TRANS` entry `Python warm-up` pointing at the matching warm-up `.ktr` via
   `${Internal.Entry.Current.Directory}`, with `run_configuration=Pentaho local` and
   `wait_until_finished=Y` — identical settings to the neighbouring transformation entries.
2. The hop `Delete file → Descriptive stats trans1` replaced by
   `Delete file → Python warm-up` and `Python warm-up → Descriptive stats trans1`.

No existing step, transformation, hop or expected result was altered, so test coverage is unchanged.

<a id="workaround-evidence"></a>
### 7.5 Evidence

Run on macOS 26.7 (build 25G229) / Apple silicon (arm64), Corretto 17.0.14, PDI 10.2.0.10.

| Configuration | Runs | Suite result | Race hits |
|---|---|---|---|
| **With** warm-up (fix applied) | 5 | 5 × `Tests run: 3, Failures: 0` | **0** |
| **Without** warm-up (control, `git checkout` of both `.kjb`) | 5 | 5 × `Tests run: 3, Failures: 1` | **10** (2 per run) |

The control arm re-reproduced the exact reported assertion on every iteration, and the fixed arm
eliminated it entirely — establishing causality rather than mere correlation.

Post-fix log evidence (`Acea_descriptive_stats_MainJob_Ubuntu.out`):

```
Starting entry [Python warm-up]
Generate row.0     - Finished processing (I=0, O=0, R=0, W=1, U=0, E=0)
Python warm-up.0   - Finished processing (I=0, O=0, R=1, W=1, U=0, E=0)
...
Max_melt.0  - Finished processing (I=0, O=0, R=75, W=4425, U=0, E=0)
Mean_melt.0 - Finished processing (I=0, O=0, R=75, W=4425, U=0, E=0)
Min_melt.0  - Finished processing (I=0, O=0, R=75, W=4500, U=0, E=0)
```

<a id="repro"></a>
## 8. Reproduction environment

<a id="repro-prereq"></a>
### 8.1 Pre-requisites

#### 8.1.1 Required

| # | Requirement | Expected value / check |
|---|---|---|
| 1 | PDI installation | `10.2.0.10` at `/Users/andjorge/pentaho/pdi/10.2.0.10/design-tools/data-integration` |
| 2 | `PENTAHO_HOME` env var | `/Users/andjorge/pentaho/pdi/10.2.0.10` — `PdiTest.preValidation` throws `PentahoHomeNotValidException` if unset or not a directory |
| 3 | `qa-automation` checkout | `/Users/andjorge/github/qa-automation` |
| 4 | JDK + Maven | JDK 17 (verified: `Thread.java:840` in the stack trace matches Corretto 17.0.14); `mvn` on `PATH`. Kitchen resolves its JVM via `set-pentaho-env.sh` → `PENTAHO_JAVA_HOME`, else `JAVA_HOME` |
| 5 | Python Executor plugin | `$PENTAHO_HOME/design-tools/data-integration/plugins/python-executor/python-executor-impl-10.2.0.10-438.jar` present |
| 6 | Python interpreter | The four `PythonExecutor` steps set `EMBED_USE_ENV=true` / `EMBED_ENV_PATH=${python_path_ubuntu}`. That variable is loaded by the job's `Set variables` entry (`SET_VARIABLES`, scope `JVM`) from `test_assets/User_Scenarios/ACEA/testProp.properties` (`python_path_ubuntu=/usr/bin/python3`, `python_path_win=C:\Users\devuser\Python38\python.exe`) |
| 7 | Python libraries | `pandas`, `numpy`, `matplotlib`, `py4j` installed **in the interpreter from row 6**. `py4j` must match the plugin's Java-side `plugins/python-executor/lib/py4j-0.10.7.jar` |

#### 8.1.2 Required only for the two non-Python tests in the suite

| # | Requirement | Expected value / check |
|---|---|---|
| 8 | PostgreSQL | Reachable on `localhost:5432` |
| 9 | Database | `regression_db` exists, owned by / accessible to user `postgres` (values from `testProp.properties`: `hostname`, `database`, `port`, `username`) |

`ACEA Thresholds` and `ACEA Simplified Data` need rows 8–9. The defects under investigation are
reproduced by `ACEA Descriptive Statistics Ubuntu` alone, which needs rows 1–7 only.

<a id="repro-status"></a>
#### 8.1.3 Verified local status

Checked on 2026-09-22 against `/Users/andjorge/pentaho/pdi/10.2.0.10`:

| # | Requirement | Status | Observed |
|---|---|---|---|
| 1 | PDI 10.2.0.10 install | MET | present |
| 2 | `PENTAHO_HOME` | MET | via `~/.zshrc` (interactive shells only — see [§8.3](#repro-gaps)) |
| 3 | `qa-automation` checkout | MET | present |
| 4 | JDK + Maven | MET | Amazon Corretto 17.0.14 via `JAVA_HOME`; Apache Maven 3.9.16 |
| 5 | Python Executor plugin jar | MET | `python-executor-impl-10.2.0.10-438.jar` |
| 6 | Python interpreter | MET | `/usr/bin/python3` → Python 3.9.6 |
| 7 | Python libraries | MET (remediated) | `pandas 2.3.3`, `numpy 2.0.2`, `matplotlib 3.9.4`, `py4j 0.10.7` installed into the user site |
| 8 | PostgreSQL `localhost:5432` | MET | accepting connections |
| 9 | `regression_db` | MET | database exists |

<a id="repro-steps"></a>
### 8.2 Reproduction steps

Each step is idempotent and independently verifiable.

**Step 1 — Verify the PDI installation and plugin jar.**

```bash
export PENTAHO_HOME=/Users/andjorge/pentaho/pdi/10.2.0.10
test -d "$PENTAHO_HOME/design-tools/data-integration" && echo "OK: PDI present"
ls "$PENTAHO_HOME/design-tools/data-integration/plugins/python-executor/"
```

*Expected:* `OK: PDI present` and a listing containing `python-executor-impl-10.2.0.10-438.jar`.

**Step 2 — Verify the Python interpreter satisfies the plugin's library check.**

```bash
/usr/bin/python3 -c "import pandas, numpy, matplotlib, py4j; print('OK: libraries present')"
```

*Expected:* `OK: libraries present`.
*If this fails*, stop — you will hit a gap from [§8.3](#repro-gaps) instead of the defect.

**Step 3 — Clear previous run artefacts so the log is unambiguous.**

```bash
rm -rf /Users/andjorge/github/qa-automation/pdi/target/output/logs
```

**Step 4 — Execute the target suite.**

```bash
cd /Users/andjorge/github/qa-automation
PENTAHO_HOME=/Users/andjorge/pentaho/pdi/10.2.0.10 \
mvn -pl pdi -DrunITs -Dit.test=PdiTest verify \
  -DpdiXmlFile=tk421/suites/pythonexecutor-config-manual/User_Scenarios_ACEA-Ubuntu-config-manual.xml
```

For the other suite, substitute `pythonexecutor_config_manual.xml`. For other PDI versions, set
`PENTAHO_HOME` to `…/11.0.0.3` or `…/11.1.0.0-SNAPSHOT/20260916`.

*Expected on a failing run:* the `ACEA Descriptive Statistics Ubuntu` test reports the assertion in
[§2](#surfaces).

**Step 5 — Identify which defect fired.** Apply the [§3 triage rule](#triage):

```bash
grep -E "FileAlreadyExistsException|installPyScript|failed to initialize" \
  pdi/target/output/logs/Acea_descriptive_stats_MainJob_Ubuntu.out
```

*Defect A signature:*

```
Caused by: java.nio.file.FileAlreadyExistsException: /.../T/<random>/pyCheck.py
	at com.pentaho.python.Python.installPyScript(Python.java:96)
ERROR : Step [Max_melt.0] failed to initialize!
```

*Defect B signature:* `pdiPyServer failed to initialize in time`.

**Step 6 — Measure the Defect A reproduction rate.**

> The [workaround](#workaround) is committed on the 10.2 QA branch. To reproduce Defect A you must
> first revert it: `git checkout -- '*Acea_descriptive_stats_MainJob_*.kjb'`, run the loop, then
> restore. Alternatively use `pythonexecutor_config_manual.xml`, which needs no asset changes.

```bash
cd /Users/andjorge/github/qa-automation
hits=0
for i in $(seq 1 5); do
  rm -rf pdi/target/output/logs
  PENTAHO_HOME=/Users/andjorge/pentaho/pdi/10.2.0.10 \
  mvn -q -pl pdi -DrunITs -Dit.test=PdiTest verify \
    -DpdiXmlFile=tk421/suites/pythonexecutor-config-manual/User_Scenarios_ACEA-Ubuntu-config-manual.xml \
    > /dev/null 2>&1
  n=$(grep -c "FileAlreadyExistsException" \
        pdi/target/output/logs/Acea_descriptive_stats_MainJob_Ubuntu.out 2>/dev/null || echo 0)
  echo "iteration $i: $n FileAlreadyExistsException occurrence(s)"
  [ "$n" -gt 0 ] && hits=$((hits+1))
done
echo "reproduced in $hits/5 runs"
```

*Expected (unfixed assets, reference machine):* `2` occurrences on every iteration and
`reproduced in 5/5 runs`. Any value greater than `0/5` confirms the defect. A `0/5` result does
**not** clear the product — it only means this environment serializes the step-init threads; verify
with Step 8 instead.

**Step 7 — Reproduce Defect B.** It needs CPU pressure. Amplify the harness by adding
`parallel="tests" thread-count="6"` to the `<suite>` element and running against busy-loop
generators:

```bash
for c in $(seq 1 24); do ( end=$((SECONDS+1400)); while [ $SECONDS -lt $end ]; do :; done ) & done
```

*Expected:* roughly 1 run in 4 fails with `failed to initialize in time` and **zero** race hits.
Remember to revert the suite XML afterwards.

**Step 8 — (Optional) Inspect the defective method directly.**

```bash
mkdir -p /tmp/pyexec && cd /tmp/pyexec
unzip -o -q "$PENTAHO_HOME/design-tools/data-integration/plugins/python-executor/python-executor-impl-10.2.0.10-438.jar" \
  'com/pentaho/python/*'
javap -p -c com/pentaho/python/Python.class | awk '/File installPyScript/,/^$/'
```

*Expected on a vulnerable build:* `File.exists` at offset `19`, followed unguarded by
`FileUtils.copyURLToFile` at offset `45`, with no monitor/lock instructions between them.

<a id="repro-gaps"></a>
### 8.3 Environment gaps that mask the defects (macOS)

These are **setup gaps, not defects**. If present they cause deterministic failures that hide the
intermittent ones, and several emit the same assertion text — see [§3](#triage):

* `regression_db` missing → `FATAL: database "regression_db" does not exist` on
  `Creat Table` / `Creat Table1` job entries.
  *Remedy:* create the database on `localhost:5432` as user `postgres`.
* `/usr/bin/python3` on macOS ships without the data-science stack → `Mean_melt` logs
  `The Python installation is missing required libraries.` followed by
  `Library 'pandas' is missing`, etc.
  *Remedy (no sudo, no repo change — installs into the per-user site directory that
  `/usr/bin/python3` already reads):*

  ```bash
  /usr/bin/python3 -m pip install --user pandas numpy matplotlib py4j==0.10.7
  ```

* `scipy` missing → `Scripting_ImageOutput` dies with
  `py4j.Py4JNetworkException: Error while sending a command: null response`. `pandas`'
  `scatter_matrix( …, diagonal='kde' )` needs it. Note the plugin pre-flight check (`pyCheck.py`)
  only validates the libraries configured on the step, so `scipy` passes the check and then fails at
  runtime.
  *Remedy:* `/usr/bin/python3 -m pip install --user scipy`
* `scikit-learn` missing → only affects `Scripting_PredictPlotModel`, which is commented out in the
  suite XML. Installing it (`scikit-learn 1.6.1`) is necessary but **not sufficient** to re-enable
  that test — see [§8.4](#repro-disabled).
  *Remedy:* `/usr/bin/python3 -m pip install --user scikit-learn`
* matplotlib **font cache not yet built** → the one-time build costs ~11.8 s and, on a first run in
  a fresh environment, alone exceeds the 10 s `pdiPyServer` budget ([Defect B](#defect-b)):

  ```
  Python warm-up.0 - Matplotlib is building the font cache; this may take a moment.
  java.lang.IllegalStateException: … but pdiPyServer failed to initialize in time.
  	at com.pentaho.python.PythonGateway.startPdiPyServer(PythonGateway.java:177)
  ```

  *Remedy:* pre-build it during provisioning; import time then drops to 0.4 s.

  ```bash
  /usr/bin/python3 -c "import matplotlib; matplotlib.use('Agg'); import matplotlib.font_manager as fm; fm.fontManager.ttflist"
  ```

* matplotlib defaulting to the **`macosx` GUI backend** → the Python process is killed outright with
  `NSInternalInconsistencyException … 'NSWindow should only be instantiated on the main thread!'`,
  surfacing on the Java side as the same `null response` py4j error. Linux CI agents default to the
  headless `Agg` backend, so this is macOS-specific.
  *Remedy:* create `~/.matplotlib/matplotlibrc` containing `backend: Agg` (or export `MPLBACKEND=Agg`).
  Verify with `python3 -c "import matplotlib; print(matplotlib.get_backend())"`.
* `PENTAHO_HOME` exported only from `~/.zshrc` → visible to **interactive** shells only.
  Non-interactive shells (CI steps, `zsh -l -c`, agent tool sessions, some IDE run configurations)
  do **not** source `~/.zshrc` and will trip `PdiTest.preValidation`.
  *Remedy:* export it inline on the command, or move the export to `~/.zshenv`.

<a id="repro-disabled"></a>
### 8.4 Disabled test and a historical workaround

* **`Scripting_PredictPlotModel` (5 `PythonExecutor` steps) is commented out** in
  `pythonexecutor_config_manual.xml`, so only 5 of the 6 declared tests run. Re-enabling it to gain
  more race pressure failed **3/3 with 0 race hits** for two stacked environment reasons: first
  `ModuleNotFoundError: No module named 'sklearn'` (remedied with `scikit-learn 1.6.1`), then a
  deterministic `TypeError: unsupported operand type(s) for -: 'str' and 'str'` raised inside
  `df.quantile()` — a pandas/numpy version incompatibility in the test script, not a concurrency
  problem. Left commented out.
* **Commit `f3cd45e4d`** *"[QUALITY-10128] removing parallel execution for PythonExecutor tests"*
  (2022) removed `parallel="tests" thread-count="3"` from **both** PythonExecutor suites. Whether
  that was an earlier workaround for these defects is **unconfirmed** — parallel TestNG tests spawn
  separate Kitchen JVMs, each with its own `PY_SCRIPT_DIR`, so it was more likely `pdiPyServer` port
  or CPU contention, i.e. closer to [Defect B](#defect-b). Note the **Windows** ACEA suite still
  carries `parallel="tests" thread-count="3"`.

<a id="repro-artefacts"></a>
### 8.5 Relevant artefacts

| Artefact | Path |
|---|---|
| Kitchen log (failing job) | `pdi/target/output/logs/Acea_descriptive_stats_MainJob_Ubuntu.out` |
| Suite definitions | `pdi/src/test/resources/tk421/suites/pythonexecutor-config-manual/` |
| Failing transformation | `pdi/src/test/resources/test_assets/User_Scenarios/ACEA/Acea_descriptive_stats/Acea_descriptive_stats_trans1_Ubuntu.ktr` |
| Test driver | `pdi/src/main/java/com/hitachivantara/qa/automation/pdi/tests/PdiTest.java` |
| Process launcher | `pdi/src/main/java/com/hitachivantara/qa/automation/pdi/executor/PdiExecutor.java` |
| Defect A source | `pdi-plugins-ee/python-executor/impl/src/main/java/com/pentaho/python/Python.java` |
| Defect B source | `pdi-plugins-ee/python-executor/impl/src/main/java/com/pentaho/python/PythonGateway.java` |
| Per-step gateway creation | `pdi-plugins-ee/.../pythonexecutor/PythonExecutor.java` lines 202, 388 |
| Python server bootstrap | `pdi-plugins-ee/python-executor/impl/src/main/resources/py/pdiPyServer.py` |
| Defective plugin jar | `$PENTAHO_HOME/design-tools/data-integration/plugins/python-executor/python-executor-impl-10.2.0.10-438.jar` |
| `commons-io` on the copy path | `$PENTAHO_HOME/design-tools/data-integration/lib/commons-io-2.16.1.jar` (all 8 copies bundled are 2.16.1) |
| JDK source for the non-atomic copy | `$JAVA_HOME/lib/src.zip` → `java.base/java/nio/file/Files.java`, line 3160 |

<a id="state"></a>
## 9. Current state and recommended next actions

<a id="state-where"></a>
### 9.1 What is where

This investigation committed nothing. The one pre-existing commit below (`43a1b13c1`, the
[workaround](#workaround)) was made before it began.

| Repo / install | Branch | State |
|---|---|---|
| `pentaho-kettle` | `QUALITY-12061` | This document only, staged `AM`. |
| `qa-automation` | `QUALITY-12061` (10.2 line) | Workaround committed as `43a1b13c1` (warm-up `.ktr` + patched `.kjb`). |
| `qa-automation` | `QUALITY-12061-111` (11.1 line) | **Clean.** All stress edits reverted. |
| `pdi-plugins-ee` | `master` | Four modified files — the [Defect B](#defect-b-poc) fix. Already contains PDI-20335. |
| PDI 10.2.0.10 install | — | Jar patched with PDI-20335; pristine backup `*.QUALITY-12061.bak`. |
| PDI 11.1 install | — | Jar rebuilt from `master` + the Defect B fix; backup `*.QUALITY-12061.bak`. |
| PDI 11.0.0.3 install | — | **Untouched — still vulnerable to Defect A.** |

Local environment changes made while reproducing (all cumulative, none reverted): `pandas 2.3.3`,
`numpy 2.0.2`, `matplotlib 3.9.4`, `py4j 0.10.7`, `scipy 1.13.1`, `scikit-learn 1.6.1` installed into
the `/usr/bin/python3` user site; matplotlib font cache pre-built; `~/.matplotlib/matplotlibrc`
created containing `backend: Agg`.

<a id="state-actions"></a>
### 9.2 Recommended actions

**[Defect A](#defect-a) — `installPyScript` race (PDI-20335)**

1. Backport `525e44a7f4` to branches `10.2` and `11.0` — two cherry-picks of the same commit, one per
   branch. Verified to auto-merge with no conflicts ([§5.3.2](#defect-a-fix)).
2. Once shipped, consider retiring the [workaround](#workaround) — it becomes redundant, and it only
   ever covered the ACEA assets, not the other two exposed transformations ([§4.1](#exposed-assets)).

**[Defect B](#defect-b) — `pdiPyServer` start-up budget**

3. Raise a product ticket; attach [§6](#defect-b) and the measurements in [§6.5](#defect-b-evidence).
4. Land the configurability change **and raise the default** — the two must ship together, see
   [§6.3](#defect-b-fix).
5. Give `killPyIOThreads()` a short grace period after `destroyForcibly` so Python tracebacks are not
   discarded on failure.
6. Consider whether *N* concurrent interpreter start-ups is desirable at all; pooling or serialising
   start-up would remove the contention rather than widen the timeout around it.

**Test suites**

7. Do not re-enable `parallel="tests"` on the PythonExecutor suites; review the Windows ACEA suite,
   which still carries it ([§8.4](#repro-disabled)).
8. `Scripting_PredictPlotModel` stays disabled until the pandas/numpy incompatibility is resolved.

<a id="state-why"></a>
### 9.3 Answering the original question — why is CI flaky?

Two mechanisms, and CI can be hitting either:

- On **10.2 / 11.0**, [Defect A](#defect-a) is live and fires whenever a transformation initialises
  two or more `PythonExecutor` steps.
- On **any** version including 11.1, [Defect B](#defect-b) fires whenever the agent is loaded enough
  to push a cold interpreter start-up past 10 s — which is exactly when CI is busiest.

Because both produce the same assertion text, a CI failure on an 11.1 build is **not** evidence that
PDI-20335 regressed. Apply the [§3 triage rule](#triage) before reopening this ticket.

<a id="changelog"></a>
## 10. Changelog

| Date (UTC) | Change |
|---|---|
| 2026-09-22T13:46:44Z | Initial analysis. Root cause identified as an unsynchronized check-then-act in `com.pentaho.python.Python.installPyScript` racing across four concurrent `StepInitThread`s in `Acea_descriptive_stats_trans1_Ubuntu.ktr`. Documented options A (product fix), B (warm-up workaround) and C (retry, rejected). No code changes made. |
| 2026-09-22T15:17:08Z | Restructured section 5 from free-form environment notes into `5.1 Pre-requisites` (required / DB-only / known masking gaps), `5.2 Reproduction steps` (7 numbered, independently verifiable steps with commands and expected output, including a loop to force the timing-dependent race) and `5.3 Relevant artefacts`. No code changes made. |
| 2026-09-22T15:21:30Z | Verified all nine pre-requisites against the local machine. Corrected two errors in 5.1: row 4 claimed JDK 11, actual runtime is Corretto 17.0.14 (confirmed by reproducing `Thread.java:840`); row 6 now documents the `EMBED_USE_ENV`/`EMBED_ENV_PATH=${python_path_ubuntu}` resolution path and row 7 the `py4j==0.10.7` version constraint imposed by `plugins/python-executor/lib/py4j-0.10.7.jar`. Added remedies to 5.1.3 and a new 5.1.4 verified-status table. Remediated gap #7 by installing `pandas 2.3.3`, `numpy 2.0.2`, `matplotlib 3.9.4`, `py4j 0.10.7` into the `/usr/bin/python3` user site. Gap #2 resolved by the user via `~/.zshrc`; noted the interactive-shell-only caveat. No code changes made. |
| 2026-09-22T15:34:36Z | Implemented Option B. Added `Acea_descriptive_stats_python_warmup_Ubuntu.ktr` and `Acea_descriptive_stats_python_warmup_Windows.ktr` (3 steps each, single `PythonExecutor`) and wired a `Python warm-up` TRANS entry between `Delete file` and `Descriptive stats trans1` in both `Acea_descriptive_stats_MainJob_*.kjb` (+48 lines each, no deletions). Verified 5/5 suite passes with 0 race hits versus a 5/5-fail control with 10 race hits. Documented in new section 6; changelog moved to section 7 (later renumbered). Also recorded an incidental first-run-only failure mode: matplotlib's 11.8 s font-cache build exceeds the `pdiPyServer` startup timeout. Added an Option A open question on whether concurrent `PythonExecutor` init occurs in production. Nothing committed. |
| 2026-09-22T15:52:00Z | Review pass over the whole document plus a code review of the `qa-automation` working-tree changes (no defects found in the changed `.ktr`/`.kjb` assets). Corrected four documentation errors. **(a)** Section 3 previously claimed `FileUtils.copyURLToFile` passes **no** `REPLACE_EXISTING`; bytecode of `commons-io-2.16.1` proves it **does**. The real mechanism is that the JDK implements `Files.copy(InputStream, Path, REPLACE_EXISTING)` as a non-atomic `deleteIfExists` + `newOutputStream(CREATE_NEW)` pair — `Files.java:3160` in Corretto 17.0.14 `src.zip` matches the observed stack frame exactly. Added the consequence that adding `REPLACE_EXISTING` is not a valid remedy, and cross-referenced it from Option A item 2. **(b)** Section 3 implied the parallel-branch topology causes the concurrency; `Trans.java:1201` sizes the init-thread array from `steps.size()`, so any transformation with two or more `PythonExecutor` steps is exposed regardless of hop layout. Added a table of the two further exposed assets (`Scripting_IOVariations.ktr`, 7 steps; `Scripting_PredictPlotModel_trans1.ktr`, 5 steps) that the Option B fix does not cover. **(c)** Section 6.3 said macOS 15; actual is macOS 26.7 (build 25G229), arm64. **(d)** Reconciled the "intermittent" framing with the measured 5/5 local control failures, rewrote 5.2 Step 6 from "loop until it fires" into a 5-iteration rate measurement, and added the missing instruction to revert the Option B fix before attempting reproduction. Nothing committed. |
| 2026-09-22T16:57:05Z | Rewrote the three section-4 option headings as ready-to-use commit subjects following the repository convention `type(scope): subject [TICKET]` (as seen in `test(pdi): Scrub local filename from test [QUALITY-12058]`). Each is 75 characters. The descriptive qualifier and recommendation status that previously lived in the heading were moved to a blockquote directly beneath it, so no information was lost. Option A additionally notes that it lands in the closed `python-executor-impl` repository and therefore needs the PDI product ticket appended as `[QUALITY-12061][PDI-XXXXX]` once raised. Nothing committed. |
| 2026-09-22T19:30:00Z | Implemented and verified Option A against the PDI 10.2.0.10 install, and extended the analysis to all three maintained versions (new section 7; changelog renumbered to 8). **Key finding: the defect is already fixed upstream.** Commit `525e44a7f4` — *"fix: prevent concurrent Python script installation [PDI-20335] (#2606)"*, `pdi-plugins-ee`, 2026-09-10 — makes `installPyScript` `synchronized` and adds a concurrency unit test, but exists on `master` only. Inspecting the shipped binaries with `javap` confirms 10.2.0.10 (`-438`) and 11.0.0.3 (`-312`) declare `static File installPyScript` and are vulnerable, while 11.1.0.0-SNAPSHOT declares `static synchronized` and needs nothing. `Python.java` is byte-identical on branches `10.2` and `11.0`, and differs on `master` only by the BSL header plus the `synchronized` keyword, so **one cherry-pick of `525e44a7f4` covers both maintenance branches**; validated with `git cherry-pick -n` in a throw-away detached worktree (auto-merge, no conflicts, 10.2 license header preserved). Verification on 10.2.0.10 was done with Option B **removed** so the race was exposed: control `static` = 5/5 fail with 10 race hits; upstream `synchronized` = 5/5 pass with 0 hits; a locally written `computeIfAbsent` + `ATOMIC_MOVE` variant = 5/5 pass with 0 hits. A new `scriptInstallationIsThreadSafe` unit test reproduces `FileAlreadyExistsException` against unfixed source and passes against fixed source. Resolved the former "Open question for Option A" — moot, since the product team already accepted and fixed the defect. Patched jar installed with a `.QUALITY-12061.bak` pristine backup; no Karaf cache copy exists, so no cache purge is needed. `qa-automation` restored to its committed Option B state. Nothing committed in any repository. |
| 2026-09-22T20:05:00Z | Code-reviewed the upstream PDI-20335 fix against the locally written variant and recorded the outcome in new section 7.6. Conclusion: upstream is **not** a worse solution — it is better. `synchronized` is provably sufficient because `installPyScript` is `static` (monitor = `Python.class`) and `PY_SCRIPT_DIR` is JVM-unique, so no cross-process contention is possible; the theoretical Karaf classloader lock-ordering concern has no reverse path and is not avoided by the local variant either. The local variant was found to carry two concrete regressions that upstream does not: `Files.move( …, ATOMIC_MOVE )` is not guaranteed by contract even within a single directory and has no fallback, and the `ConcurrentHashMap` memoization caches the `File` permanently so a later-deleted script is never reinstalled. Its single real advantage — crash-safe publication, since JDK `Files.copy` performs `in.transferTo( out )` with no cleanup and therefore leaves a truncated file that subsequent `exists()` checks accept — is technically valid but low-materiality for a few-KB resource in a fresh temp directory, and if ever required should be implemented as a staged copy *inside* the synchronized method and raised on `master` first. Decision recorded: discard the local variant, cherry-pick `525e44a7f4`. Nothing committed. |
| 2026-09-22T20:55:00Z | Applied the upstream cherry-pick and re-verified on PDI 10.2.0.10. Discarded the local variant, ran `git cherry-pick -n 525e44a7f4` onto `pdi-plugins-ee` branch `QUALITY-12061` (staged, **not committed**): auto-merged with no conflicts, diff is exactly the `synchronized` keyword plus the upstream `scriptsCanBeInstalledConcurrently` test, 10.2 proprietary licence header untouched. Module build green at **70 tests, 0 failures**. Deployed by restoring the pristine jar from `.QUALITY-12061.bak`, confirming it still declared plain `static`, then injecting the freshly built class. Verification used **two** suites with Option B removed so the race was exposed; `pythonexecutor_config_manual.xml` is the stronger evidence since it requires no test-asset changes at all and exercises `Scripting_IOVariations.ktr` (7 `PythonExecutor` steps). Control on the pristine jar: ACEA `Failures: 1` / 2 race hits, python-executor suite `Failures: 2` / 4 race hits. With PDI-20335 applied: **10/10 runs green across both suites, 0 race hits**. Two unrelated local environment defects were uncovered and fixed while verifying, both surfacing as the same `py4j … null response` error and neither connected to this ticket: missing `scipy` (needed by `scatter_matrix( diagonal='kde' )`, and invisible to the plugin's own `pyCheck.py` pre-flight), and matplotlib defaulting to the `macosx` GUI backend, which hard-kills the Python process with `NSWindow should only be instantiated on the main thread!`; both added to section 5.1.3. Also recorded that `Scripting_PredictPlotModel` (5 `PythonExecutor` steps) is commented out in the suite XML so only 5 of 6 tests run, and that commit `f3cd45e4d` [QUALITY-10128] removed `parallel="tests"` from both PythonExecutor suites in 2022 — flagged as possibly-related but **unconfirmed**, since parallel TestNG tests use separate Kitchen JVMs and therefore separate `PY_SCRIPT_DIR`s. `qa-automation` restored to its committed Option B state. Nothing committed in any repository. |
| 2026-09-22T22:20:00Z | Stress-verified PDI-20335 on PDI 11.1 and identified a **second, unrelated flakiness source** that produces the same assertion text (new sections 7.7–7.10). Both suites were temporarily amplified with `parallel="tests" thread-count="6"` and run against 12–24 CPU load generators; these edits have since been reverted and `qa-automation` is clean. A **negative control** — `synchronized` deliberately stripped from 11.1, rebuilt and injected — reproduced the race with 2 (ACEA) and 6 (python-executor) `FileAlreadyExistsException` hits, proving the amplified harness genuinely detects the defect. With the fix restored, **every** run scored **0 race hits**, including a 1-of-4 loaded ACEA failure. That failure's real cause was `pdiPyServer failed to initialize in time`, thrown simultaneously by three `PythonExecutor` steps: `PythonGateway.MAX_PDI_PY_SERVER_INIT_WAIT` is a **hard-coded 10 s with no override** (line 54, awaited at line 172), and each step spawns its own CPython process that must import pandas/numpy/matplotlib before counting the latch down — a budget that three contending interpreter start-ups exceed on a loaded agent. This is a distinct defect, not addressed by PDI-20335, and is the more plausible explanation for CI failures on already-fixed builds since it is purely load-sensitive. Added section 7.9 cataloguing **five** distinct root causes that all surface as `expected [0] but found [1]`, with the triage rule `grep FileAlreadyExistsException pdi/target/output/logs/*.out` — no hits means it is not QUALITY-12061. Also recorded (7.10) that `Scripting_PredictPlotModel` cannot be re-enabled: after installing `scikit-learn 1.6.1` it still fails deterministically with `TypeError: unsupported operand type(s) for -: 'str' and 'str'` inside `df.quantile()`, a pandas/numpy incompatibility unrelated to concurrency. Nothing committed in any repository. |
| 2026-09-22T23:40:00Z | Drafted the companion product fix for the `pdiPyServer` start-up timeout identified in 7.8 (new section 8; changelog renumbered to 9). `PythonGateway.MAX_PDI_PY_SERVER_INIT_WAIT` was a hard-coded `10` with no override of any kind; it is now `DEFAULT_PDI_PY_SERVER_INIT_WAIT = 10` plus a `KETTLE_PYTHON_SERVER_INIT_WAIT_SECONDS` property read through `EnvUtil.getSystemProperty`, matching the convention already used by `PythonExecutor` for `Const.KETTLE_PYTHON_STEP_REPLACE_NULLS`. Unparseable or non-positive values are logged via a new `PythonGateway.Error.InvalidInitWait` message and fall back to the default, so the out-of-the-box behaviour is unchanged. The `FailedToInit` message now reports the budget actually used and names the property, in both `en_US` and `zh_TW`. Three tests added to `PythonGatewayTest`; module build green at **121 tests, 0 failures**, with the two pre-existing tests that stub `latch.await( 10, SECONDS )` passing untouched. Verified end-to-end on PDI 11.1: with `KETTLE_PYTHON_SERVER_INIT_WAIT_SECONDS=60` in `kettle.properties` the Kitchen log emitted `waited 60 seconds`, proving `EnvUtil.environmentInit` → `applyKettleProperties` carries the value into the Kitchen JVM. **[SUPERSEDED — retracted by the 2026-09-23 entry and section 8.3; the two claims below rest on faulty measurements.]** **However (section 8.3), raising the budget did not fix the loaded failure**: ACEA still failed 1-in-4 loaded runs at 60 s, the same rate as at 10 s, with zero Python output of any kind. A 1 s budget with no load produced no failures at all, showing the latch is normally released in well under a second — it is counted down when the script reports its port, *before* the heavy imports — and `ProcessGovernorImpl` is constructed with a limit of 2000, so governor throttling is not involved either. A child that emits nothing for 60 s is hung or never scheduled, so a **third, still-undiagnosed defect** sits behind the loaded ACEA failures; it is neither QUALITY-12061 (0 race hits in every run) nor a too-small timeout. Suggested next step recorded: capture Python-side state during a loaded failure via `py-spy dump` or by raising py4j logging above `OFF` in `setPy4jLogging`. `qa-automation` and `~/.kettle/kettle.properties` restored to their original state. Nothing committed in any repository. |
| 2026-09-23T13:25:00Z | Diagnosed the "third defect" postulated in the previous entry and **retracted it — there is no third defect** (section 8.3 rewritten). The loaded ACEA failures are entirely explained by the `pdiPyServer` start-up budget. Two measurement faults had produced the wrong conclusion: (a) on timeout `destroyProcess` calls `killPyIOThreads()`, which cancels the stdout/stderr reader futures and **discards any buffered Python output**, so "the process emitted nothing" was an artefact of teardown rather than evidence of a silent child; and (b) the polling script used `ps -o etimes=`, which does not exist in the BSD `ps` shipped with macOS — it errored on every iteration and sampled nothing at all, including healthy long-lived servers, so its "no slow processes" result was meaningless. A temporary diagnostic build first established the children were `isAlive=true` at timeout with `exitValue=137` (killed by PDI's own `destroyForcibly`, not crashed), then a second build logged the real latch wait on every start. The distribution is sharply **bimodal**: across six loaded runs (24 busy-loop generators on 12 cores, `parallel="tests" thread-count="6"`, budget raised to 300 s) five runs handshook in **679–773 ms**, while one run took **38441, 38444, 38539 and 68446 ms**. The tight clustering of the three concurrently-started steps is the signature of shared contention, not a hang: `pdiPyServer.py` imports `matplotlib`, `pandas` and `numpy` at **module level**, before it reports its port, so several simultaneous cold imports on an oversubscribed machine are simply very slow. This explains every earlier observation — 10 s fails outright, 60 s absorbs the 38 s cluster but not the 68 s outlier (hence the persistent 1-in-4 failure), and 300 s gave **19 of 19 loaded runs green**. Two further recommendations added for the product ticket: raise the default above 10 s **[the 60 s figure originally suggested here was withdrawn by the 2026-09-23T15:29Z entry — 60 s still fails ~1 loaded run in 4]**, and give `killPyIOThreads()` a short grace period after `destroyForcibly` so genuine Python tracebacks are not swallowed. All diagnostic instrumentation has been removed; the plugin module rebuilds green at **121 tests, 0 failures** and the clean jar is deployed to the 11.1 install. `qa-automation` and `~/.kettle/kettle.properties` restored; only the four section-8 source files remain modified. Nothing committed in any repository. |
| 2026-09-23T15:29:00Z | Consolidated every finding from the investigation into the document. **Section 1** rewritten to present the ticket as **two independent product defects** rather than one: defect **A**, the `installPyScript` race (fixed upstream as PDI-20335, shipped on 11.1, not backported to 10.2/11.0), and defect **B**, the hard-coded 10 s `pdiPyServer` start-up budget (unfixed, fix drafted in section 8), with a warning to triage before attributing. Frontmatter `description` and `tags` widened accordingly. **New structural insight added to 1 and 7.8:** both defects occupy the *same* window — `PythonExecutor.init()` calls `initGateway()` at `PythonExecutor.java:202`, which constructs a **new** `PythonGateway`, and therefore a new CPython process, per step instance at `PythonExecutor.java:388` with no pooling or session reuse, while PDI runs `init()` on concurrent `StepInitThread`s; a transformation with *N* `PythonExecutor` steps thus performs *N* unsynchronised script installs **and** *N* cold interpreter start-ups simultaneously, which is precisely why the two defects are so easily conflated. Section 7.8 upgraded from hypothesis to measured fact and now records that `pdiPyServer.py` imports `matplotlib`, `pandas` and `numpy` at module level, before it reports its callback port. Section 7.9 clarified that causes #2 and #3 are different faults sharing one message (a recurring load cost versus a one-off cold-cache cost) and the triage rule expanded into two concrete `grep` commands. Section 8.2 extended with the efficacy table — 10 s and 60 s budgets each fail ~1 loaded run in 4, 300 s gives **19/19 green**. Section 5.1.3 gained two further environment gaps with remedies: missing `scikit-learn`, and an unbuilt matplotlib font cache. **New section 9** added covering the exact state of all three repositories and both patched installs (nothing committed anywhere), the cumulative local environment changes, prioritised next actions for each defect and for the test suites, and a direct answer to the original question of why CI is flaky — namely that a failure on an 11.1 build is not evidence that PDI-20335 regressed, because defect B produces an identical message and worsens exactly when agents are busy. Changelog renumbered to section 10. Nothing committed in any repository. |
| 2026-09-23T15:50:00Z | Independent review pass over the whole document; six inconsistencies corrected. **(a)** Section 6 said Option B was "not committed" while 9.1 recorded it as commit `43a1b13c1`, and both the 9.1 heading and the previous changelog entry asserted that nothing is committed in any repository. Clarified throughout: Option B *is* committed on `qa-automation` branch `QUALITY-12061`, by work predating this investigation, which itself committed nothing. **(b)** Option A in section 4 offered "catch `FileAlreadyExistsException` and return the existing `File`" as an alternative remedy; this is unsafe, because the competing writer may have only just created the target and still be writing it, so the caller can receive a truncated script — the very partial-publication hazard analysed in 7.6. Replaced with an explicit warning against it. **(c)** Section 7.5 claimed the upstream `scriptsCanBeInstalledConcurrently` regression test "now ships on all three branches"; it ships on `master`/11.1 only and will accompany the proposed backports. **(d)** Section 8.3 called the configurability change "the correct and sufficient remedy" while section 8.1 deliberately preserves the 10 s default, so as drafted it cures nothing out of the box. Reworded to state that configurability and a raised default must ship together. **(e)** The recommended 60 s default contradicted this document's own efficacy data — at 60 s the harness still failed about one loaded run in four, since the 68 s outlier exceeds it. Recommendation withdrawn in sections 8.3 and 9.2 and marked as withdrawn in the superseded changelog entry; of the values tested only 300 s has green evidence (19/19), and the production default should be chosen from a measurement on representative CI hardware, above the observed tail with margin. **(f)** "One cherry-pick covers both" was technically wrong — a cherry-pick applies to the checked-out branch only; reworded to two cherry-picks of the same commit, one per maintenance branch. Nothing committed in any repository. |
| 2026-09-23T16:35:00Z | Restructured the whole document around the three remedies. Reframed it as an analysis of PythonExecutor step flakiness rather than a list of options: former Option A became section 5 (Defect A), the former section 7.8 + section 8 timeout work became section 6 (Defect B), former Option B became section 7 (Workaround), and the rejected retry option was folded into 7.2. Each of the three now has the same subsection shape (summary, description, how to fix, local PoC implementation, evidence). Promoted the five-root-causes triage table to section 3 and the shared concurrent step-initialization context to section 4, since both apply to all three items. Added HTML anchors and cross-reference links throughout. No technical content was removed or altered. |
| 2026-09-23T19:30:00Z | Added [Annex A](#annex-a), a copy-and-paste-ready Jira ticket draft (title plus a non-technical summary/description) for [Defect B](#defect-b), the hard-coded `pdiPyServer` start-up budget. Implementation detail was deliberately left out so it can be decided by whoever picks the ticket up. |

<a id="annex-a"></a>
## Annex A — Jira ticket draft for Defect B

Ready to copy and paste into a new PDI ticket. Supporting technical detail lives in
[§6 Defect B](#defect-b); the measurements behind the numbers quoted below are in
[§6.5](#defect-b-evidence).

### A.1 Title

```
Python Executor: make the hard-coded 10s start-up wait configurable
```

68 characters, so it fits a 72-column commit subject line unchanged.

### A.2 Summary / description

```
Summary
-------
When a transformation runs several Python Executor steps at the same time, the steps can fail
to start on a busy machine. The step gives the Python side a fixed 10 seconds to become ready,
and that budget cannot be changed by the user in any way. It should be configurable.

What happens
------------
Every Python Executor step starts its own Python process and then waits for that process to
report back that it is ready. If the process has not reported back within 10 seconds, PDI kills
it and the step fails to initialise, which aborts the whole transformation with:

  Python process started with [...pdiPyServer.py, <port>], but pdiPyServer failed to
  initialize in time.

The 10 second budget is a constant in the plugin source
(com.pentaho.python.PythonGateway, MAX_PDI_PY_SERVER_INIT_WAIT, line 54). There is no
system property, kettle.properties entry, step option or environment variable that can change
it, so a user who hits this has no supported way to work around it.

Why 10 seconds is not enough
----------------------------
Starting the Python side is not instant: the Python process has to load several large scientific
libraries (matplotlib, pandas, numpy) before it can report that it is ready. On an idle machine
that takes well under a second, so the 10 second budget looks generous.

The problem is that a transformation with N Python Executor steps starts N independent Python
processes at the same moment, and they all load those libraries simultaneously. On a machine
that is already busy - a shared CI agent, a server running other jobs, a developer machine
under load - those simultaneous start-ups compete for CPU and disk, and the start-up time does
not degrade gently. It jumps.

Measurements taken on a 12-core machine under load (see the linked analysis for the full data):

  - normal case: about 0.7 seconds
  - roughly one loaded run in five: 38 to 68 seconds

So the failure is not a hang and it is not a crash. The Python processes were confirmed alive
and healthy at the moment PDI killed them - they were simply still loading. The 10 second budget
leaves no margin whatsoever between a 0.7 second happy path and a measured 68 second worst case.

Impact
------
  - Intermittent, hard-to-diagnose transformation failures that appear only under load, which is
    exactly when a production or CI system is busiest.
  - The error message is a dead end: it does not tell the user what the budget was, that a budget
    exists, or that there is anything they could adjust.
  - No supported remedy exists today. The only ways out are to reduce the number of Python
    Executor steps running concurrently, or to make sure the machine is idle.
  - This affects all currently supported versions, including 11.1. It is unrelated to PDI-20335.

What is being asked for
-----------------------
  1. Make the start-up wait configurable by the user, following whatever convention the plugin
     and PDI already use for tunable settings, so that administrators can raise it on slow or
     heavily loaded systems. Implementation approach is left to the assignee.

  2. Raise the default. Making the value configurable does not fix anything on its own, since
     out of the box everyone still gets 10 seconds. The default needs to sit comfortably above
     the worst case observed on realistic hardware. For reference, in testing a 60 second budget
     still failed about one loaded run in four, while 300 seconds passed 19 runs out of 19. A
     production default should be chosen from a measurement on representative hardware, because
     the slow tail is a property of the machine rather than of PDI.

  3. Improve the error message so it states the budget that was used and names the setting to
     change. Today the message gives the user nowhere to go.

  4. Stop discarding the Python output when the start-up fails. On timeout the plugin cancels the
     threads reading the Python process output, so anything Python had printed - including a real
     traceback - is thrown away. Allowing those readers a brief moment to flush before teardown
     would make genuine Python errors visible instead of invisible.

  5. Worth considering separately: each Python Executor step creates its own Python process, with
     no pooling or reuse. Sharing or staggering those start-ups would remove the contention
     altogether rather than just widening the timeout around it.

Notes
-----
  - A working proof of concept covering points 1 and 3 was built and verified locally during the
    QUALITY-12061 investigation; it is available if useful, but the final approach should be
    decided by whoever picks this up.
  - Important for triage: this failure produces the same generic assertion text in the QA
    integration suites as a separate, already-fixed race condition (PDI-20335). The two are told
    apart by the log line - "failed to initialize in time" for this issue, versus
    "FileAlreadyExistsException" for PDI-20335.
```
