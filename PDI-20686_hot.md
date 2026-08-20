# PDI-20686 — Hot deploy of the DET EE KAR into a running PDI 10.2.0.X

> **Read [`PDI-20686.md`](./PDI-20686.md) first.** That document is the source of truth for the **cold
> start** fix, which is committed and verified. **This** document is the source of truth for **hot
> deploy**, and is scoped to what hot deploy needs *on top of* the cold-start change set.
>
> **Status:** the OSGi half works with the committed cold-start change set — hot deploy, undeploy and
> redeploy have all been exercised on the test install and are now **clean end to end**
> ([§8.2](#82-the-undeployredeploy-cycle-verified-end-to-end)): deploy 16 s, undeploy 13 s, redeploy
> 12 s, and **zero exceptions of any kind** in `SpoonDebug.txt`. What is **not** yet committed is a
> small set of Spoon-side fixes that make a hot deploy indistinguishable from a cold start for objects
> Spoon had **already created** before the KAR arrived. They currently sit as **uncommitted changes in
> `pentaho-det-ee`**, plus **two uncommitted changes in `pentaho-osgi-bundles`** and one in
> **`pentaho-platform`** (the un/redeploy fixes, [§7A](#7a-defect-4--undeploy-aborts-pentaho-pdi-platforms-activation-stale-bundlecontext)
> and [§7B](#7b-defect-5--stale-class-loaders-on-pooled-threads-after-a-redeploy)) — see
> [§9 Remaining work](#9-remaining-work).

---

## 0. How to read this document

| Audience | Start here |
|---|---|
| **Deciding whether to ship hot deploy** | [§1 Answer in one paragraph](#1-answer-in-one-paragraph), [§9 Remaining work](#9-remaining-work), [§12 Recommendation](#12-recommendation) |
| **Implementing / reviewing the code** | [§5](#5-defect-1--xul-overlays-and-already-open-tabs), [§6](#6-defect-2--requirejs-config-served-truncated-not-a-500), [§7](#7-defect-3--det-state-is-not-restored-in-already-open-transformations), [§7A](#7a-defect-4--undeploy-aborts-pentaho-pdi-platforms-activation-stale-bundlecontext), [§7B](#7b-defect-5--stale-class-loaders-on-pooled-threads-after-a-redeploy) |
| **Testing hot deploy / undeploy** | [§3.3 Procedure](#33-procedure) and, before anything else, [§7C](#7c-not-a-defect--a-kar-present-in-deploy-at-startup-can-never-be-undeployed) — the one setup mistake that silently invalidates every undeploy result |
| **Re-measuring** | [§3 Method](#3-method), [§13 Reproducing this analysis](#13-reproducing-this-analysis) |

Two rounds of measurement went into this document. The first produced three wrong conclusions; they are
kept, with the reasons they were wrong, in [§10](#10-corrections-to-the-first-pass), because the reasons
are reusable. **Numeric measurements in [§4](#4-results-the-osgi-layer) and
[§8](#8-result-with-all-fixes-experiment-4) were taken against the pre-simplification change set**
(184 bundles); the committed change set provisions **181** bundles. Timings and pass/fail outcomes were
re-confirmed by manual testing after the simplification, but the byte-level figures were not re-captured
— they are marked where it matters.

---

## 1. Answer in one paragraph

**Hot deploy is not impossible, and the OSGi half already works.** Dropping the DET EE KAR into
`system/karaf/deploy/` of a fully booted PDI provisions DET in **19–21 seconds** (budget: 51 s), with no
stuck Blueprint containers and every endpoint — REST, Analyzer content generators, geo, and the full
RequireJS configuration — answering correctly. This is a direct consequence of the cold-start change
set: the single-batch feature resolution ([`PDI-20686.md` §6.4](./PDI-20686.md#64-why-nothing-in-the-kar-may-be-prerequisitetrue))
and the removal of the analyzer's cross-bundle `ApplicationContext` handshake
([`PDI-20686.md` §6.2](./PDI-20686.md#62-the-spring-context-factory)) work in **both** directions.

What does **not** work out of the box is everything that Kettle and Spoon bind **once**, at the moment an
object is created. Three defects were found, all of that same family:

| # | Defect | Effect | Fix | Where |
|---|---|---|---|---|
| 1 | DET's XUL overlay is applied only to `trans-graph` containers built **after** the plugin arrives | In transformation tabs already open at deploy time: no *Inspect* / *Run and Inspect* in the canvas right-click menu, **and a `NullPointerException` on every step right-click** | ~60 lines | `pentaho-det-ee` (uncommitted) |
| 2 | `RequireJsConfigManager` does not retry when its cached future is **cancelled**, and the servlet has already committed ~88 KB before the call | `require-init.js` is served as **HTTP 200 with a silently truncated, syntactically invalid body** — measured at 5 / 900 requests | ~10 lines | `pentaho-osgi-bundles` (uncommitted) |
| 3 | Kettle fires `TransAfterOpen` once; DET attaches its state listeners there | Transformations already open at deploy time get **no DET state loaded, no dirty marking, no save prompt, and nothing written back to the `.ktr`** | ~75 lines | `pentaho-det-ee` (uncommitted) |

A fourth defect, of a different family, was found on **undeploy**, and a fifth on the **redeploy** that
followed it; they are documented in
[§7A](#7a-defect-4--undeploy-aborts-pentaho-pdi-platforms-activation-stale-bundlecontext) and
[§7B](#7b-defect-5--stale-class-loaders-on-pooled-threads-after-a-redeploy). Both are *stale-reference*
defects: something captured at deploy time keeps being used after the bundle behind it is gone.

| # | Defect | Effect | Fix | Where |
|---|---|---|---|---|
| 4 | `PentahoSystem` holds **one static `BundleContext`**, and nothing invalidates it when the bundle that donated it is refreshed | Removing the KAR aborts the activation of `pentaho-pdi-platform`, which then stays **`Resolved`** — its `WebContextServlet`, `LocalizationServlet` and `XmiToDatabaseMetaDatasourceService` are gone until PDI restarts | ~50 lines | `pentaho-platform` (uncommitted) |
| 5 | A `CompositeClassLoader` captured as the context class loader of a **pooled thread** outlives the bundle revision it delegates to | After a redeploy, every XML parse on those threads fails — `Unable to load file step-attributes.xml`, ×10 in one session. Swallowed by `BaseStepMeta`, so it is **log noise**, not a functional failure | ~30 lines | `pentaho-osgi-bundles` (uncommitted) |

With all three applied, a hot deploy is functionally **indistinguishable** from a cold start, including
for transformations that were already open.

**Remaining effort: 4–5 engineer-days**, dominated by test-matrix execution rather than coding — the
code is written and compiles, and the suites it touches pass.

---

## 2. Why this needed testing at all

The cold-start design was described in earlier documents as *"cold-start only"*, on the assumption that
restoring Spring DM's behaviour would not survive dynamic provisioning. That assumption is **wrong**, and
it is now wrong for a stronger reason than when it was first disproven: the design no longer contains an
extender at all. Each plugin builds its own Spring context as an ordinary bean of its own Blueprint
container, so the context is created whenever Blueprint starts that bundle — at boot or hours later — and
disposed when Blueprint stops it. There is no ordering problem left to get wrong.

The real hot-deploy risk was never in OSGi. It is in **Kettle/Spoon objects that already exist** when
DET arrives.

---

## 3. Method

### 3.1 Environment

The measurements in [§4](#4-results-the-osgi-layer)–[§8](#8-result-with-all-fixes-experiment-4), and
the later manual cold **and** hot deploy confirmation of the committed change set, were taken on the
test / debug install:

```
~/pentaho/pdi/10.2.0.0-SNAPSHOT/20260818/pdi-ee-client-10.2.0.0-20260818.001718-1758-osgi/data-integration
```

with the pristine install kept read-only for comparison:

```
~/pentaho/pdi/10.2.0.0-SNAPSHOT/20260818/pdi-ee-client-10.2.0.0-20260818.001718-1758-osgi_clean/data-integration
```

This single test / debug install replaces the separate `…-osgi_cold` copy used by earlier revisions:
cold start and hot deploy are now exercised on the same install, reset between runs by step 4 of
[`PDI-20686.md` §9](./PDI-20686.md#9-deploy). Both installs carry the Karaf SSH boot-delegation fix of
[`SSH.md`](./SSH.md).

### 3.2 Controlling for the confounds that spoiled the first pass

| Confound | Control applied |
|---|---|
| Stale PDI processes surviving a "kill" and holding the Karaf/HTTP ports, so later probes hit the **wrong instance** | Processes are discovered with `pgrep -f "<install dir>"` (plus `SpoonDebug.sh` and the Spoon main class), terminated, then **polled until gone**, escalating to `-9`. Startup **refuses** to run if any instance is already alive. |
| Not knowing which instance a probe reached | Every run prints `Karaf Instance Number`, both ports, the owning PID (cross-checked against `lsof -iTCP -sTCP:LISTEN`), and the artifact **md5**. |
| Comparing "with fix" against a differently built "without fix" | Both variants are built from the same tree, stored under `/tmp/art/{fixed,nofix}/`, and identified by md5 in every run. `nofix` differs from `fixed` **only** by the files listed in [§9.1](#91-code--written-compiling-and-suites-green) — it is exactly the cold-start delivery, nothing else. |
| Under-sampling a sub-second failure window | Probes run at **~10 Hz** (900 requests per deploy), not 1 Hz. |
| A detector that cannot see the failure mode | Response **bodies are captured and validated with `node --check`**, not just status codes. |

> 📌 **Every** stop in this pass required escalation to `kill -9`; `SIGTERM` alone never stopped the
> JVM within 25 s. In the first pass the harness sent a single `SIGTERM` and moved on after 3–4 s.
> That is almost certainly why ports drifted between 8802/9051 and 8803/9052 in the first pass — a
> previous instance was still holding them. In this pass **every run used 8802/9051**, confirming a
> clean teardown each time.

### 3.3 Procedure

```bash
# Stage 1 - boot WITHOUT the KAR
prepare.sh {fixed|nofix} hot     # installs assembly-side jars, empties deploy/, clears caches
start_pdi.sh                     # refuses to start over a live instance; blocks until HTTP answers
# (a transformation is opened in Spoon here - this is the tab that exposes defects 1 and 3)

# Stage 2 - drop the KAR into the running instance
hotdrop.sh {fixed|nofix}         # drops the KAR, polls endpoints, 1 Hz + log diff
body_probe.sh {fixed|nofix}      # 10 Hz probe that saves and syntax-checks every response body
```

> ⚠️ **Stage 1 must boot with an empty `deploy/`, and `deploy/` must be emptied again before any
> restart.** A KAR that is present in `deploy/` when PDI starts is installed *concurrently* with
> Karaf's asynchronous boot provisioning and ends up **untracked**: it runs, but `kar:list`,
> `feature:list -i` and the FeaturesService state know nothing about it, so removing the file
> uninstalls nothing and the undeploy silently appears to do nothing at all. See
> [§7C](#7c-not-a-defect--a-kar-present-in-deploy-at-startup-can-never-be-undeployed).

### 3.4 Experiments run

| # | Variant | Mode | Purpose |
|---|---|---|---|
| 1 | `nofix` | hot | Baseline — what actually breaks |
| 2 | `fixed` | hot | Do fixes 1 and 2 work |
| 3 | `fixed` | cold | Control — is defect 3 hot-deploy-specific, and no cold-start regression |
| 4 | `fixed` | hot | Final — all three fixes together |

Plus two high-frequency body probes (`nofix`, `fixed`) over undeploy/redeploy cycles.

---

## 4. Results: the OSGi layer

### 4.1 Provisioning is fast and clean

| Metric | Before drop | After drop | Verdict |
|---|---|---|---|
| Active bundles | 110 | **184** ⚠️ | ✅ |
| `bundle:diag` | empty | **empty** | ✅ nothing stuck in grace period |
| `require-init.js` | 93 534 B | **180 969 B** | ✅ identical to cold start |
| Time to all endpoints ready | — | **19 s** (fixed) / **21 s** (nofix) | ✅ budget 51 s |
| `/content/analyzer/service`, `/generatedContent`, `/content/geojson`, `/cxf/det/core/persistence` | 404 | **200** | ✅ |

> ⚠️ **184 is a pre-simplification figure.** The committed change set ships four fewer bundles; a cold
> start of it lands on **181**, and a hot deploy should land on the same number. The *outcomes* above
> (empty `bundle:diag`, all endpoints 200, well inside the 51 s budget) were re-confirmed by manual
> testing of the committed change set; the exact counts and byte sizes were not re-captured.

Experiment 4, sampled every 3 s:

```
T+ 3s  det=404 analyzer=404 requireInit=93534B
T+ 6s  det=404 analyzer=404 requireInit=93534B
T+ 9s  det=404 analyzer=404 requireInit=93534B
T+13s  det=404 analyzer=404 requireInit=93534B
T+16s  det=404 analyzer=404 requireInit=93534B
T+19s  det=200 analyzer=200 requireInit=180969B     ← READY
```

### 4.2 Undeploy and redeploy are clean

Removing the KAR unwinds the feature (184 → 113 active bundles, DET endpoints 404), leaving only three
leaf libraries that other features also require (`commons-io`, `commons-lang`,
`com.google.guava.failureaccess`). Redeploy returns to a fully working state. The cycle was
exercised four times across this pass without divergence.

### 4.3 Spring context construction is not the bottleneck

The theoretical risk carried over from the cold-start work was the analyzer's Blueprint
`<reference id="spring" … timeout="5000"/>`. Measured, at the time, by cycling the bundle:

```
bundle:stop  <analyzer>  →  /content/analyzer/service = 404
bundle:start <analyzer>  →  /content/analyzer/service = 200   (within one shell round trip)
```

✅ Context construction and servlet re-registration are effectively instantaneous.

📌 **This risk has since been removed outright, not merely measured as small.** The committed change set
replaced that `<reference>` with a local `SpringContextFactory` bean
([`PDI-20686.md` §6.2](./PDI-20686.md#62-the-spring-context-factory)), so there is no cross-bundle
service handshake and no 5 s window in either direction. The context is created when Blueprint starts
the bundle and closed (`destroy-method="close"`) when Blueprint stops it — exactly the lifecycle a hot
deploy/undeploy needs.

---

## 5. Defect 1 — XUL overlays and already-open tabs

### 5.1 The two mechanisms DET uses to reach Spoon

| Mechanism | Used for | Registered via | Dynamic? |
|---|---|---|---|
| **Kettle extension points** | step flyout bar, DET step indicator, preview-panel button, right-click *enablement* | `<pen:di-plugin type="…ExtensionPointPluginType"/>` → `ExtensionPointMap` | ✅ Yes — `ExtensionPointMap` is a `PluginTypeListener` and resolves handlers on every call |
| **XUL menu overlays** | the *Inspect* (Shift-Ctrl-F9) and *Run and Inspect* (Ctrl-F9) items | `<pen:di-plugin type="…SpoonPluginType"/>` → `SpoonPluginManager` | ⚠️ Only while a container is being **constructed** |

```java
// Spoon.java:904     - once, while Spoon builds its main window   (category "spoon"  → Action menu)
// TransGraph.java:436 - once per transformation tab               (category "trans-graph" → canvas menu)
SpoonPluginManager.getInstance().applyPluginsForContainer( category, container );
```

### 5.2 Measured behaviour

| Scenario | Step flyout | Canvas right-click menu | Action menu | NPE on step right-click |
|---|---|---|---|---|
| Cold start (control) | ✅ | ✅ | ✅ | none |
| Hot deploy, tab opened **after** the drop | ✅ | ✅ | ✅ | none |
| **Hot deploy, tab already open** — *no fix* | ✅ | ❌ | ✅ | **yes, every right-click** |
| **Hot deploy, tab already open** — *with fix* | ✅ | ✅ | ✅ | none |

The gap is therefore **narrower than first reported** — the Action menu is fine — but **more severe**,
because it is not merely a missing menu item:

```
ERROR : Error calling TransStepRightClick extension point
ERROR : java.lang.NullPointerException
   at com.pentaho.det.di.impl.plugin.TransStepRightClickExtensionPoint.callExtensionPoint(TransStepRightClickExtensionPoint.java:42)
   at org.pentaho.di.core.extension.ExtensionPointMap.callExtensionPoint(ExtensionPointMap.java:142)
   at org.pentaho.di.ui.spoon.trans.TransGraph.setMenu(TransGraph.java:2611)
   at org.pentaho.di.ui.spoon.trans.TransGraph.mouseDown(TransGraph.java:830)
```

Line 42 is `menu.getElementById( "det-inspect" ).setDisabled( false );`. The extension point (dynamic,
so it *does* fire) assumes the element the overlay creates (not dynamic, so it *is* missing) exists.
The two mechanisms are coupled, and hot deploy breaks the coupling.

> ⚠️ **Not fully traced.** `applyPluginsForContainer("spoon", …)` has exactly one caller
> (`Spoon.java:904`, at startup), yet the Action menu items *do* appear after a hot deploy. Some path
> re-applies the `spoon`-category plugins; it was not identified. The conclusion is unaffected — the
> measured gap is exclusively the `trans-graph` container of already-open tabs — but anyone wanting to
> remove or narrow the fix should trace this first.

### 5.3 The fix

`DetMenuPlugin` already had `destroy-method="removeFromContainer"` — hot **un**deploy was designed for.
The fix is its missing counterpart, plus a correction to the removal path that only matters once
redeploy becomes a supported operation.

```diff
   <bean id="detMenuPlugin" class="com.pentaho.det.di.impl.plugin.DetMenuPlugin" scope="singleton"
+        init-method="applyToExistingContainers"
         destroy-method="removeFromContainer">
```

```java
public void applyToExistingContainers() {
  final Spoon spoon = getSpoon();
  if ( !isUiUp( spoon ) ) {
    return;                       // Cold start: applyToContainer will be called normally.
  }
  onUiThread( spoon, () -> {
    try {
      XulDomContainer main = spoon.getMainSpoonContainer();
      if ( main != null ) {
        applyToContainer( SPOON_CATEGORY, main );
      }
      for ( XulDomContainer c : openTransGraphContainers( spoon ) ) {
        applyToContainer( TRANS_GRAPH_CATEGORY, c );
      }
      spoon.enableMenus();
    } catch ( XulException e ) {
      getLogger().error( "DET Menu Plugin - could not apply overlays to the running Spoon instance", e );
    }
  } );
}
```

`applyToContainer()` itself is what makes this idempotent: it now returns early for a container it has
already overlaid. `isUiUp( Spoon )` and `onUiThread( Spoon, Runnable )` are the same guard and the same
`Display.syncExec` as before, factored into two `protected` methods so that the three call sites cannot
drift apart — and so that the tests, which cannot mock SWT's native `Display`, have a seam.

* **No `pentaho-kettle` change required** — `Spoon.getMainSpoonContainer()`,
  `Spoon.delegates.tabs.getTabs()` and `TransGraph.getXulDomContainer()` are already public. Hot-deploy
  support stays a **KAR-only** deliverable.
* **Self-disabling on cold start**, **idempotent**, and **symmetric** with `removeFromContainer()`.

**Latent bug in `removeFromContainer()`, fixed at the same time.** The Spoon container's overlay was
being removed with the *transformation* overlay's path:

```diff
  if ( spoonContainer != null ) {
-   spoonContainer.removeOverlay( DET_TRANS_OVERLAY_PATH );
+   spoonContainer.removeOverlay( DET_SPOON_OVERLAY_PATH );
  }
```

Both removals also shared a single `try`, so a failure on the first one skipped the second. This was
harmless while undeploy was effectively "shutting PDI down anyway", but with `applyToExistingContainers`
in place it is not: an undeploy that leaves the Spoon overlay behind means the next hot **re**deploy
applies it a second time, duplicating the *Inspect* items in the Action menu. Each removal now has its
own `try`.

**Two further defects in the same method, found in review.**

* The plugin is a **singleton** but Spoon builds **one `trans-graph` container per transformation tab**,
  and `applyToContainer()` kept the container in a single `transContainer` field — each tab overwriting
  the last. With three transformations open, an undeploy unwound only the third; the other two kept an
  overlay and an event handler pointing at classes of an uninstalled bundle (a class-loader leak, exactly
  what [§7.4](#74-the-fix) exists to avoid), and a redeploy overlaid them a second time. The containers
  are now kept in a list and all of them are unwound.
* `removeFromContainer()` had **no display guard**, unlike the three methods around it. On PDI shutdown
  `Spoon.dispose()` disposes the display *before* Blueprint destroys the beans, so `syncExec` would throw
  `SWTException( ERROR_DEVICE_DISPOSED )` out of the destroy-method on **every** shutdown. It now takes
  the same `isUiUp( spoon )` early return as the rest.

⚠️ **Not yet exercised at runtime.** The undeploy→redeploy cycle was measured at the OSGi layer
([§4.2](#42-undeploy-and-redeploy-are-clean)), but the *UI* consequence of a redeploy — duplicate menu
items — was never observed before or after this correction. It is reasoned from the code, not measured.
See [§9](#9-remaining-work).

---

## 6. Defect 2 — RequireJS config served truncated (not a 500)

### 6.1 What actually happens

`RequireJsConfigServlet.doGet` writes roughly **88 KB of boilerplate JavaScript** to the response
*before* it asks for the configuration:

```java
printWriter.write( ... );                                            // ~88 KB of helper functions
printWriter.write( "\n  var requireCfg = " + this.manager.getRequireJsConfig( contextRoot ) + "\n" );
```

By the time `getRequireJsConfig` runs, the response is **already committed**. Meanwhile every bundle
that registers a RequireJS package calls:

```java
public void invalidateCachedConfigurations() {
  this.cachedConfigurations.forEach( ( s, f ) -> f.cancel( true ) );   // ← cancels in-flight waiters
  ...
}
```

and `getRequireJsConfig`'s three-attempt retry loop catches only `InterruptedException` and
`ExecutionException`:

```java
try {
  result = cache.get();
} catch ( InterruptedException e ) {
  // ignore
} catch ( ExecutionException e ) {
  lastException = e;
  this.invalidateCachedConfigurations();
}
```

`Future.get()` on a **cancelled** future throws `CancellationException`, which is unchecked, escapes
the loop, and propagates out of the servlet. Jetty logs it — and, because the response is already
committed, **cannot** turn it into a 500. The client receives **HTTP 200 with a truncated body**.

### 6.2 Evidence

900 requests at ~10 Hz across an undeploy/redeploy window, every body saved and syntax-checked:

| | `nofix` | `fixed` |
|---|---|---|
| Requests | 900 | 900 |
| **Non-200 responses** | **0** | **0** |
| Response sizes seen | 88 213 ×5, 93 534 ×455, 180 969 ×440 | 88 952 ×4, 92 542 ×3, 93 534 ×363, 180 969 ×530 |
| **Bodies failing `node --check`** | **5** (all the 88 213 B ones) | **0** |
| `CancellationException` in log | **7** | **0** |
| `WARN:oejs.HttpChannel … require-init.js` | 7 | **0** |

The 88 213-byte bodies end exactly where the configuration should have been written:

```
    return versionedBaseModuleId + moduleIdLeaf;
  }
```

— i.e. the file stops immediately before `var requireCfg = `. `node --check` reports a syntax error.
Jetty's WARN timestamps (15:21:18.348/.388/.546/.571/.596/.676/.702) match the probe's own request
timestamps to within ~7 ms, confirming these were the probe's requests and that each returned **200**.

> **This failure mode is undetectable by status code.** That is exactly why the first pass declared it
> fixed on evidence that could not have seen it ([§10](#10-corrections-to-the-first-pass)).

### 6.3 The fix

```diff
       try {
         result = cache.get();
       } catch ( InterruptedException e ) {
         // ignore
+      } catch ( CancellationException e ) {
+        // [PDI-20686] A concurrent invalidateCachedConfigurations() cancelled the future this call was
+        // waiting on. Routine while bundles are still registering their RequireJS packages - most
+        // visibly during a hot deploy - so retry and pick up the newly scheduled build.
+        lastException = e;
+
+        this.cachedConfigurations.remove( baseUrl, cache );
       } catch ( ExecutionException e ) {
         lastException = e;

         this.invalidateCachedConfigurations();
       }
```

The `remove( baseUrl, cache )` matters: `invalidateCachedConfigurations()` cancels every future and
*then* clears the map, so a thread that observes the cancellation between those two steps would
otherwise read the same dead future back out on the next iteration and burn a retry. Removing it
conditionally (only if it is still the entry this thread was waiting on) makes the retry loop reach a
freshly scheduled build on its very next pass, and cannot evict a build that a later invalidation
already replaced.

Note this catch does **not** call `invalidateCachedConfigurations()` — unlike the `ExecutionException`
branch. A cancellation means an invalidation has *already* happened; invalidating again would cancel the
replacement build that this thread is about to wait on, and the three retries would chase each other.

With the fix the intermediate responses (88 952 / 92 542 bytes) are **complete, valid** configurations
that simply do not yet list DET's packages — the correct behaviour while provisioning is in progress.

**The exhausted-retry message, corrected in review.** A `CancellationException` carries neither a cause
nor a message, so three cancellations in a row produced
`{}; // Error computing RequireJS Config: unknown error` — and since the class has no logger and the
exception is no longer propagated to Jetty, that string was the *only* trace of the failure anywhere.
The fallback now falls back to `lastException.toString()`, which at least names the exception type in
the served response. (The measured "0 `CancellationException` in log" in [§6.2](#62-evidence) is
therefore partly the exception no longer being logged — the retry is what makes the response valid.)

> ⚠️ **Delivery note.** `requirejs-manager-impl` ships **inside the PDI assembly**, not in the KAR.
> Like the Analyzer bean restoration, this is a **PDI 10.2.0.10 deliverable**, so hot-deploy support
> is *not* purely a KAR change. Testing it therefore requires the locally built jar to be copied over
> the one in the install — step **2b** of [`PDI-20686.md` §9](./PDI-20686.md#9-deploy); rebuilding
> `pentaho-osgi-bundles` alone is not enough.
>
> 📄 The bug is **pre-existing and generic** — any bundle installation that changes the RequireJS
> package set can trigger it, in PDI *and* Pentaho Server. Hot-deploying DET just makes it easy to hit.

---

## 7. Defect 3 — DET state is not restored in already-open transformations

### 7.1 Symptom

Reported during Experiment 1 and reproduced in Experiment 2: after a hot deploy, in a transformation
that was **already open**, entering DET shows an empty Explore perspective even though the `.ktr` has
saved DET state; creating visualisations and leaving DET loses them; and closing the transformation
does **not** raise the usual "save transformation?" prompt — the edits never mark it dirty.

### 7.2 Root cause

```java
@ExtensionPoint( id = "TransformationAfterOpenExtensionPoint", extensionPointId = "TransAfterOpen" )
public void callExtensionPoint( LogChannelInterface log, Object object ) {
  TransMeta transformation = (TransMeta) object;
  transformation.addNameChangedListener( transNameChangedListener );
  transformation.addFilenameChangedListener( transFileNameChangedListener );
  transformation.addStepChangeListener( stepNameChangedListener );
  stateRegistry.setStates( stateUtil.getTransformationKey( transformation ), transformation );
}
```

`TransAfterOpen` fires **once**, when a transformation is opened. A `TransMeta` that already existed
when DET arrived therefore has **no DET listeners attached** and **no state registered** — so nothing
is read from the `.ktr`, nothing marks the transformation dirty, and nothing is written back.

### 7.3 Confirmation

| Test | Result |
|---|---|
| Hot deploy, already-open tab | ❌ state lost |
| Same session — **close and reopen** the transformation | ✅ **state persists correctly** |
| **Cold start** (Experiment 3), same `.ktr`, same workflow | ✅ works normally |

The reopen test is decisive: it is the manual equivalent of firing `TransAfterOpen`, and it fully
restores correct behaviour. This is hot-deploy-specific, **not** a pre-existing DET bug.

**Workaround without any code change:** after hot-deploying DET, close and reopen any transformation
that was already open.

### 7.4 The fix

Same pattern as defect 1 — an `init-method` on the extension point's own Blueprint bean, which
re-runs it against the transformations Spoon already has open. It calls DET's own extension point
directly rather than re-firing Kettle's global `TransAfterOpen`, so no other plugin's handler runs
twice.

```diff
   <bean id="transTransformationAfterOpenExtensionPoint"
         class="com.pentaho.det.di.impl.plugin.TransformationAfterOpenExtensionPoint"
-        scope="singleton">
+        scope="singleton"
+        init-method="applyToOpenTransformations">
```

```java
public void applyToOpenTransformations() {
  Spoon spoon = Spoon.getInstance();
  if ( spoon == null || spoon.getDisplay() == null || spoon.getDisplay().isDisposed()
      || spoon.delegates == null || spoon.delegates.tabs == null ) {
    return;                                        // Cold start: nothing is open yet.
  }
  LogChannelInterface log = spoon.getLog();
  spoon.getDisplay().syncExec( () -> {
    for ( TabMapEntry entry : spoon.delegates.tabs.getTabs() ) {
      if ( !( entry.getObject() instanceof TransGraph ) ) {
        continue;
      }
      TransMeta transMeta = ( (TransGraph) entry.getObject() ).getTransMeta();
      if ( transMeta == null ) {
        continue;
      }
      try {
        callExtensionPoint( log, transMeta );
      } catch ( KettleException e ) {
        log.logError( "DET could not restore its state for the already open transformation '"
            + transMeta.getName() + "'", e );
      }
    }
  } );
}
```

✅ Verified in Experiment 4: with all three fixes, an already-open `DET_Smoketest.ktr` gets its menus,
throws no NPE, and **restores its saved DET state**.

**The missing counterpart, added afterwards.** Unlike `DetMenuPlugin`, this bean had **no
`destroy-method`** — so the fix above was asymmetric:

```diff
   <bean id="transTransformationAfterOpenExtensionPoint"
         class="com.pentaho.det.di.impl.plugin.TransformationAfterOpenExtensionPoint"
         scope="singleton"
         init-method="applyToOpenTransformations"
+        destroy-method="detachFromOpenTransformations">
```

`TransMeta` instances belong to Spoon and **outlive the DET bundle**. Without the detach, a hot undeploy
leaves three listeners — whose classes come from an uninstalled bundle — attached to every open
transformation, which both leaks the bundle's class loader and means a subsequent hot **re**deploy
registers a *second* set of listeners, handling every rename and step change twice.

```java
public void detachFromOpenTransformations() {
  Spoon spoon = Spoon.getInstance();
  if ( spoon == null || spoon.getDisplay() == null || spoon.getDisplay().isDisposed()
      || spoon.delegates == null || spoon.delegates.tabs == null ) {
    return;                                        // PDI shutdown: the display is already disposed.
  }
  spoon.getDisplay().syncExec( () -> {
    for ( TabMapEntry entry : spoon.delegates.tabs.getTabs() ) {
      // ... for each open TransGraph's TransMeta:
      transMeta.removeNameChangedListener( transNameChangedListener );
      transMeta.removeFilenameChangedListener( transFileNameChangedListener );
      transMeta.removeStepChangeListener( stepNameChangedListener );
    }
  } );
}
```

The three `remove*Listener` methods already exist on `TransMeta` / `AbstractMeta`, and the listeners are
Blueprint singletons injected into this bean, so removal by identity is exact. No `pentaho-kettle`
change is needed here either.

⚠️ **Not yet exercised at runtime** — same caveat as the `removeFromContainer()` correction in
[§5.3](#53-the-fix). See [§9](#9-remaining-work).

---

## 7A. Defect 4 — undeploy aborts `pentaho-pdi-platform`'s activation (stale `BundleContext`)

Found on 2026-08-18, on a run that had nothing to do with the three defects above: DET was working in
the steps of both open transformations, the KAR was then **removed**, and `SpoonDebug.txt` grew this.

### 7A.1 Symptom

```
org.apache.karaf.features.internal.util.MultiException: Error restarting bundles:
  Activator start error in bundle pentaho-pdi-platform [201].
  …
  Caused by: java.lang.IllegalStateException: Invalid BundleContext.
    at org.apache.felix.framework.BundleContextImpl.checkValidity(BundleContextImpl.java:491)
    at org.apache.felix.framework.BundleContextImpl.registerService(BundleContextImpl.java:308)
    at org.pentaho.platform.engine.core.system.objfac.OSGIRuntimeObjectFactory.registerReference(OSGIRuntimeObjectFactory.java:112)
    …
    at org.pentaho.platform.engine.core.system.PentahoSystem.registerObject(PentahoSystem.java:1541)
    at org.pentaho.platform.pdi.PdiPlatformActivator.start(PdiPlatformActivator.java:46)
```

**It is not normal, and it is not cosmetic.** It is an *undeploy*-only failure — every prior
measurement of the undeploy path looked at `bundle:diag` and the DET endpoints, neither of which
notices it.

### 7A.2 Root cause

1. `PentahoSystem` bridges to OSGi through **one static `BundleContext`**, donated once at boot by
   `pentaho-blueprint-activators`
   (`pentaho-karaf-assembly/pentaho-blueprint-activators/src/main/resources/standard.xml` →
   `PentahoOSGIActivator.setBundleContext` → `PentahoSystem.setBundleContext` →
   `OSGIRuntimeObjectFactory.bundleContext`).
2. Removing the KAR triggers a refresh cascade that stops and restarts **both**
   `pentaho-blueprint-activators` and `pentaho-pdi-platform` **[201]**.
3. A `BundleContext` is invalidated the moment its bundle stops, and Blueprint containers are rebuilt
   **asynchronously** — so there is a window in which the static field still points at the **dead**
   context, and nothing in the code path notices.
4. `PdiPlatformActivator.start()` runs inside that window. Line 45 —
   `PentahoSystem.get( IAuthorizationPolicy.class )` — returns `null` (`objectDefined()` already
   swallowed the same `IllegalStateException`), so line 46 goes on to *register*, which throws.
5. Felix turns that into `BundleException: Activator start error`, and **bundle 201 never starts**.

The one existing guard (`objectDefined()`) therefore made the failure *worse*: it converted "OSGi is
unavailable" into "the object is missing", which is precisely what drove the activator into the
registration branch.

### 7A.3 Consequence

`pentaho-pdi-platform` stays `Resolved`, so everything its Blueprint publishes is gone until PDI is
restarted: `WebContextServlet` (`webcontext.js`), the **`LocalizationServlet`** (`/i18n`) and
`XmiToDatabaseMetaDatasourceService`.

🔍 That is a strong candidate explanation for residual risk 8 / [`PDI-20686-errata.md`
§C.6](./PDI-20686-errata.md#c6-new-finding-a-failed-deploy-poisons-the-container-until-pdi-restarts)
— "a failed deploy poisons the container", whose signature was an Analyzer 500 with a
**`LocalizationServiceImpl` NPE**, served by exactly this bundle. ⚠️ Hypothesis, not yet proven: it
predicts that the poisoned state is `pentaho-pdi-platform` sitting at `Resolved`, which a single
`bundle:list` at that moment would confirm.

### 7A.4 The fix

`pentaho-platform` → `core/…/objfac/OSGIRuntimeObjectFactory.java`. The factory **already** has a
deferral mechanism for "OSGi is not available yet", used before the context arrives, and
`setBundleContext()` replays everything held in it. The fix is to treat *invalidated* exactly like *not
yet available*, so the same mechanism carries the registrations across the refresh:

```java
private boolean isBundleContextUsable() {
  BundleContext context = this.bundleContext;
  if ( context == null ) {
    return false;
  }
  try {
    context.getBundle();   // throws IllegalStateException once the context has been invalidated
    return true;
  } catch ( IllegalStateException e ) {
    return false;
  }
}
```

| Change | Why |
|---|---|
| `registerReference()` defers instead of calling `registerService()` when the context is unusable | The activator completes, the object stays available on the non-OSGi factory, and the registration is republished by the next `setBundleContext()` — **self-healing**, no restart needed |
| The registration loop also catches `IllegalStateException`, unregisters what it managed to publish, and defers | The context can be invalidated *between* the check and the call |
| `objectDefined()` and `getReferencesByQuery()` degrade to the non-OSGi factory while the context is unusable; `getServiceReference()` moved **inside** the guarded `try` | It sat outside the existing `IllegalStateException` catch, so a lookup in the same window threw rather than falling back |
| `setBundleContext()` drains the deferred list under the lock and replays **outside** it | A replay can now defer again (if the new context is already dead), which would otherwise mutate the list being iterated. Also fixes a pre-existing bug: the iterator was created *outside* the `synchronized` block |
| `bundleContext` is `volatile` | It is written by the Blueprint thread and read from arbitrary threads |

**Rejected alternative:** catching the exception in `PdiPlatformActivator.start()`
(`pentaho-osgi-bundles`, already in the change set). One line, but the bundle would then start with
`IAuthorizationPolicy`, `IPluginResourceLoader` and `IPluginManager` silently **unregistered** — it
hides the failure instead of healing it.

### 7A.5 Status

✅ 5 new tests in `OSGIRuntimeObjectFactoryTest` (invalid on entry, invalid mid-registration, replay on
the next context, lookup fallback); `pentaho-platform-core` green at **602** tests.
✅ The two rebuilt classes are patched into the test install's
`lib/pentaho-platform-core-10.2.0.0-SNAPSHOT.jar` (original kept as `…jar.PDI-20686.bak`).
✅ **Verified at runtime, twice.** First on 2026-08-18 19:36–19:41: a full deploy → undeploy → deploy
cycle with the patched jar produced **0** `MultiException` and **0** `Invalid BundleContext` in
`SpoonDebug.txt`, and DET worked in both open transformations afterwards — that run also exposed
defect 5 ([§7B](#7b-defect-5--stale-class-loaders-on-pooled-threads-after-a-redeploy)). Confirmed
again on the fully clean cycle of [§8.2](#82-the-undeployredeploy-cycle-verified-end-to-end).

---

## 7B. Defect 5 — stale class loaders on pooled threads after a redeploy

Found on 2026-08-18, on the first **deploy → undeploy → deploy** cycle run with the defect-4 fix in
place. DET itself was fine — switching between the two transformations and opening DET on steps in both
worked — but `SpoonDebug.txt` collected **10** stack traces.

### 7B.1 Symptom

```
org.pentaho.di.core.exception.KettleException:
Unable to load file step-attributes.xml
Error reading information from input stream
…
Caused by: java.lang.NullPointerException
  at org.apache.felix.framework.BundleRevisionImpl.getResourcesLocal(BundleRevisionImpl.java:558)
  at org.apache.felix.framework.BundleWiringImpl$BundleClassLoader.getResources(BundleWiringImpl.java:2550)
  at org.pentaho.platform.pdi.CompositeClassLoader.getResources(CompositeClassLoader.java:49)
  at org.pentaho.platform.pdi.CompositeClassLoader.getResources(CompositeClassLoader.java:49)
  at java.base/java.util.ServiceLoader$LazyClassPathLookupIterator.nextProviderClass(ServiceLoader.java:1196)
  …
  at java.xml/javax.xml.parsers.DocumentBuilderFactory.newInstance(DocumentBuilderFactory.java:140)
  at org.pentaho.di.core.xml.XMLParserFactoryProducer.createSecureDocBuilderFactory(XMLParserFactoryProducer.java:50)
  at org.pentaho.di.trans.step.BaseStepMeta.loadStepAttributes(BaseStepMeta.java:900)
  at org.pentaho.di.trans.steps.selectvalues.SelectValuesMeta.<init>(SelectValuesMeta.java:87)
  at org.pentaho.di.trans.dataservice.SqlTransGenerator.generateConversionStep(SqlTransGenerator.java:227)
  … mondrian.rolap.agg.SegmentLoader$SegmentLoadCommand.call …
```

**Timing is the diagnosis:** the same DET operations at 19:37–19:38, after the **first** deploy,
produced **zero** of these. All 10 fall between 19:40:20 and 19:40:42 — after the undeploy (≈19:38) and
the redeploy (19:39).

### 7B.2 Root cause

`CompositeClassLoader` appears **twice** in the stack because two of them are nested:

| Layer | Built by | Delegates to |
|---|---|---|
| inner | `SpringContextFactory.createForBundle()` — kept for the life of the `ApplicationContext` | `wiring.getClassLoader()` (the plugin **bundle**) → Spring's loader |
| outer | `ContentGeneratorServlet.service()` — built per request from `applicationContext.getClassLoader()` | the inner composite → the caller's context class loader |

The servlet restores the context class loader in a `finally`, but a `Thread` **inherits** its context
class loader from whichever thread created it. Mondrian builds its `SegmentLoader` pool lazily, on the
first query — i.e. inside a content-generator request — so those pooled threads keep the outer composite
**forever**.

The undeploy then disposes the plugin bundle's revision. Felix answers a request on a disposed revision
with a **`NullPointerException`** out of `BundleRevisionImpl.getResourcesLocal` (its content is `null`)
rather than with an empty result. So every `DocumentBuilderFactory.newInstance()` on one of those pooled
threads — `ServiceLoader` enumerating `META-INF/services/…` through the context class loader — blows up,
and Kettle reports it as a failure to read `step-attributes.xml`.

**Why it is only noise:** `BaseStepMeta`'s constructor catches the exception and calls
`printStackTrace()` ([`BaseStepMeta.java:102`](https://github.com/pentaho/pentaho-kettle/blob/10.2/engine/src/main/java/org/pentaho/di/trans/step/BaseStepMeta.java)),
so the step is built without its attribute descriptions. Every data-service query in the log still
returned its rows, and the redeployed DET worked in both open transformations.

⚠️ It also means each cycle leaves one more dead composite pinned to those threads — a slow class-loader
leak, and the reason the traces would multiply over repeated cycles.

### 7B.3 The fix

`pentaho-osgi-bundles` → `pentaho-pdi-platform/…/CompositeClassLoader.java`: treat a delegate that
belongs to a disposed revision as one that has nothing to contribute, and let the other one answer.

| Method | Behaviour when a delegate throws `NullPointerException` |
|---|---|
| `getResources()` | Skip that delegate, merge what the other one has. This is the one that matters: the surviving delegate is PDI's `lib/`, which **does** carry `META-INF/services/javax.xml.parsers.DocumentBuilderFactory` — so the parse now succeeds instead of merely failing quietly |
| `getResource()` | Same, falling through to the secondary; `null` only if both are dead |
| `loadClass()` | The superclass delegates to the primary *before* `findClass()` is reached, so a dead primary is caught here and the secondary is asked instead |
| `findClass()` | A dead secondary becomes a `ClassNotFoundException` (with the NPE as its cause) rather than an NPE escaping into arbitrary caller code |

Each skip is logged at **debug** — it is expected after a redeploy and would otherwise become noise of
its own.

Catching `NullPointerException` is deliberate and is the only signal available: the bundle is `Active`
(with a *new* revision) while the *old* revision's class loader is the dead one, so
`Bundle.getState()` cannot tell them apart.

### 7B.4 Status

✅ 6 new tests in `CompositeClassLoaderTest` (disposed primary, disposed secondary, both disposed,
`loadClass` fallback, `findClass` mapping); `pentaho-pdi-platform` green at **75** tests.
✅ The rebuilt class is patched into the test install's
`system/karaf/system/pentaho/pentaho-pdi-platform/10.2.0.0-SNAPSHOT/pentaho-pdi-platform-10.2.0.0-SNAPSHOT.jar`
(original kept as `…jar.PDI-20686.bak`); it needs a **PDI restart** to take effect, since Karaf caches
the bundle.
✅ **Runtime-verified** on 2026-08-18 (20:50–20:55) — see [§8.2](#82-the-undeployredeploy-cycle-verified-end-to-end).
Full deploy → undeploy → deploy with two transformations open and DET exercised in both tabs after
**each** deploy: **0** `step-attributes.xml` traces (10 in the previous run) and **0** exceptions of
any kind. The class actually loaded was confirmed to be the patched one (`logDisposedDelegate` and
the slf4j `LOGGER` are present in the deployed bundle and absent from the `.bak`).

📄 **Note for the commit:** `CompositeClassLoader.java` is already part of the *committed* cold-start
change set ([`PDI-20686.md` §5](./PDI-20686.md#5-change-footprint-per-repository)), so this lands as a
follow-up commit on the same branch, not as a new file.

---

## 7C. Not a defect — a KAR present in `deploy/` **at startup** can never be undeployed

> **This is not caused by the change set, and there is no code fix in PDI-20686.** It is a
> pre-existing Karaf/PDI boot race. It is documented here because it *looks* exactly like "hot
> undeploy is broken", and it cost a full diagnosis round to rule out.

### 7C.1 Symptom

The KAR is removed from `system/karaf/deploy/`, PDI keeps working **as if DET were still deployed**,
and `SpoonDebug.txt` is completely clean — no errors at all, so there is nothing to chase.

### 7C.2 Evidence

Taken from the live instance after the failed undeploy:

| Probe | Result |
|---|---|
| `kar:list` | **empty** |
| `feature:repo-list \| grep det` | **nothing** |
| `feature:list -i \| grep det` | **nothing** |
| `bundle:diag` | **empty** |
| `bundle:list \| grep det` | **`det-api-core`, `det-impl-core`, `det-impl-pdi`, `det-impl-pdi-metastore-supplier`, `det-impl-pdi-webclient`, `det-webclient-impl` — all `Active`** |
| `/cxf/det/core/persistence/states`, `/osgi/webcontext.js`, `/content/analyzer/service` | **200** |
| `${karaf.data}/kar/` | **emptied** at the moment the KAR was removed |

So Karaf *did* process the removal — it deleted its own KAR record — but uninstalled nothing.

Two file-system facts settle it. First, the Felix bundle cache
(`system/karaf/caches/spoon/data-1/cache/bundle<N>/bundle.info`) dates **every one of the 75 KAR
bundles to 19:59:51**, 11 seconds after Karaf launched (19:59:40) and while boot provisioning was
still running (the last boot bundle was written at 20:00:14). The KAR was therefore installed
*during* boot, not into a running instance. Second, the FeaturesService state file
(`caches/spoon/data-1/cache/bundle19/data/state.json`, bundle 19 =
`org.hitachivantara.karaf.features.core`) lists **213 managed bundles — and not one of those 75**.
Their feature repository is absent from `repositories`, their features are absent from `features`
and `installed`. The whole KAR payload is **orphaned**: running, but unknown to the FeaturesService.

### 7C.3 Root cause

`etc/org.apache.karaf.features.cfg` sets **`featuresBootAsynchronous=true`**, so boot features are
provisioned on a background thread while `felix.fileinstall` (poll 1000 ms, start level 80) is
already scanning `deploy/`. A KAR sitting there at startup is therefore installed **concurrently**
with boot provisioning, and the boot `Deployer`'s `saveState` — computed from a snapshot taken
before the KAR arrived — overwrites the repository, feature and managed-bundle records the KAR
install had just written. The bundles stay `Active`; the bookkeeping that owns them is gone.

Undeploy then hits this, in Karaf 4.4.6 `KarServiceImpl` (verified against the 4.4.6 sources):

```java
public void uninstall( String karName, boolean noAutoRefreshBundles ) throws Exception {
  File karDir = new File( storage, karName );
  if ( !karDir.exists() ) { throw new IllegalArgumentException( … ); }
  List<URI> featuresRepositories = readFromFile( new File( karDir, FEATURE_CONFIG_FILE ) );
  uninstallFeatures( featuresRepositories, noAutoRefreshBundles );   // ← see below
  for ( URI featuresRepository : featuresRepositories ) {
    featuresService.removeRepository( featuresRepository );          // ← no-op, not registered
  }
  deleteRecursively( karDir );                                       // ← this is what did run
}

private void uninstallFeatures( List<URI> featuresRepositories, boolean noAutoRefreshBundles ) {
  for ( Repository repository : featuresService.listRepositories() ) {   // ← KAR's repo is NOT in here
    for ( URI karFeatureRepoUri : featuresRepositories ) {
      if ( repository.getURI().equals( karFeatureRepoUri ) ) { … }       // ← never matches
    }
  }
}
```

`uninstallFeatures` only acts on repositories **currently registered** with the FeaturesService.
The KAR's repository is not, so the method is a silent no-op, `removeRepository` is a no-op, and the
only thing that happens is `deleteRecursively( karDir )` — which is precisely why `${karaf.data}/kar/`
emptied while everything stayed running. Nothing is logged, hence the clean `SpoonDebug.txt`.

### 7C.4 Consequence and guidance

**Hot deploy is only valid into a fully started PDI.** There are exactly two supported ways to get
DET in:

| Path | How | Undeployable? |
|---|---|---|
| **Cold start** | KAR content in `system/` + the feature in `featuresBoot` ([`PDI-20686.md` §9](./PDI-20686.md#9-deploy)) | n/a — it is part of the assembly |
| **Hot deploy** | drop the KAR into `system/karaf/deploy/` **after** PDI answers on its HTTP port | ✅ yes, cleanly |
| ⛔ **KAR left in `deploy/` across a restart** | — | ❌ **no** — untracked install, undeploy is a no-op |

Rules for anyone testing or documenting hot deploy:

1. **Never start PDI with the KAR already in `deploy/`.** `prepare.sh` empties `deploy/` for this
   reason; keep it that way.
2. **Remove the KAR from `deploy/` before shutting PDI down**, or accept that the next boot produces
   an untracked install.
3. **Recovering from one** requires either uninstalling the orphan bundles by id from the Karaf
   shell, or — simpler and reliable — clearing `system/karaf/caches/` and restarting.

A code-level fix does exist (`featuresBootAsynchronous=false` in `pentaho-karaf-assembly` would
serialise boot provisioning ahead of the deploy-dir scan, eliminating the race) but it is **out of
scope for PDI-20686**: it changes Karaf boot behaviour for every Pentaho client and trades startup
latency for a case that only arises in developer workflows. Tracked as residual risk 13.

---

## 8. Result with all fixes (Experiment 4)

| Check | Cold start | Hot deploy |
|---|---|---|
| Time to fully provisioned | ~40 s (budget 61 s) | **19 s** (budget 51 s) |
| Genuine errors in `SpoonDebug.txt` | 0 | **0** |
| `CancellationException` | 0 | **0** |
| Active bundles / `bundle:diag` | 184 ⚠️ / empty | **184 ⚠️ / empty** |
| `require-init.js` | 180 969 B | **180 969 B** |
| Truncated bodies (900 × 10 Hz probe) | n/a | **0** |
| Analyzer, geo, DET REST endpoints | 200 | **200** |
| Model view returns real folders/fields | ✅ | ✅ |
| DET menus — new tabs | ✅ | ✅ |
| DET menus — **already-open** tabs | n/a | ✅ |
| NPE on step right-click | none | **none** |
| DET state restored — **already-open** tabs | n/a | ✅ |

⚠️ 184 is the pre-simplification bundle count; the committed change set lands on **181**
([§4.1](#41-provisioning-is-fast-and-clean)).

**No cold-start regression:** Experiment 3 ran a full cold start with all three fixes — 79-line log,
zero errors, all endpoints 200, `require-init.js` 180 969 bytes.

### 8.1 One log message that is *not* a defect

```
ERROR [AnalyzerContentGenerator] Exception occurred in Pentaho Analyzer content generator.
java.lang.RuntimeException: No controller mapping for: /service
```

This is produced **by the health-check probe itself**. Proven directly: with the count at 2, a single
`GET /content/analyzer/service` took it to 3 — one request, one message, HTTP **200**. A bare GET on
that path carries no valid Analyzer command, so the content generator renders its error page. Its
appearance actually **confirms** the servlet and Spring context are wired correctly. It occurs
identically on cold start and should be excluded when scanning logs. The full list of expected log
noise is in [`PDI-20686.md` §10.1](./PDI-20686.md#101-expected-log-noise--none-of-these-are-regressions).

### 8.2 The undeploy/redeploy cycle, verified end to end

Run on 2026-08-18 20:50–20:55 against the 20260818 install with **both** `pentaho-platform-core` and
`pentaho-pdi-platform` patched, a cleared Karaf cache, an empty `deploy/`
([§7C](#7c-not-a-defect--a-kar-present-in-deploy-at-startup-can-never-be-undeployed)) and **two
transformations open before the KAR arrived**. DET was exercised in both tabs after each deploy —
which is the path that produced defect 5's traces on the previous run.

| Step | Time to result | DET REST | `require-init.js` | `${karaf.data}/kar/` |
|---|---|---|---|---|
| **Deploy** | **16 s** | 200 | 200, **180 969 B** | KAR present |
| **Undeploy** | **13 s** | 404 | 200, **93 534 B** | **emptied** |
| **Redeploy** | **12 s** | 200 | 200, **180 969 B** | KAR present |

`SpoonDebug.txt` for the entire run — 712 lines covering all three transitions and the DET usage in
between:

| Assertion | Result |
|---|---|
| `Unable to load file step-attributes.xml` | **0** (was 10) — defect 5 closed |
| `BundleRevisionImpl.getResourcesLocal` | **0** — defect 5 closed |
| `Error restarting bundles` / `Invalid BundleContext` | **0** / **0** — defect 4 closed |
| `CancellationException` | **0** |
| `unresolved requirement` | **0** |
| `ERROR` lines | **0** — even the Analyzer probe noise of [§8.1](#81-one-log-message-that-is-not-a-defect) is absent, because this run used no `/content/analyzer/service` probe |
| **Any exception at all** | **0** |
| Orphaned bundles ([§13](#13-reproducing-this-analysis)) | **0** — the install is properly tracked |
| DET state after the redeploy | **9** and **8** step states — identical to the rev 5 figures, so defect 3's fix survives a full cycle |

This is the first run of the whole cycle with **nothing** in the log, and it closes tasks 8 and 10 of
[§9.2](#92-what-is-genuinely-left).

---

## 9. Remaining work

Everything in this section is **uncommitted** and deliberately kept out of the cold-start commits, so
that [`PDI-20686.md`](./PDI-20686.md) can ship on its own.

### 9.1 Code — written, compiling, and suites green

| Repository | File | Change |
|---|---|---|
| `pentaho-det-ee` | `det/impls/pdi/…/plugin/DetMenuPlugin.java` | `applyToExistingContainers()` + `openTransGraphContainers()` (defect 1); `removeFromContainer()` overlay-path and try-scope correction ([§5.3](#53-the-fix)) |
| `pentaho-det-ee` | `det/impls/pdi/…/plugin/TransformationAfterOpenExtensionPoint.java` | `applyToOpenTransformations()` (defect 3) + `detachFromOpenTransformations()` ([§7.4](#74-the-fix)) |
| `pentaho-det-ee` | `det/impls/pdi/…/OSGI-INF/blueprint/blueprint.xml` | wires the `init-method`s and the new `destroy-method` |
| `pentaho-osgi-bundles` | `pentaho-requirejs-osgi-manager/core/impl/…/RequireJsConfigManager.java` | `CancellationException` retry (defect 2, [§6.3](#63-the-fix)) |
| `pentaho-platform` | `core/…/objfac/OSGIRuntimeObjectFactory.java` (+ `OSGIRuntimeObjectFactoryTest`) | Stale-`BundleContext` deferral (defect 4, [§7A.4](#7a4-the-fix)) — branch `PDI-20686` |
| `pentaho-osgi-bundles` | `pentaho-pdi-platform/…/CompositeClassLoader.java` (+ `CompositeClassLoaderTest`) | Disposed-revision tolerance (defect 5, [§7B.3](#7b3-the-fix)) |

✅ Both affected modules compile and their existing suites pass unchanged
(`det/impls/pdi`: 27 tests; `pentaho-requirejs-osgi-manager/core/impl`: 180 tests).

### 9.2 What is genuinely left

| # | Task | Estimate | Notes |
|---|---|---|---|
| 1 | **Runtime-verify the two symmetry fixes** | 0.25 d | 🟡 **Partly done.** An undeploy→redeploy cycle *with two transformations open* has now been run: undeploy unwinds cleanly (181→113 `Active`, `bundle:diag` empty) and the redeploy restores DET state for both tabs in 8 s, with no duplicate menu items reported and no errors in the log. What is still unmeasured is the **negative** control — the same cycle with `removeFromContainer()`'s overlay path and `detachFromOpenTransformations()` reverted, to prove they are what prevents the duplicate *Inspect* items and the double rename/step-change handling. |
| 2 | Unit tests for the four new methods | 1 d | Mock `Spoon`, `TabMapEntry`, `TransGraph`, `XulDomContainer`, `TransMeta`; assert the cold-start/shutdown no-ops, idempotency, and that init/destroy are inverses |
| 3 | Unit test for the `CancellationException` retry | 0.25 d | `RequireJsConfigManagerTest` exists; add cancel-then-retry and assert the result is a complete configuration, not the `"{}; // Error computing RequireJS Config"` fallback |
| 4 | Hot-deploy test matrix | 1 d | deploy / undeploy / redeploy; with and without open tabs; jobs as well as transformations; repeated cycles. Re-capture the bundle count against the committed change set (expect **181**, not 184) |
| 5 | Audit for **other** once-only bindings | 0.5 d | Same family as defects 1 and 3 — e.g. `JobGraph` (`job-graph` category), perspectives, other `TransAfterOpen` consumers |
| 6 | Pentaho Server regression for the RequireJS change | 0.5 d | Shared component |
| 7 | Documentation and PR review | 0.25 d | |
| 8 | ~~**Runtime-verify defect 4**~~ ([§7A](#7a-defect-4--undeploy-aborts-pentaho-pdi-platforms-activation-stale-bundlecontext)) | ✅ **done** | Deploy → undeploy → deploy on 2026-08-18: **0** `MultiException`, **0** `Invalid BundleContext`. What is still worth one run is a `bundle:list` right after a *failed* deploy, to confirm or kill the residual-risk-8 hypothesis |
| 9 | Pentaho Server regression for the `OSGIRuntimeObjectFactory` change | 0.5 d | Shared core; the change only adds a fallback on a path that previously threw |
| 10 | ~~**Runtime-verify defect 5**~~ ([§7B](#7b-defect-5--stale-class-loaders-on-pooled-threads-after-a-redeploy)) | ✅ **done** | Full deploy → undeploy → deploy on 2026-08-18 20:50–20:55 with two transformations open and DET exercised in both tabs after **each** deploy: **0** `Unable to load file step-attributes.xml`, **0** `BundleRevisionImpl.getResourcesLocal`, and in fact **0 exceptions of any kind** in the whole `SpoonDebug.txt` |
| | **Total** | **4–5 engineer-days** | |

### 9.3 What would have made it expensive — and does not apply

| Hypothetical blocker | Finding |
|---|---|
| The Spring context mechanism only works at boot | ❌ It is a plain Blueprint bean of each plugin's own container; it is built whenever Blueprint starts that bundle and closed when it stops |
| Karaf cannot resolve the KAR against a started container | ❌ Resolves in 19–21 s, `bundle:diag` empty. The single-batch rule from [`PDI-20686.md` §6.4](./PDI-20686.md#64-why-nothing-in-the-kar-may-be-prerequisitetrue) is what makes this true in both directions |
| Installing common-ui/Analyzer forces a refresh cascade | ❌ None observed; the 110 pre-existing bundles stayed `Active` |
| Kettle ignores plugins added after startup | ❌ `ExtensionPointMap` and `SpoonPluginManager` are both `PluginTypeListener`s |
| DET's Spoon integration needs `pentaho-kettle` changes | ❌ All required accessors — including the three `remove*Listener` methods — are already public |

---

## 10. Corrections to the first pass

A first pass reached three wrong conclusions. They are recorded here because the *reasons* they were
wrong are reusable lessons, and because anyone reading earlier notes needs to know which parts to
discard.

| # | First-pass claim | Corrected finding | Why it was wrong |
|---|---|---|---|
| 1 | After a hot deploy, an already-open tab is missing the DET items in **both** the right-click menu **and** the Action menu | Only the **right-click** menu is affected; the Action menu works. But the defect also causes a **`NullPointerException` on every step right-click**, which was missed entirely | Relied on a single ambiguous UI observation ("the main menu of the ktr canvas") without disambiguating menus, and never inspected the log for the consequences |
| 2 | `require-init.js` returns **HTTP 500** during the deploy window; the fix was verified by 60 polls at 1 Hz showing "0 bad responses" | It returns **HTTP 200 with a truncated, invalid body**. The verification was **incapable of detecting the defect**, so its "pass" was meaningless | The 500 was **inferred from a stack trace, never measured**. The detector checked only the status code and an error marker — neither of which appears. Sampling at 1 Hz also under-sampled a ~0.5 s window |
| 3 | Two defects total | **Three** — DET state is not restored in already-open transformations | Only menus were exercised in the UI. The state/persistence path was never tested, and was found only because it was reported during this pass |

Two methodological faults produced all three:

* **Process control.** The first-pass teardown sent one `SIGTERM` and continued after 3–4 s. In this
  pass **every** shutdown required `kill -9`, so first-pass runs very likely overlapped with a live
  previous instance — consistent with the observed port drift between 8802/9051 and 8803/9052. Any
  first-pass measurement may have addressed the wrong process.
* **Verification that cannot fail.** Checking a status code for a defect that returns 200 will always
  report success. Where the failure mode is unknown, capture and **validate the artefact itself** —
  here, syntax-checking the served JavaScript.

---

## 11. Residual risks

| # | Risk | Severity | Mitigation |
|---|---|---|---|
| 1 | Other once-only bindings of the same family may remain — jobs (`job-graph`), perspectives, other `TransAfterOpen` consumers | **Medium** | Explicit audit task in [§9.2](#92-what-is-genuinely-left). Transformations are covered and verified; **jobs were not tested** |
| 2 | The two symmetry fixes are **reasoned, not measured** | **Medium** | `removeFromContainer()`'s overlay path and `detachFromOpenTransformations()` only manifest on undeploy→redeploy with a tab open. Task 1 in [§9.2](#92-what-is-genuinely-left) |
| 3 | `syncExec` from the Blueprint init/destroy thread | Low | All four methods marshal onto the SWT thread and **block**. If the UI thread were itself waiting on the OSGi framework this would deadlock the Blueprint container. No such path is known, and `removeFromContainer()` has always done exactly this, so the pattern is pre-existing — but it is worth keeping in mind if a hot deploy ever appears to hang |
| 4 | Validated on one machine and OS (macOS, warm Maven repo, SSD) | Medium | Repeat on the supported OS matrix; the 51 s budget has ~2.5× headroom at 19–21 s |
| 5 | Real transient window: for ~19 s after the drop, DET endpoints return 404. Clicking *Inspect* in that window fails rather than waiting | Low–Medium | Optional: keep DET menu items disabled until the DET REST endpoint answers |
| 6 | The Action-menu re-application path on hot deploy is **not fully traced** ([§5.2](#52-measured-behaviour)) | Low | Trace before narrowing or removing the fix |
| 7 | The RequireJS fix is in a component shared with Pentaho Server | Medium | Change is strictly "retry instead of abort"; still needs Server regression |
| 8 | **A failed hot deploy may poison the container** | **Medium** | Observed while testing the errata: after a deploy that failed (a missing dependency), redeploying a *known-good* KAR in the same JVM still produced `analyzer` 500s (`LocalizationServiceImpl` NPE). Ruled out repeated cycling and the specific errata change by isolation, leaving residual state from the failed attempt as the likely cause. ⚠️ **Inferred by elimination, not directly reproduced.** Practical impact: a PDI restart may be needed after a failed hot deploy. 🔍 **Likely explained by defect 4** ([§7A.3](#7a3-consequence)): `LocalizationServiceImpl` is served by `pentaho-pdi-platform`, the bundle whose activation the stale `BundleContext` aborts — and defect 4's fix is now verified at runtime over two independent deploy→undeploy→deploy cycles ([§8.2](#82-the-undeployredeploy-cycle-verified-end-to-end)), so this should be **re-tested and most likely withdrawn**. See [`PDI-20686-errata.md` §C.6](./PDI-20686-errata.md#c6-new-finding-a-failed-deploy-poisons-the-container-until-pdi-restarts) |
| 9 | ~~Mondrian `DriverManager` registration~~ | **Withdrawn — disproven** | Instrumentation showed the guard `DriverManager.getDriver("jdbc:mondrian:")` **always succeeds** (mondrian auto-registers from PDI's `lib/` via `META-INF/services/java.sql.Driver`), so the registration branch never ran and nothing could leak — confirmed unchanged across a real undeploy→redeploy. The dead code has since been removed from the change set. See [`PDI-20686-errata.md` §C.4](./PDI-20686-errata.md#c4--the-predicted-mondrian-drivermanager-defect-is-disproven) |
| 10 | ~~The mini Spring extender emits no visible log output~~ | **Withdrawn — no longer applicable** | There is no extender. Each plugin's Spring context is a Blueprint bean, so failures surface as ordinary Blueprint container errors |
| 11 | ~~Pivot table `Provider "bundle" not installed`~~ | **Resolved** | Fixed in `ResourceFileManager`; see [`PDI-20686.md` §6.8](./PDI-20686.md#68-plugin-resources-that-are-not-files) and [§11](./PDI-20686.md#11-residual-risks-and-known-issues) risk 4. Verified on cold start; **needs re-verification on a hot deploy**, since the resource is materialized from the bundle the first time it is asked for |
| 12 | **Class-loader leak across cycles** | Low–Medium | Each deploy pins one more `CompositeClassLoader`, and through it a dead bundle revision, to Mondrian's pooled threads ([§7B.2](#7b2-root-cause)). Defect 5's fix makes those pins **harmless**, not absent — nothing releases them short of a PDI restart. Only matters for long sessions with many cycles, i.e. developer machines, not customers |
| 13 | **A KAR present in `deploy/` at startup produces an untracked install that cannot be undeployed** | Low (developer workflow only) | Pre-existing Karaf/PDI boot race caused by `featuresBootAsynchronous=true`, fully diagnosed in [§7C](#7c-not-a-defect--a-kar-present-in-deploy-at-startup-can-never-be-undeployed). **Not** introduced by this change set and **not** fixed by it. Mitigation is procedural: only drop the KAR into a **running** PDI, and remove it from `deploy/` before shutdown. Recovery: clear `system/karaf/caches/` and restart. A real fix (`featuresBootAsynchronous=false` in `pentaho-karaf-assembly`) is deliberately out of scope |

---

## 12. Recommendation

1. **Ship the cold-start solution as PDI-20686** — it is complete, committed and verified.
2. **Add hot deploy as a follow-up scoped at 4–5 days.** All the code is written and compiles; what
   remains is the runtime verification of the two symmetry fixes, unit tests, the once-only-binding
   audit, and cross-product regression ([§9.2](#92-what-is-genuinely-left)).
3. **Ship the `RequireJsConfigManager` fix regardless of the hot-deploy decision.** It is a genuine
   pre-existing bug in a shared component that silently serves corrupt JavaScript, in PDI and in
   Pentaho Server alike.
4. **If hot deploy ships without the DET-side fixes, document the workaround:** after hot-deploying
   DET, close and reopen any transformation that was already open. That single action resolves both
   defect 1 and defect 3 for that transformation.
5. **Do not re-introduce the "cold-start only" framing.** It was assumed, never measured, and the
   committed design is if anything *better* suited to dynamic provisioning than the one it replaced.

---

## 13. Reproducing this analysis

Harness scripts (session folder): `pdi.sh` (process control), `prepare.sh`, `start_pdi.sh`,
`hotdrop.sh`, `hf_probe.sh`, `body_probe.sh`.

```bash
PDI=~/pentaho/pdi/10.2.0.0-SNAPSHOT/20260818/pdi-ee-client-10.2.0.0-20260818.001718-1758-osgi/data-integration

# 0. Build both variants. "fixed" = the current working tree; "nofix" = the same tree with the
#    four files of §9.1 reverted to HEAD. Store them under /tmp/art/{fixed,nofix}/ and record md5s.

# 1. Assert nothing is running, then boot WITHOUT the KAR.
pgrep -f "$PDI"                       # must be empty
prepare.sh nofix hot && start_pdi.sh  # prints Karaf/HTTP ports, PIDs, artifact md5

# 2. Open a transformation in Spoon and LEAVE IT OPEN. This is the tab that exposes defects 1 and 3.

# 3. Drop the KAR and probe.
hotdrop.sh nofix                      # endpoint readiness + log diff
body_probe.sh nofix                   # 10 Hz, saves + syntax-checks every response body

# 4. Assertions
$SHELL_CMD "bundle:list -s" | grep -c Active           # expect 181 (184 pre-simplification)
$SHELL_CMD "bundle:diag"                               # expect empty
grep -c CancellationException $PDI/SpoonDebug.txt      # nofix: >0   fixed: 0
awk -F, '$3==88213 {print $1}' /tmp/art/bodies_nofix/index.csv | while read i; do
  node --check /tmp/art/bodies_nofix/r_$(printf '%04d' $i).js    # nofix: SYNTAX ERROR
done

# 5. In Spoon, in the tab that was ALREADY open:
#      - right-click a step        → nofix: no DET items + NPE in the log;  fixed: items, no NPE
#      - open DET                  → nofix: empty state, no save prompt;    fixed: state restored
#      - close and reopen the .ktr → fixes both, even without the code fix (the workaround)

# 6. NOT YET RUN - the undeploy/redeploy UI cycle (see §9.2 task 1)
#      - with a transformation open, remove the KAR, wait for 404, drop it back in
#      - expect exactly one set of "Inspect" items in the Action menu, and each rename
#        or step-name change handled exactly once
```

**Checking whether an install is actually tracked** (the [§7C](#7c-not-a-defect--a-kar-present-in-deploy-at-startup-can-never-be-undeployed)
trap). Run this *before* trusting any undeploy result — it needs no running shell, only the files:

```bash
export DATA=$PDI/system/karaf/caches/spoon/data-1
python3 - <<'EOF'
import json, glob, os, time
data = os.environ['DATA']
managed = set(json.load( open(f'{data}/cache/bundle19/data/state.json') )['managed']['root'])
for info in glob.glob(f'{data}/cache/bundle*/bundle.info'):
    bid, loc = open(info).read().splitlines()[:2]
    # PDI's webpackage deployer installs its bundles programmatically, not via a feature,
    # so exactly one 'pentaho-webpackage:jardir:' bundle is legitimately unmanaged.
    if int(bid) not in managed and int(bid) >= 100 and not loc.startswith('pentaho-webpackage:jardir:'):
        print(int(bid), time.strftime('%H:%M:%S', time.localtime(os.path.getmtime(os.path.dirname(info)))), loc[:90])
EOF
```

Bundle 19 is `org.hitachivantara.karaf.features.core`, so `state.json` is the FeaturesService's own
state. **Any output means orphaned bundles** — they are running but the FeaturesService does not own
them, and removing the KAR will not uninstall them. A correctly hot-deployed KAR prints nothing.

> **Note on the Karaf shell.** `$SHELL_CMD` above stands for whatever console access is available.
> The Karaf SSH console does **not** start out of the box in the 10.2.0.x client assemblies, but it
> can be enabled: add `ssh` to `featuresBoot` in `etc/org.apache.karaf.features.cfg` and
> `org.apache.sshd.*` to `org.osgi.framework.bootdelegation` in `etc/config.properties` (see
> [`SSH.md`](./SSH.md)). With that in place `sshpass -p karaf ssh -p 8802 karaf@127.0.0.1
> -oHostKeyAlgorithms=+ssh-rsa` works and gives `kar:list`, `feature:list`, `bundle:list` and
> `bundle:diag` — which is what produced the evidence in
> [§7C](#7c-not-a-defect--a-kar-present-in-deploy-at-startup-can-never-be-undeployed). All the
> decisive evidence elsewhere in this document (endpoint status, response bodies, log contents) is
> reachable over HTTP and the log file alone. **Use `127.0.0.1`, never `localhost`** — Jetty's
> `IPAccessHandler` rejects the IPv6 loopback with an HTTP 500.

**Excluded from log scanning** (expected, not defects): `AnalyzerContentGenerator … No controller
mapping for: /<name>` — generated by the health-check GET itself ([§8.1](#81-one-log-message-that-is-not-a-defect)).

---

## 14. Change log

| Date | Change |
|---|---|
| 2026-08-18 (rev 9) | **Defect 5 verified at runtime; the full un/redeploy cycle is now clean end to end — new [§8.2](#82-the-undeployredeploy-cycle-verified-end-to-end).** Fresh run with a cleared Karaf cache, an empty `deploy/` and both patches in place, two transformations open before the KAR arrived and DET exercised in both tabs after each deploy: deploy **16 s**, undeploy **13 s**, redeploy **12 s**; `require-init.js` 180 969 → 93 534 → 180 969 B; `${karaf.data}/kar/` correctly emptied on undeploy. `SpoonDebug.txt` (712 lines): **0** `step-attributes.xml` traces (10 before), **0** `MultiException`/`Invalid BundleContext`, **0** `CancellationException`, **0** unresolved requirements, **0 exceptions of any kind** — the first fully silent cycle. DET state after the redeploy: **9** and **8** step states, matching rev 5, so defect 3's fix survives the cycle. The loaded class was confirmed to be the patched `CompositeClassLoader`. Tasks 8 and 10 of [§9.2](#92-what-is-genuinely-left) closed; residual risk 8 downgraded to "re-test and most likely withdraw". The [§13](#13-reproducing-this-analysis) orphan detector now skips the one `pentaho-webpackage:jardir:` bundle, which PDI's webpackage deployer installs programmatically rather than through a feature. |
| 2026-08-18 (rev 8) | **"Hot undeploy stopped working" investigated and closed as *not a defect* — [§7C](#7c-not-a-defect--a-kar-present-in-deploy-at-startup-can-never-be-undeployed), new residual risk 13.** Symptom: the KAR was removed, PDI kept behaving as if DET were deployed, and `SpoonDebug.txt` was completely clean. Probing the live instance showed `kar:list` empty, no DET feature repository, no DET features installed and `bundle:diag` empty — yet all DET bundles `Active` and every endpoint 200. The Felix bundle cache dates **all 75 KAR bundles to 19:59:51**, 11 s after Karaf launched and 23 s before boot provisioning finished, and the FeaturesService `state.json` lists 213 managed bundles of which **none** are those 75. Cause: `featuresBootAsynchronous=true` means a KAR sitting in `deploy/` at startup is installed concurrently with boot provisioning, whose `saveState` then overwrites the KAR's bookkeeping; `KarServiceImpl.uninstall` only uninstalls features whose repository is *currently registered*, so it degrades to `deleteRecursively( karDir )` and nothing else. **Not caused by the change set and not fixed by it** — mitigation is procedural (only drop the KAR into a running PDI; empty `deploy/` before any restart; recover by clearing `system/karaf/caches/`). Warning added to [§3.3](#33-procedure) and a row to [§0](#0-how-to-read-this-document). |
| 2026-08-18 (rev 7) | **Defect 4's fix verified at runtime, and defect 5 found on the same run ([§7B](#7b-defect-5--stale-class-loaders-on-pooled-threads-after-a-redeploy)).** A full deploy → undeploy → deploy cycle with the patched `pentaho-platform-core`: **0** `MultiException`, **0** `Invalid BundleContext` — defect 4 closed at the OSGi layer, `pentaho-pdi-platform` survives the undeploy. The same run surfaced **10** `Unable to load file step-attributes.xml` traces, all after the redeploy (19:40) and none after the first deploy (19:37–19:38), from an `NullPointerException` in `BundleRevisionImpl.getResourcesLocal`: Mondrian's `SegmentLoader` pool threads inherit the `ContentGeneratorServlet` composite class loader and keep it across the un/redeploy, so `ServiceLoader` lookups run against a **disposed bundle revision**. Functionally harmless (`BaseStepMeta` swallows it and DET worked in both tabs) but it is also a per-cycle class-loader leak. Fixed defensively in `CompositeClassLoader` (skip a dead delegate, let the other answer — which additionally makes the parse *succeed*), 6 new tests, `pentaho-pdi-platform` green at 75. Effort unchanged at 4–5 d; verification task 10 added to [§9.2](#92-what-is-genuinely-left). |
| 2026-08-18 (rev 6) | **Defect 4 added ([§7A](#7a-defect-4--undeploy-aborts-pentaho-pdi-platforms-activation-stale-bundlecontext)): removing the KAR aborts `pentaho-pdi-platform`'s activation.** Observed on a run where DET worked in both open transformations and the KAR was then removed: `MultiException: Error restarting bundles … Activator start error in bundle pentaho-pdi-platform [201] … IllegalStateException: Invalid BundleContext`. Root cause traced to the **single static `BundleContext`** `PentahoSystem` receives from `pentaho-blueprint-activators`, which nothing invalidates when that bundle is refreshed — so `PdiPlatformActivator.start()` registers against a dead context and Felix leaves bundle 201 `Resolved`, taking `WebContextServlet`, `LocalizationServlet` and `XmiToDatabaseMetaDatasourceService` with it. Fixed in `pentaho-platform` (branch `PDI-20686`, uncommitted) by treating an invalidated context like "OSGi not available yet" and reusing the existing deferral, which republishes on the next `setBundleContext()` — self-healing, no restart. 5 new unit tests; `pentaho-platform-core` green at 602. Effort 3–4 d → **4–5 d** (new verification task 8 and a Server regression task 9 in [§9.2](#92-what-is-genuinely-left)). Flagged as the likely explanation of residual risk 8 / errata §C.6. ⚠️ Runtime verification of the fix still pending. |
| 2026-08-18 (rev 5) | **Hot deploy re-verified on the current assembly, and the undeploy→redeploy cycle exercised at the UI layer for the first time.** Environment moved to `pdi-ee-client-10.2.0.0-20260818.001718-1758-osgi` (§3.1); the separate `…-osgi_cold` install is retired. ✅ Drop with two transformations already open: ready in **21 s**, **181** bundles `Active` (the committed figure, confirming the 184 in §4.1 was pre-simplification), `bundle:diag` empty, `require-init.js` **180 969 B** and valid, all Analyzer / geo / DET REST endpoints 200, and **DET state restored for both already-open transformations** (9 and 8 step states in `/cxf/det/core/persistence/states`) — defect 3's fix confirmed against a genuine first-time hot deploy. ✅ Undeploy → 181→113 `Active`, DET 404, `require-init.js` back to 93 534 B, `bundle:diag` empty; **redeploy** → ready in **8 s**, back to 181, state restored again — which closes remaining-work item 1 of §9.2 at the OSGi layer and exercises the `destroy-method` path that was previously reasoned from code only. ✅ `SpoonDebug.txt` for the whole run: **0** `CancellationException`, **0** `require-init` WARN, **0** NPE, **0** unresolved requirements; the only `ERROR`s are the probe-induced `AnalyzerContentGenerator … No controller mapping` of [`PDI-20686.md` §10.1](./PDI-20686.md#101-expected-log-noise--none-of-these-are-regressions), 1:1 with the requests made. Added the delivery note that `requirejs-manager-impl` must be copied into the install (step 2b of [`PDI-20686.md` §9](./PDI-20686.md#9-deploy)) for defect 2's fix to be under test at all. |
| 2026-08-18 (rev 4) | **Review pass over the uncommitted changes, with unit tests.** Three defects fixed in `DetMenuPlugin`: the single `transContainer` field kept only the *last* open transformation tab, so an undeploy left the other tabs' overlays, handlers and class-loader registrations behind and a redeploy duplicated them (now a list); `removeFromContainer()` had no display guard and would throw `SWTException` out of the Blueprint destroy-method on every PDI shutdown; and the guard plus `Display.syncExec` are now the `protected` seams `isUiUp( Spoon )` / `onUiThread( Spoon, Runnable )`, shared by all four methods — `TransformationAfterOpenExtensionPoint` gets the same seams plus `getSpoon()`. In `RequireJsConfigManager`, three cancellations in a row no longer serve an unattributable "unknown error" ([§6.3](#63-the-fix)). New `DetMenuPluginTest` (23 tests) and 7 more in `TransformationAfterOpenExtensionPointTest` — `det-impl-pdi` is green at 57 tests; `RequireJsConfigManagerTest` gains 3, the module green at 183. |
| 2026-08-13 (rev 3) | **Promoted to the source of truth for hot deploy.** Restructured around what remains rather than around the investigation: added [§0 How to read](#0-how-to-read-this-document) and a real [§9 Remaining work](#9-remaining-work); removed the "these fixes have been reverted" framing — they are **uncommitted changes in `pentaho-det-ee` and `pentaho-osgi-bundles`**, and they compile with their suites green. Added two **symmetry fixes** found while completing the work: the `removeFromContainer()` overlay-path/try-scope correction ([§5.3](#53-the-fix)) and `detachFromOpenTransformations()` ([§7.4](#74-the-fix)), both needed for undeploy→**re**deploy and both flagged as reasoned-not-measured. Corrected [§6.3](#63-the-fix) to the fix as actually written (it also evicts the cancelled future, and deliberately does *not* re-invalidate). Withdrew the "extender only works at boot" and "extender has no logging" risks — there is no extender. Marked every 184-bundle figure as pre-simplification (the committed set is 181) and removed the remaining SSH-specific commands. Effort revised 4–6 d → **3–4 d**. |
| 2026-08-11 (rev 2) | Withdrew the Mondrian `DriverManager` risk — **disproven** by instrumentation, and the dead code removed from the change set. |
| 2026-08-11 | Annotated for the cold-start simplification (errata A1 + B5 + C1/C3/C7/C8b). Conclusions unchanged; the three defects and their fixes are unaffected, but the measurements predate the simplification. |
| 2026-08-07 (2nd pass) | **Rewritten after a controlled re-run.** Corrected defect 1 (Action menu is fine; NPE found), corrected defect 2 (HTTP 200 truncated body, not 500; earlier verification was incapable of detecting it), added defect 3 (DET state not restored in already-open transformations) with its fix and a no-code workaround, added the cold-start control, identified the Analyzer log message as a probe artifact, hardened process control. See [§10](#10-corrections-to-the-first-pass). |
| 2026-08-07 (1st pass) | Initial version. Superseded. |
