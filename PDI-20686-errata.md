# PDI-20686 — Errata and simplification options

> **Companion to:** [`PDI-20686.md`](./PDI-20686.md) (the cold-start solution).
> **Purpose:** find the most pragmatic, lowest-impact shape of the change set that still makes DET work.
> **Status:** ✅ **APPLIED and since COMMITTED.** Options **A1, B5, C1, C3, C6, C7, C8b** were applied,
> built and verified by cold start plus a full functional pass. See [§I](#i-what-was-applied) for the
> outcome, including one regression B5 caused and how it was fixed. `PDI-20686.md` has been updated to
> describe the resulting design and is the source of truth; this document is kept as the record of *why*
> each simplification was or was not taken. C4, C5, C8a were kept (proven required); C2 was reverted.

| Marker | Meaning |
|---|---|
| ✅ **Proven** | Measured in this environment during this pass — build + cold start + endpoint checks. |
| 📄 **Sourced** | Traced to a specific file, manifest or commit. |
| 🔍 **Proposed** | Design option, technically analysed but **not yet built or run**. |

### Hot-deploy impact markers

Every item is additionally marked for its effect on the hot-deploy work in
[`PDI-20686_hot.md`](./PDI-20686_hot.md). **[§E](#e-hot-deploy-impact-of-every-errata-item) collects
them in one table** — read that before adopting anything.

| Marker | Meaning |
|---|---|
| 🟢 **HOT-SAFE** | No effect on hot deploy, or measured safe under hot deploy. |
| 🟡 **HOT-CHECK** | Plausibly safe, but only cold start was measured. Re-test under hot deploy before adopting. |
| 🔴 **HOT-RISK** | Would remove or weaken something that hot deploy relies on more than cold start does. |

---

## 0. Executive summary

Two findings materially shrink the change set. The first removes a module outright and, with it, the
team's blocker.

| # | Topic | Current | Recommendation | Impact |
|---|---|---|---|---|
| **A** | `ILineageClient` stub in `pentaho-det-ee` | New module `det/impls/lineage-stub` | ✅ **Delete it. It is not needed at all.** 🟢 | −1 module, −1 bundle, −1 repo concern. **Blocker disappears — no relocation debate needed.** |
| **B** | Mini Spring extender | New module + new bundle in the KAR | 🔍 **Replace with a Blueprint factory bean in `pentaho-pdi-platform`** (no extender, no new bundle) — or, if that is too invasive, merge it into the platform-plugin deployer 🟡 | −1 module, −1 bundle, removes the 5 s Blueprint race entirely |
| **C** | Assorted | 8 further candidates | **All now measured.** ✅ 6 removable (C1, C2, C3, C6, C7, C8b) · 🔴 3 proven **required** (C4, C5, C8a) | Smaller diff, fewer questions at review |

**If A and B are both adopted, `pentaho-osgi-bundles` and `pentaho-det-ee` each lose a whole new
module, and the DET KAR feature loses two `<bundle>` lines.**

> ⚠️ **Testing overturned four of the first revision's assessments** — two over-claimed as risky (C3,
> C7 — both actually safe), one under-claimed (C5 — actually **required**, its removal breaks the pivot
> table), and one predicted defect **disproven** (the Mondrian `DriverManager` leak — the code could
> never run). Only **C4** survived as a genuine hot-deploy dependency. See
> [§C.5](#c5-what-the-testing-changed).

---

## A. The `ILineageClient` stub — delete it 🟢 HOT-SAFE (measured)

### A.1 What `PDI-20686.md` says today, and why it is wrong

> §6.3: *"✅ Verified: **no bundle publishes that service** in either 10.2.0.1 or 10.2.0.0-SNAPSHOT —
> metaverse ships as a Kettle plugin (`plugins/metaverse-plugin/`), not as an OSGi bundle."*

The premise was tested by scanning `system/karaf/system/**/*.jar` for a Blueprint that publishes
`ILineageClient`. That search returns nothing — but it was **the wrong search**, because the service is
not published by a Blueprint at all.

### A.2 What actually happens ✅ Proven

Booting the 10.2.0.X test install **with no DET KAR at all** and querying the live service registry:

```
karaf@root()> service:list org.pentaho.metaverse.api.ILineageClient
[org.pentaho.metaverse.api.ILineageClient]
 service.bundleid = 105
 Provided by : Bundle 105          →  pentaho-blueprint-activators

karaf@root()> bundle:services 206   (pentaho.pdi-dataservice-server-plugin)
 ...
 [org.pentaho.di.trans.dataservice.IDataServiceMetaFactory]      ← container BUILT
```

`pentaho-blueprint-activators` runs `org.pentaho.platform.osgi.PentahoOSGIActivator`, which bridges
objects registered in `PentahoSystem` — including the **real** `LineageClient` from the metaverse
Kettle plugin — into the OSGi service registry. So:

* `ILineageClient` **is** available in a stock PDI client;
* `pdi-dataservice-server-plugin`'s Blueprint container **does** build and **does** publish
  `IDataServiceMetaFactory`;
* the chain `IDataServiceMetaFactory → IGeoService/IGeoStorage → content/geojson` and DET's own
  `det-core` / `det-pdi` containers was **never blocked** by a missing lineage service.

### A.3 Confirmation by removal ✅ Proven

The `det-impl-lineage-stub` bundle and its dependency were removed from the KAR, rebuilt, and
cold-started:

| Metric | With stub | **Without stub** |
|---|---|---|
| Active bundles | 184 | **183** (exactly the stub) |
| `bundle:diag` | empty | **empty** |
| Errors in `SpoonDebug.txt` | 0 (79 lines) | **0 (79 lines)** |
| `require-init.js` | 180 969 B | **180 969 B** |
| `/cxf/det/core/data/datasources` | 200 | **200** |
| `/cxf/det/core/persistence/states` | 200 | **200** |
| `/content/analyzer/service` | 200 | **200** |
| `/content/geojson` | 200 | **200** |
| DET + Model view in Spoon | works | **works** (user-confirmed) |

### A.4 Hot deploy: measured, and safer than cold ✅ Proven 🟢

The stub-free KAR was also exercised against a **running** PDI (booted with an empty `deploy/`):

| Scenario | Result |
|---|---|
| Hot deploy of the stub-free KAR | **ready at T+12 s** (budget 51 s); 183 active bundles, no `lineage` bundle, `bundle:diag` empty |
| Undeploy | 183 → 113 bundles, DET endpoints 404 — clean unwind |
| **Redeploy** | **ready at T+16 s**; all endpoints 200, `require-init.js` 180 969 B, 0 `CancellationException` |
| `service:list ILineageClient` throughout | still provided by **bundle 105** (`pentaho-blueprint-activators`) |

📄 `pentaho-blueprint-activators` is a **boot** bundle, so on hot deploy the real `ILineageClient` is
*guaranteed* to be registered before the KAR is even dropped. Deleting the stub is therefore at least
as safe under hot deploy as under cold start.

> **Harness note.** The first attempt at this test showed DET never coming up (404 for 92 s, with
> *zero* log errors). That signature — nothing happens at all — was a **harness failure**: the KAR
> source directory had been cleaned, so `cp` silently copied nothing into `deploy/`. `hotdrop.sh` now
> asserts the artifact exists and prints the deployed file. Recorded because "404 with an empty log"
> is easy to misread as a product failure.

### A.5 Why the stub appeared to be necessary

Most likely a **misattributed symptom**. Early in the investigation the DET containers genuinely were
not coming up — but that was caused by the `prerequisite="true"` feature problem
([`PDI-20686.md` §6.4](./PDI-20686.md)), which starved the Karaf feature installer and left features
`Uninstalled`. The stub was added around the same time and was never re-validated after the feature
fix landed. Because the stub is registered at `Integer.MIN_VALUE` ranking, the *real* provider always
won anyway — so with both present, the stub was already dead code at runtime.

### A.6 Options

| Option | Description | Repos touched | Verdict |
|---|---|---|---|
| **A1 — Delete the module** ⭐ 🟢 | Remove `det/impls/lineage-stub`, its `<module>` entry, the KAR `<dependency>` and the `<bundle>` line. | `pentaho-det-ee` only (a **removal**) | ✅ **Recommended.** Proven. Resolves the blocker without any relocation. |
| A2 — Relocate to `pentaho-platform-plugin-geo` 🟢 | Add the no-op class + `<service>` to the existing `dataservice-geo-impl` blueprint (the first consumer in the chain). Geo already hosts DET-specific modules (`pdi-geo`, `pentaho-geo-visual-map`). | `pentaho-platform-plugin-geo` | Only if A1 is rejected. Semantically odd — geo publishing a metaverse service. |
| A3 — Relocate to `pentaho-osgi-bundles` 🟡 | Neutral shared home, repo already in the delivery. | `pentaho-osgi-bundles` | Only if A1 is rejected. Adds a bundle to a shared repo for a problem that does not exist. |
| A4 — Fix `pdi-dataservice-server-plugin` 🟡 | Make the reference `availability="optional"`, or ship a default no-op there. Correct ownership. | `pdi-dataservice-server-plugin` — **not cloned locally** | Unnecessary given A.2. Would also need the repo added. |
| A5 — Install the real `pentaho-metaverse-core` as a bundle 🔴 | It already contains a Blueprint that publishes `ILineageClient`. | PDI assembly | ❌ **Infeasible.** 📄 Its `MANIFEST.MF` has **no OSGi headers at all** (no `Bundle-SymbolicName`, no `Import-Package`) — it is a plain jar. It would need `wrap:`, plus its 673-line Blueprint with 16 `reference-list`s and a 2.4 MB `lib/` folder. Far more risk than the problem warrants. |

> **Only `ILineageClient` was audited this way.** DET's other cross-bundle references
> (`IDataServiceMetaFactory`, `PentahoCacheManager`, `CacheService`, …) were not individually
> re-verified. They are all satisfied in the working runs, so nothing is *missing* — but if any other
> element of the change set was added on the same "nothing provides this" reasoning, it deserves the
> same `service:list` check.

---

## B. The mini Spring extender — where it should live, or whether it should exist

### B.1 The constraint that cannot be negotiated away 📄

`paz-plugin-ce` ships its own Blueprint, and it hard-requires a Spring context as an OSGi service:

```xml
<!-- analyzer/OSGI-INF/blueprint/analyzer_beans.xml, shipped inside paz-plugin-ce -->
<reference id="spring" interface="org.springframework.context.ApplicationContext"
           filter="(Bundle-SymbolicName=analyzer)" availability="mandatory" timeout="5000"/>

<bean class="com.pentaho.analyzer.content.AnalyzerLifecycleListener" init-method="init">
    <argument ref="spring"/>
</bean>
```

and `SpringFileHandler` generates one servlet per content-generator bean that also uses it:

```java
argument.setAttribute( "ref", "spring" );   // → ContentGeneratorServlet(applicationContext, beanId)
```

So **something must produce a Spring `ApplicationContext` for the analyzer bundle.** Spring DM used to;
it was removed by SP-6858. The only real question is *what shape* the replacement takes.

### B.2 Options

#### B1 — Status quo: standalone `pentaho-mini-spring-extender`, listed by the KAR — 🟢 HOT-SAFE (already measured)

* New module in `pentaho-osgi-bundles`, new bundle in the DET KAR feature.
* Generic: tracks **every** `ACTIVE` bundle containing `META-INF/spring/*.xml`.
* Keeps the cross-bundle service handshake, so the analyzer's 5 s `timeout` remains a (measured
  comfortable, but real) constraint.
* **Cost:** 1 new module, 1 new bundle, 1 KAR line, plus the "why is there a new extender?" review
  conversation.
* **Hot deploy 🟢:** this is the configuration measured in `PDI-20686_hot.md` — the extender is
  installed *inside the KAR batch*, becomes `ACTIVE`, and builds the analyzer context within the 5 s
  window. It is also removed with the KAR, so undeploy tears its contexts down trivially.

#### B2 — Merge into `pentaho-platform-plugin-deployer` 🔍 🟡 HOT-CHECK

The deployer is what *generates* the Blueprint that needs the context, so it is the natural owner.

* Move the 3 classes (361 lines) into the existing deployer bundle as `Private-Package`; start the
  tracker from the deployer's existing Blueprint via `init-method` / `destroy-method` (it uses
  Blueprint, not an activator — no `Bundle-Activator` needed).
* 📄 The deployer already ships in the **`pentaho-deployers`** feature, which the DET KAR **already
  declares** → the KAR line disappears, no new feature, no new artifact.
* ⚠️ **Risk:** the deployer would gain `org.springframework.*` imports. It is a boot bundle for
  Pentaho Server as well; if Spring were ever absent there, the deployer would fail to resolve and
  *all* platform-plugin deployment would break. Mitigate with `resolution:=optional` imports and a
  guarded start, and regression-test Server.
* **Hot deploy 🟡 — changes the lifecycle in both a good and a risky way.**
  *Better:* the extender becomes a **boot** bundle, already `ACTIVE` when the KAR lands, so the
  analyzer's 5 s timeout no longer races against the extender's own installation.
  *Riskier:* today the extender **dies with the KAR**, so undeploy disposes its Spring contexts for
  free. As a boot bundle it **outlives** the KAR, which makes `BundleTracker.removedBundle()` →
  `context.close()` load-bearing on every undeploy/redeploy cycle. That path exists in the code but
  has never been exercised in this configuration. **Must be soak-tested over repeated
  deploy/undeploy/redeploy cycles before adoption.**

#### B3 — Keep the bundle, add it to the `pentaho-deployers` feature 🔍 🟡 HOT-CHECK

* Same "available wherever the deployer is" benefit, without changing the deployer bundle's imports.
* 📄 Requires editing `pentaho-karaf-features-standard` in **`pentaho-karaf-assembly`** — which adds a
  sixth repository to the delivery.
* **Hot deploy 🟡:** identical lifecycle consequences to B2 — the extender becomes long-lived and must
  correctly dispose contexts on undeploy. Same soak test required.

#### B4 — Revive the existing `pentaho-spring-dm-extender` module 🔍 🟢 HOT-SAFE

📄 That module **still exists in the `pentaho-osgi-bundles` tree** (`PentahoOsgiBundleXmlApplicationContext.java`,
`ApplicationContextCreator.java`) but was dropped from `<modules>` by commit `e5898ab918`
(**SP-6858**, the Spring 3.2.18 CVE removal) — the same commit documented in
[`PDI-20686.md` §2.1](./PDI-20686.md) as timeline item #5. It is currently orphaned dead code.

* Re-enable that module with a Spring-DM-free implementation instead of adding a new one.
* Presents to reviewers as *"SP-6858 disabled this; here it is restored without the CVE-bearing
  dependency"* rather than *"here is a new extender"* — a much easier conversation.
* Also deletes orphaned dead code.
* ⚠️ The artifactId would be misleading (it is no longer Spring DM). Consider
  `pentaho-spring-extender`, accepting that this is then a rename rather than a revival.
* **Hot deploy 🟢:** behaviourally identical to B1 — same bundle, same KAR-scoped lifecycle, only the
  coordinates change. Nothing to re-test beyond a smoke run.

#### B5 — No extender at all: a Blueprint factory bean ⭐ 🔍 🟡 HOT-CHECK (likely an improvement)

The strongest simplification. Instead of a **bundle that watches other bundles**, let the plugin's
**own** Blueprint container build its Spring context.

📄 Aries Blueprint predefines the component `blueprintBundle` (an `org.osgi.framework.Bundle`) — already
used elsewhere in this product (`pentaho-blueprint-activators` uses the sibling
`blueprintBundleContext`). So the analyzer's own Blueprint can do:

```xml
<!-- analyzer/OSGI-INF/blueprint/analyzer_beans.xml -->
<bean id="spring" class="org.pentaho.platform.pdi.SpringContextFactory"
      factory-method="createForBundle" destroy-method="close">
  <argument ref="blueprintBundle"/>
</bean>

<bean class="com.pentaho.analyzer.content.AnalyzerLifecycleListener" init-method="init">
  <argument ref="spring"/>
</bean>
```

`SpringContextFactory` is the existing `BundleApplicationContextFactory` + `CompositeClassLoader`,
moved into **`pentaho-pdi-platform`** — a bundle that the generated Blueprint **already depends on**
for `org.pentaho.platform.pdi.ContentGeneratorServlet`, and whose `pdi-platform` feature the DET KAR
**already declares**.

| Consequence | Effect |
|---|---|
| `pentaho-mini-spring-extender` module | **deleted** |
| New bundle in the KAR feature | **none** |
| New feature dependency | **none** |
| Analyzer's 5 s `timeout` race | **gone** — no cross-bundle service handshake; the context is a local bean |
| Lifecycle | Context is created and closed *with* the plugin's own container — strictly correct, no `BundleTracker` |
| Scope | Only bundles that ask for it get a context, instead of every bundle with `META-INF/spring/*.xml` |
| Pentaho Server blast radius | **none** — no shared bundle changes behaviour |

⚠️ **Costs and risks (not yet built or run):**
1. Requires editing `analyzer_beans.xml` **inside `pentaho-analyzer`**. We already modify that repo for
   the 4 content-generator beans, so the marginal cost is one more file — but it does mean the analyzer
   plugin zip must ship the change (same delivery path as the beans, see
   [`PDI-20686.md` §6.1](./PDI-20686.md)).
2. `SpringFileHandler` currently emits `ref="spring"` assuming that id exists. Under B5 it still does —
   satisfied by the factory bean instead of a service reference. Any *other* platform plugin deployed
   this way would need the same bean; `SpringFileHandler` could **generate** it when the plugin's own
   Blueprint does not already define `spring`, which would make the mechanism self-contained.
3. The `LinkageError` constraint documented in [`PDI-20686.md` §6.2](./PDI-20686.md) still applies —
   the factory must keep using `addProtocolResolver(...)` and must not subclass the context.
4. **Unverified.** B5 is a design, not a measurement. It should be prototyped and cold-started before
   being adopted.

**Hot deploy 🟡 — analysed as an improvement, but unmeasured:**

| Aspect | Effect under hot deploy |
|---|---|
| Context lifecycle | 🟢 **Strictly better.** The context becomes a bean of the analyzer's *own* Blueprint container, with `destroy-method="close"`. Undeploy destroys the container → context closed. Redeploy → fresh container, fresh context. No long-lived tracker to get wrong (contrast B2/B3). |
| The 5 s `timeout` | 🟢 **Eliminated.** No cross-bundle service handshake at all, so the one timing constraint that hot deploy shares with cold start disappears. |
| Blueprint id resolution | 🟢 Aries merges **all** `OSGI-INF/blueprint/*.xml` in a bundle into one container, so the deployer-generated servlets and the `spring` bean in `analyzer_beans.xml` share a namespace — the same reason the current design works. |
| Container startup | 🟡 The Spring context is now built **synchronously during Blueprint container startup** instead of afterwards by a tracker. If context creation is slow this delays the container rather than running in parallel. Measured context build is sub-second, so this is expected to be immaterial. |
| ⚠️ Verification | Must be exercised over **deploy → undeploy → redeploy**, not just a single deploy, because the whole point is the changed disposal path. |

### B.3 Recommendation

**B5** if there is appetite for one more `pentaho-analyzer` file and a prototype cycle — it is the only
option that removes the component rather than moving it, and it also removes the 5 s race.
**B2** as the pragmatic fallback: it eliminates the new module and the KAR line with no analyzer change,
at the cost of Spring imports on a Server-shared bundle.
**B4** if the priority is minimising *reviewer* friction rather than component count.

---

## C. Other simplification candidates — **all now measured**

> Every candidate below has been **built and run**, cold start and/or hot deploy, since the first
> revision of this document. Four of the original assessments turned out to be wrong — two in each
> direction. See [§C.5](#c5-what-the-testing-changed) for the scorecard.

### C.1 ✅ Proven removable

| # | Item | Evidence | Hot deploy |
|---|---|---|---|
| **C1** | `<bundle>mvn:org.mozilla/rhino/1.7.13</bundle>` in the DET KAR feature | Removed and rebuilt. `rhino 1.7.13` is **still present** (`org.mozilla.rhino`, pulled in by `pentaho-cpf-pdi`), all endpoints 200, clean log. Redundant. | 🟢 **HOT-SAFE — measured.** Confirmed on a hot deploy as well: `bundle:list` still shows `org.mozilla.rhino` at 183 active bundles. |
| **C2** | `nodejs.version` v18.20.4 → v18.20.2 in `pentaho-analyzer/client/pom.xml` | Local-build-only hack ([`PDI-20686.md` §5.1](./PDI-20686.md)). Must not ship. | 🟢 **HOT-SAFE.** Build-time only. |
| **C3** | `cxfBus` reference + `bus="cxfBus"` on 5 `jaxrs:server` endpoints | ✅ **Removed from all 5 endpoints and soaked: 3 cold starts + 3 undeploy/redeploy cycles. Zero `No DestinationFactory` errors, all DET REST endpoints 200 every time, clean logs.** | 🟢 **HOT-SAFE — measured.** My earlier 🔴 was **over-claimed** (see §C.5). |
| **C7** | `pdi-data-refinery` → plain `<feature>` (drop `dependency="true"`) | ✅ **Reverted and hot-deployed. No pre-existing bundle changed id or state (no refresh cascade), all endpoints 200, and on undeploy `pdi-data-refinery` stayed `Started` — it was not uninstalled.** | 🟢 **HOT-SAFE — measured.** My earlier 🔴 was **over-claimed** (see §C.5). Only observable difference: the feature stays flagged `Required` after undeploy — a cosmetic state residue. |
| **C8b** | `wrap:mvn:pentaho/pentaho-connections/…` | ✅ **Removed in isolation on a fresh JVM: deploy + undeploy + redeploy, analyzer 200 throughout, 183 → 182 bundles.** User-confirmed functionally: Geo, Model view and pivot table all worked. | 🟢 **HOT-SAFE — measured.** |

### C.2 🔴 Proven **required** — do not remove

| # | Item | Evidence |
|---|---|---|
| **C4** | `PdiPlatformActivator` — `removeProvider("mtm")` before `addProvider` | ✅ **Directly proven.** Baseline (fix present): `bundle:restart` of `pentaho-pdi-platform` → **0** mtm messages. Fix reverted, then `bundle:update <id> file:///…jar` (a real refresh path) → <br>`ERROR [PdiPlatformActivator] There is already a vfs provider registered for scheme mtm`<br>`org.apache.commons.vfs2.FileSystemException: Multiple providers registered for URL scheme "mtm".`<br>📄 `PdiPlatformActivator.stop()` is **empty**, so the provider survives the restart and the re-registration is rejected — leaving the *old* provider, bound to an invalidated classloader. <br>⚠️ **Scope note:** a DET KAR deploy/undeploy/redeploy cycle does **not** refresh `pentaho-pdi-platform` (0 mtm messages observed), so this protects other refresh scenarios — patching `pdi-platform` itself, or any bundle change that re-wires it — not the DET KAR cycle specifically. |
| **C5** | `WebjarsURLConnection` version fallback | ✅ **Proven required — and it took a human to catch it.** With the fallback removed, every `org.webjars.bower/datatables.net*` resource **404s**, which breaks the DET pivot table:<br>`/datatables.net@1.10.12/js/jquery.dataTables.js` → **404**<br>`/datatables.net-scroller@1.4.2/js/dataTables.scroller.js` → **404**<br>`/datatables.net-fixedheader@3.1.1/js/dataTables.fixedHeader.js` → **404**<br>With the fallback restored, all three return **200**.<br>⚠️ **My automated check passed while the feature was broken**: `require-init.js` was still 200, still 180 969 bytes, still valid JavaScript, with **0 null versions**. The configuration *looked* correct; only the mapped resources 404'd. This was found because a pivot table was exercised in the UI. |
| **C8a** | `<bundle>mvn:com.ibm.icu/icu4j/63.1</bundle>` | ✅ **Proven required.** Removed → the analyzer Spring context fails to build:<br>`ERROR [MiniSpringExtender] failed to build ApplicationContext for bundle 'analyzer'`<br>`Caused by: java.lang.NoClassDefFoundError: com/ibm/icu/text/DateFormat`<br>`/content/analyzer/*` → 404. Nothing else supplies `com.ibm.icu`. |

### C.3 ✅ C6 — `ContentGeneratorServlet` rewrite, bisected

The rewrite was decomposed into three hunks and each was reverted independently on a clean-cache install
(`…-osgi`), with a functional pass in Spoon after each.

| Hunk | Verdict | Evidence |
|---|---|---|
| **(a) `CompositeClassLoader` as the TCCL** | 🟡 **No measurable effect — but keep** | Reverted to the plain plugin class loader: all endpoints 200, log clean, and the user confirmed **pivot table, geo map and charts all work**. Instrumentation showed the fallback resolving only *resources*, never classes: 4× `META-INF/services/org.apache.commons.logging.LogFactory` and 3× Spring XSDs. The Spring-XSD lookups happen during **context build** (where the factory's own composite loader is what matters, [§B5](#b5--no-extender-at-all-a-blueprint-factory-bean----hot-check-likely-an-improvement), not the servlet's), and the commons-logging lookups are a *fallback to a default* rather than a failure. **Recommendation: keep** — it is cheap, it is the same class the factory already needs, and "no failure observed in one pass" is weaker evidence than the C5 lesson warrants. |
| **(b) `SimpleOutputHandler` instead of `GeneratorStreamingOutput`** | 🔴 **REQUIRED — proven** | Reverting the servlet to its original form breaks **every** content generator with a hard `LinkageError` (see below). `/content/analyzer/service`, `/generatedContent` and `/content/geojson` all returned **500**. |
| **(c) `registerMondrianDriverOnce`** | ✅ **REMOVED — proven dead code** | See [§C.4](#c4--the-predicted-mondrian-drivermanager-defect-is-disproven). Removed from the staged set, verified by cold start plus a user pivot-table test. |

**Hunk (b) — the exact failure when reverted:**

```
java.lang.LinkageError: loader constraint violation: when resolving method
  'void …GeneratorStreamingOutput.<init>(…, javax.servlet.http.HttpServletResponse, …)'
  the class loader …BundleClassLoader@5d7ecb35 of the current class,
  org/pentaho/platform/pdi/ContentGeneratorServlet, and the class loader
  java.net.URLClassLoader@3c5a99da for the method's defining class,
  org/pentaho/platform/web/http/api/resources/GeneratorStreamingOutput,
  have different Class objects for the type javax/servlet/http/HttpServletResponse
```

📄 `GeneratorStreamingOutput` lives in PDI's `lib/` (main class loader) and its constructor takes
`HttpServletRequest`/`HttpServletResponse`. The servlet lives in an OSGi bundle wired to a *different*
`javax.servlet` package. Passing servlet types across that boundary is a loader-constraint violation.
`SimpleOutputHandler` takes only an `OutputStream`, so **no servlet type crosses the boundary** — which
is precisely why the rewrite exists. This is the single load-bearing change in the whole rewrite.

**Net result:** C6 shrank the diff by **39 lines** (hunk (c), plus the now-unused logger and three
`java.sql` imports). Hunks (a) and (b) stay.

> ⚠️ **Honest limit on hunk (a).** It was exercised for one cold start and one functional pass. The
> composite class loader exists to merge `getResources()` across two loaders; a consumer that only
> reads an *optional* resource would degrade silently rather than fail — exactly the shape of the C5
> defect that endpoint checks missed. Keeping it costs nothing.

### C.4 ✅ The predicted Mondrian `DriverManager` defect is **disproven**

An earlier revision of this document reported, from code reading, that
`ContentGeneratorServlet.registerMondrianDriverOnce` would leak a driver into the JVM-global
`java.sql.DriverManager` across redeploys and then hand back a stale driver from an uninstalled
bundle's class loader. A first attempt failed to reproduce it, and it was downgraded to a
code-review note.

**It has now been instrumented directly and is disproven.** A temporary probe was added to the method
and run on a clean-cache install (`…-osgi`):

```
PROBE-C4: getDriver(jdbc:mondrian:) SUCCEEDED -> registration SKIPPED.
          driver=mondrian.olap4j.MondrianOlap4jDriver loader=java.net.URLClassLoader@3c5a99da
PROBE-C4: registered mondrian driver #1 loader=java.net.URLClassLoader@3c5a99da
PROBE-C4: total mondrian drivers in DriverManager = 1
```

**Mechanism.** 📄 `lib/mondrian-10.2.0.0-SNAPSHOT.jar` ships
`META-INF/services/java.sql.Driver` containing `mondrian.olap4j.MondrianOlap4jDriver`. `DriverManager`
auto-registers it from the **PDI launcher class loader** on first use. The servlet's guard —
`DriverManager.getDriver("jdbc:mondrian:")` — therefore *always* succeeds, and the fallback
registration **never runs**.

**Held across a real redeploy.** ✅ A genuine undeploy (verified: DET → 404 at T+15 s) followed by a
redeploy re-fired the probe from a fresh servlet instance:

| | Deploy #1 | After undeploy → redeploy |
|---|---|---|
| `getDriver` outcome | SUCCEEDED → skipped | SUCCEEDED → skipped |
| Mondrian drivers in `DriverManager` | **1** | **1** |
| Driver's class loader | `URLClassLoader@3c5a99da` | `URLClassLoader@3c5a99da` (**unchanged**) |

**Conclusion.** The servlet never registers a driver, so nothing can leak, the count never grows, and
the single driver belongs to a class loader that is never invalidated. The defect **cannot occur**.
The original concern came from reading the registration branch without checking that its guard always
short-circuits.

> **Lesson:** "this code would leak if it ran" is only a defect if the code actually runs. Instrument
> the guard before reporting the body.

### C.5 What the testing changed

| # | First-revision call | Measured | Direction |
|---|---|---|---|
| C3 `cxfBus` | 🔴 HOT-RISK — "keep it" | 🟢 Safe over 3 cold + 3 hot cycles | **Over-claimed** |
| C7 refinery flag | 🔴 HOT-RISK — "keep it" | 🟢 Safe on deploy and undeploy | **Over-claimed** |
| C4 `mtm` | 🔴 HOT-RISK — "keep it" | 🔴 **Confirmed** required under refresh | Correct |
| C5 webjars | 🟡 "probably unrelated to DET" | 🔴 **Required** — pivot table breaks without it | **Under-claimed** |
| C8 icu4j / connections | 🟡 "likely genuine" | icu4j **required**; connections **removable** | Half right |
| C1 rhino | 🟡 "cold start only" | 🟢 Confirmed on hot too | Correct |
| Mondrian defect | 🔴 "new defect" | ✅ **Disproven** — the guard always short-circuits, so the code never runs | **Over-claimed** |
| C6 servlet rewrite | 🟡 unassessed | Bisected: 1 hunk **required** (`LinkageError`), 1 dead (**removed**), 1 no-effect (kept) | Now settled |

Two lessons worth carrying forward:

1. **Reasoning about races over-predicts.** Both 🔴 markers I derived from mechanism alone (C3, C7)
   were wrong; the only 🔴 that survived (C4) was the one I could exercise *deterministically* by
   forcing a bundle refresh. Where a hypothesis cannot be turned into a deterministic trigger, treat it
   as unproven rather than as risk.
2. **Endpoint health checks are not functional tests.** C5 removed a feature that DET's pivot table
   needs, and every automated signal I had — HTTP status, response size, `node --check`, null-version
   scan — stayed green. Only driving the actual UI surfaced it.

### C.6 New finding: a failed deploy poisons the container until PDI restarts

While isolating C8, a deploy with `icu4j` missing left the analyzer bundle's Spring context in a failed
state. After that, **within the same JVM**, undeploying and redeploying a *known-good* KAR produced:

```
javax.servlet.ServletException: Error generating content for bean xanalyzer.service
Caused by: java.lang.NullPointerException
    at com.pentaho.analyzer.service.impl.LocalizationServiceImpl.getBundle(LocalizationServiceImpl.java:78)
    at com.pentaho.analyzer.content.AnalyzerContentGenerator.createContent(AnalyzerContentGenerator.java:260)
```

`/content/analyzer/service` → **500**, persisting across further cycles.

**Ruled out by isolation:**

* *Not* caused by repeated cycling — the unmodified KAR survived **three** deploy/undeploy/redeploy
  cycles on a fresh JVM with `analyzer=200` every time and zero `LocalizationServiceImpl` messages.
* *Not* caused by removing `pentaho-connections` — that change alone, on a fresh JVM, survived deploy
  **and** redeploy with `analyzer=200`.

The only remaining difference is the earlier **failed** deploy, so residual state from a failed
provisioning attempt is the most likely cause. ⚠️ **Inferred by elimination — not directly reproduced.**
Confirming it would take one deliberate run: deploy a knowingly-broken KAR, undeploy, then deploy a good
one.

**Practical impact:** if a hot deploy fails for any reason, a PDI **restart** may be required before a
corrected KAR will work. That is a meaningful operational caveat for hot deploy and belongs in
[`PDI-20686_hot.md`](./PDI-20686_hot.md).

### C.7 Not candidates — keep as-is

| Item | Why it must stay |
|---|---|
| The 4 analyzer content-generator beans | The root enabler of the Model view. |
| `SpringFileHandler` whiteboard properties + `<bean ` regex anchor | Pax Web 8 requirement; the regex fix prevents phantom servlets from XML comments. |
| `pentaho-platform-plugin-geo` unique `osgi.http.whiteboard.servlet.name` | Proven: without it only 1 of 4 servlets registers. |
| Empty `karaf-maven-plugin.prerequisiteFeatures` + no `prerequisite="true"` in the KAR graph | Proven: otherwise boot features are left `Uninstalled`. |
| `pax-web-jsp` | CDE's `commons-jxpath` has a mandatory `javax.servlet.jsp` import. |
| `common-ui` in the KAR | PDI does not ship it. |
| `icu4j 63.1` (**C8a**) | ✅ Proven: without it the analyzer Spring context dies with `NoClassDefFoundError: com/ibm/icu/text/DateFormat`. |
| `WebjarsURLConnection` version fallback (**C5**) | ✅ Proven: without it all `datatables.net*` webjar resources 404 and the pivot table breaks. |
| `PdiPlatformActivator` `removeProvider("mtm")` (**C4**) | ✅ Proven: without it a bundle refresh logs `Multiple providers registered for URL scheme "mtm"` and strands a stale provider. |

---

## D. Resulting change set (projected — see [§I.1](#i1-final-change-set) for what actually landed)

| Repository | Before this errata | After |
|---|---|---|
| `pentaho-analyzer` | 2 files (4 beans + nodejs hack) | **1 file** + `analyzer_beans.xml` = 2 files, nodejs hack dropped |
| `pentaho-osgi-bundles` | 11 files, incl. **new module** `pentaho-mini-spring-extender` (5 files) | **~7 files, no new module** (2 classes added to `pentaho-pdi-platform` instead) |
| `pentaho-det-ee` | 9 files, incl. **new module** `det/impls/lineage-stub` (3 files) | **~2 files, no new module** (C3 removes 2 blueprints, C7 removes the core feature edit) |
| `pentaho-det` | 1 file | **0 files** (C3 proven safe to drop) |
| `pentaho-platform-plugin-geo` | 1 file | 1 file |
| **New OSGi bundles introduced** | **2** | **0** |
| **New Maven modules introduced** | **2** | **0** |

That is the headline: **the change set can very plausibly introduce no new modules and no new bundles
at all** — only edits to existing ones.

---

## E. Hot-deploy impact of every errata item

The hot-deploy work is documented separately in [`PDI-20686_hot.md`](./PDI-20686_hot.md); its three
fixes are currently **reverted** from the repositories, so nothing in this errata conflicts with
working-tree state today. What follows is what would happen **if** hot deploy is later adopted.

### E.1 Summary

| Item | Change | Hot deploy | Basis |
|---|---|---|---|
| **A1** | Delete the lineage stub | 🟢 **HOT-SAFE** | ✅ Measured: hot deploy T+12 s, undeploy clean, redeploy T+16 s, 183 bundles, `bundle:diag` empty |
| A2 | Stub → geo | 🟢 HOT-SAFE | Same bundle, same KAR lifecycle |
| A3 | Stub → osgi-bundles | 🟡 HOT-CHECK | Would likely become a long-lived bundle rather than a KAR-scoped one |
| A4 | Fix the dataservice plugin | 🟡 HOT-CHECK | Changes a boot bundle's Blueprint |
| A5 | Install real metaverse | 🔴 HOT-RISK | Infeasible anyway (no OSGi manifest) |
| **B1** | Extender stays as-is | 🟢 **HOT-SAFE** | ✅ This is the configuration measured in `PDI-20686_hot.md` |
| B2 | Extender → deployer bundle | 🟡 HOT-CHECK | Extender becomes long-lived; `removedBundle()` disposal becomes load-bearing |
| B3 | Extender → deployers feature | 🟡 HOT-CHECK | Same as B2 |
| B4 | Revive the old module | 🟢 HOT-SAFE | Behaviourally identical to B1 |
| **B5** | No extender (factory bean) | 🟡 HOT-CHECK — **likely an improvement** | Ties the context to the plugin's own container and removes the 5 s timeout; unmeasured |
| **C1** | Drop redundant `rhino` | 🟢 **HOT-SAFE** | ✅ Measured hot: `org.mozilla.rhino` still present via `pentaho-cpf-pdi` |
| C2 | Drop the nodejs hack | 🟢 HOT-SAFE | Build-time only |
| **C3** | Drop `cxfBus` | 🟢 **HOT-SAFE** | ✅ Measured: 3 cold starts + 3 undeploy/redeploy cycles, 0 `DestinationFactory` errors — **earlier 🔴 was wrong** |
| **C4** | Revert `removeProvider("mtm")` | 🔴 **HOT-RISK — confirmed** | ✅ Measured: forcing a bundle refresh reproduces `Multiple providers registered for URL scheme "mtm"` |
| **C5** | Revert the webjars fallback | 🔴 **REQUIRED — confirmed** | ✅ Measured: all `datatables.net*` webjars 404 → **pivot table broken**; earlier 🟡 was wrong |
| C6 | Minimise `ContentGeneratorServlet` | 🟡 HOT-CHECK | Not tested — needs a per-hunk bisect |
| **C7** | Revert `dependency="true"` | 🟢 **HOT-SAFE** | ✅ Measured: no refresh cascade, refinery survives undeploy — **earlier 🔴 was wrong** |
| **C8a** | Drop `icu4j` | 🔴 **REQUIRED — confirmed** | ✅ Measured: `NoClassDefFoundError: com/ibm/icu/text/DateFormat`, analyzer 404 |
| **C8b** | Drop `pentaho-connections` | 🟢 **HOT-SAFE** | ✅ Measured on a fresh JVM: deploy + redeploy clean; user-confirmed Geo/Model/pivot |

### E.2 The 🔴 items after testing

Only **one** of the three items I originally marked 🔴 survived measurement, and testing promoted two
new items into the category.

* **C4 — `removeProvider("mtm")` — 🔴 confirmed, keep.** The only hypothesis I could turn into a
  *deterministic* trigger (force a bundle refresh with `bundle:update`), and it reproduced immediately.
  Scope caveat: a DET KAR cycle does not itself refresh `pentaho-pdi-platform`, so this protects other
  refresh paths.
* **C5 — webjars version fallback — 🔴 required, keep.** Promoted from 🟡. Without it the DET pivot
  table is broken by 404s on every `datatables.net*` resource.
* **C8a — `icu4j` — 🔴 required, keep.** Promoted from 🟡. Without it the analyzer Spring context
  cannot be built at all.
* **C3 — `cxfBus` — was 🔴, now 🟢.** Over-claimed. Survived 3 cold starts and 3 redeploy cycles with
  zero `DestinationFactory` errors. The original race was observed in a much earlier code state, before
  the `prerequisite="true"` feature fix; it appears to have been fixed by that change.
* **C7 — refinery `dependency="true"` — was 🔴, now 🟢.** Over-claimed. No refresh cascade on deploy,
  and the feature is not uninstalled on undeploy.

### E.3 File-level collision to plan for

`pentaho-det-ee/det/impls/pdi/src/main/resources-filtered/OSGI-INF/blueprint/blueprint.xml` is edited
by **both** work streams:

| Line | Change | Source |
|---|---|---|
| 13 | `<reference id="cxfBus" …/>` | cold start (removed by **C3**) |
| 134 / 142 / 163 | `bus="cxfBus"` on 3 `jaxrs:server` elements | cold start (removed by **C3**) |
| 92 | `init-method="applyToOpenTransformations"` on the `TransAfterOpen` bean | hot deploy (defect 3 fix) |

Now that C3 is measured safe and therefore *recommended for removal*, this collision becomes **easier**
rather than harder: applying C3 first leaves the file with only the hot-deploy edit, so the two work
streams no longer overlap on the `cxfBus` lines. Apply C3 before the hot-deploy fixes.

### E.4 What adopting A1 + B5 would do *for* hot deploy

Both headline recommendations are neutral-to-positive:

* **A1** removes a bundle from the KAR — one fewer artifact to install, resolve and dispose on every
  deploy/undeploy cycle. Measured faster (12 s vs 19 s), though that difference is within noise.
* **B5** removes the only remaining **timing constraint** shared by both modes (the analyzer's 5 s
  `ApplicationContext` timeout) and replaces a long-lived `BundleTracker` with a container-scoped bean
  whose disposal is handled by Blueprint. If it works, it makes hot deploy *structurally* simpler.

### E.5 Honest limits of this analysis

* **All of §A1 and §C are now measured.** The **B** options remain analysis only — B2, B3 and B5 have
  not been prototyped, so their 🟡 markers are unchanged.
* The hot-deploy runs used here did **not** include the three hot-deploy fixes (they are reverted), so
  they exercised the OSGi layer only — which is all that A1 and the §C items affect. The DET
  state-persistence symptom seen during testing was confirmed to be hot-deploy **defect 3**, not an
  errata effect, by the close-and-reopen diagnostic.
* Soak depth is **modest**: C3 got 3 cold starts + 3 redeploy cycles, not the ≥10 originally proposed.
  For a timing-sensitive change that is suggestive, not conclusive.
* **C6 is untested** and the Mondrian `DriverManager` concern is **unreproduced**.
* The B2/B3/B5 context-disposal concerns still need a soak, not a single pass.

---

## F. Open questions for you *(answered — see [§I](#i-what-was-applied))*

1. **A1** — confirm deleting `det-impl-lineage-stub` outright (recommended, proven), rather than
   relocating it.
2. **B** — pick **B5** (prototype the factory bean; removes the component and the 5 s race), **B2**
   (merge into the deployer), **B3** (feature-level, adds `pentaho-karaf-assembly`), or **B4** (revive
   the orphaned module).
3. **§C is settled by measurement** — confirm the resulting split:
   * **Remove:** C1 (rhino), C2 (nodejs hack), C3 (cxfBus, 5 endpoints across 2 repos), C7 (refinery
     flag), C8b (pentaho-connections).
   * **Keep:** C4 (mtm), C5 (webjars fallback), C8a (icu4j) — each now has a reproduction.
   * **Open:** C6 — do you want a per-hunk bisect of the `ContentGeneratorServlet` rewrite, or is a
     large-but-working diff acceptable there?
4. **Hot deploy** — still in scope? If yes, prefer **B5**, and apply **C3 before** the hot-deploy fixes
   to avoid the file collision in §E.3.
5. **Two follow-ups worth their own tickets** (neither is a PDI-20686 blocker):
   * the failed-deploy poisoning in [§C.6](#c6-new-finding-a-failed-deploy-poisons-the-container-until-pdi-restarts);
   * the unreproduced Mondrian `DriverManager` observation in [§C.4](#c4--the-predicted-mondrian-drivermanager-defect-is-disproven).
6. **`pdi-dataservice-server-plugin`** is **not cloned** under `~/github`. It is only needed if option
   A4 is chosen — which A.2 makes unnecessary. Flagging so nothing is silently assumed.

---

## G. Corrections that were made to `PDI-20686.md` ✅ done

Once the options above are settled:

| § | Current text | Correction |
|---|---|---|
| §1 | Lists the lineage stub implicitly in the change footprint | Remove from the narrative |
| §3 | *"`det-core` / `det-pdi` Blueprint containers never build → Missing `ILineageClient` provider"* | Wrong attribution. That symptom was caused by the `prerequisite="true"` feature starvation (§6.4) |
| §5.3 | `det/impls/lineage-stub` *(new module)* | Delete the row |
| §6.3 | *"Why a no-op `ILineageClient` stub"* — the whole subsection | Replace with a short note: `ILineageClient` **is** provided in a stock PDI client by `pentaho-blueprint-activators`, which bridges the metaverse Kettle plugin's `LineageClient` into OSGi. No stub is required. |
| §4 diagram | Shows `det-impl-lineage-stub` in the KAR | Remove the box |
| §5.2 / §6.2 | Describe `pentaho-mini-spring-extender` as a new module | Update to whichever of B1–B5 is chosen |
| §11 risk 5 | *"`CompositeClassLoader` duplicated"* | Re-evaluate — under B5 there is one copy, in `pentaho-pdi-platform` |
| §12.4 traps | — | Add: *"a service can be published by the `PentahoSystem`→OSGi bridge, not only by a Blueprint. Always confirm with `service:list <interface>` on a running instance before concluding that nothing provides it."* |

And in [`PDI-20686_hot.md`](./PDI-20686_hot.md):

| § | Correction |
|---|---|
| §9.2 remaining work | Add a task to confirm the failed-deploy poisoning (§C.6 of this errata) |
| §11 residual risks | Add: **a failed hot deploy can poison the container** — after a deploy that fails, redeploying a corrected KAR in the same JVM can still fail (`LocalizationServiceImpl` NPE, analyzer 500); a PDI restart may be required. Inferred by elimination |
| §11 residual risks | Add the Mondrian `DriverManager` observation **as a code-review note, explicitly unreproduced** — do not record it as a confirmed defect |
| §13 reproduction | Add the harness guard: assert the KAR file exists before `cp`, and print the deployed file — a silent no-op copy produces "404 with an empty log", which reads exactly like a product failure |

---

## I. What was applied

**Date:** 2026-08-11. **Applied:** A1, B5, C1, C3, C6, C7, C8b. **Kept:** C4, C5, C8a (proven
required). **Reverted:** C2 (the local nodejs hack — never committed).

### I.1 Final change set

Now committed on branch `PDI-20686` in each repository, under
`fix: it is not possible to use DET on Pentaho 10.2.x [PDI-20686]`.

| Repository | Files | Notes |
|---|---|---|
| `pentaho-analyzer` | 2 | `plugin.spring.xml` (4 CG beans), `analyzer_beans.xml` (**B5** factory bean) |
| `pentaho-osgi-bundles` | 7 source + 8 test | `SpringContextFactory.java` (**new class**, B5), `CompositeClassLoader.java`, `ContentGeneratorServlet.java` (−39 lines after the C6 bisect), `PdiPlatformActivator.java`, `SpringFileHandler.java`, `PluginXmlStaticPathsHandler.java`, `WebjarsURLConnection.java` |
| `pentaho-det-ee` | 3 | `feature.xml`, `det/assemblies/pdi/pom.xml`, root `pom.xml` (`icu4j.version`) |
| `pentaho-platform-plugin-geo` | 1 | `blueprint.xml` — whiteboard servlet names **+ B5 factory bean** |
| `pentaho-det` | **0** | Removed from the delivery entirely by **C3** |

See [`PDI-20686.md` §5](./PDI-20686.md#5-change-footprint-per-repository) for the authoritative,
file-by-file footprint.

**Removed:** 2 Maven modules (`det/impls/lineage-stub`, `pentaho-mini-spring-extender`), 3 OSGi bundles
from the KAR, and 1 repository from the delivery.

> **The delivery introduces no new Maven modules and no new OSGi bundles.**

### I.2 ⚠️ B5 caused one regression — found and fixed

The first build of B5 updated only the analyzer. Cold start then showed `/content/geojson` returning
**HTTP 500**:

```
ServiceUnavailableException: No matching service for optional OSGi service reference:
  (&(Bundle-SymbolicName=pentaho-geo-agile-bi)(objectClass=org.springframework.context.ApplicationContext))
```

**Cause.** The deleted extender was *generic* — it built a context for **every** bundle shipping
`META-INF/spring/*.xml`, which silently included `pentaho-geo-agile-bi`. B5 is opt-in per plugin, so geo
lost its context. §B5's own risk note predicted this ("any *other* platform plugin deployed this way
would need the same bean") but the first implementation did not act on it.

**Fix.** The same three-line bean in geo's blueprint. It ships **inside** the `pentaho-geo-agile-bi`
bundle it used to filter on, so `blueprintBundle` resolves to that same bundle — no cross-bundle lookup
needed.

**Scope confirmed by scanning the built KAR:** exactly two bundles carry `META-INF/spring/*.xml` —
`paz-plugin-ce` and `pentaho-geo-agile-bi`. Both are now updated. `common-ui` ships none.

> This is why [`PDI-20686.md` §11](./PDI-20686.md#11-residual-risks-and-known-issues) risk 5 was added:
> the opt-in model trades the extender's over-reach for a footgun, and the durable fix would be to have
> `SpringFileHandler` *generate* the bean when a plugin does not define `spring` itself.

### I.3 Verification

| Check | Result |
|---|---|
| Cold start, clean cache | ✅ clean |
| `SpoonDebug.txt` | **71 lines, zero errors** (was 79 with the previous design) |
| Active bundles | **181** (was 184 — the 3 removed) |
| `bundle:diag` | **empty** |
| Boot features (`pentaho-client-minimal`, `pentaho-big-data-plugin-osgi`) | all **Started** |
| `require-init.js` | **200, 180 969 B** |
| `/cxf/det/core/data/datasources`, `/cxf/det/core/persistence/states` | **200** |
| `/content/analyzer/service`, `/generatedContent` | **200** |
| `/content/geojson`, `/content/geoprint` | **200** |
| Bundles absent as intended (`mini-spring`, `lineage`, `pentaho-connections`) | **0 / 0 / 0** |
| `rhino`, `com.ibm.icu` still present via other suppliers | **1 / 1** |
| **Functional pass in Spoon** | ✅ Model view, charts, **geo map**, pivot table — all confirmed working |

### I.4 Not applied, and why

| Item | Decision |
|---|---|
| **C2** nodejs downgrade | ✅ **Reverted.** A local-build-only hack; it was never committed and is no longer present in the working tree |
| **C4** `removeProvider("mtm")` | Kept — proven required by forcing a bundle refresh |
| **C5** webjars version fallback | Kept — proven required; without it the pivot table breaks |
| **C8a** `icu4j` | Kept — proven required; without it the analyzer Spring context cannot build |
| **C6** `ContentGeneratorServlet` minimisation | ✅ **Done (2026-08-11).** Bisected into 3 hunks; the dead Mondrian registration was removed (−39 lines). See [§C.3](#c3--c6--contentgeneratorservlet-rewrite-bisected). |
| **Unit tests** | ✅ **Added (2026-08-11).** 45 new tests across 5 files covering every behaviour change, plus `PdiPlatformActivatorTest` and an order-independence fix to `MetadataToMondrianVfsTest`. The webjars test was **verified to fail without its fix** (`stateHelper@null/*`). |
| **A2–A5, B1–B4** | Superseded by A1 and B5 |

### I.5 Honest limits

* **Every measurement in [§I.3](#i3-verification) is a cold start.** Hot deploy has since been exercised
  manually against the committed change set and behaves correctly, but it was not re-measured with the
  instrumented harness. The B5 shape removes the 5 s timeout and scopes context disposal to the
  container, so it should make hot deploy *better* — see [`PDI-20686_hot.md`](./PDI-20686_hot.md).
* **Only `paz-plugin-ce` was updated, not `paz-plugin-ee`** — the CE variant is what PDI ships.
* C6 hunk (a) (the composite class loader as TCCL) showed **no measurable effect** in one pass and was
  kept on precaution, not on evidence of need. See the caveat in [§C.3](#c3--c6--contentgeneratorservlet-rewrite-bisected).

---

## J. Change log

| Date | Change |
|---|---|
| 2026-08-13 (rev 7) | **Reconciled with the committed change set.** Corrected the status header and [§0](#0-executive-summary) (C6 was closed in rev 5 but still listed as unassessed; the Mondrian defect is *disproven*, not merely unreproduced). Corrected [§I.1](#i1-final-change-set) file counts against the actual commits (analyzer 3→2, osgi-bundles 6→7 source + 8 test, det-ee 2→3) and pointed it at `PDI-20686.md` §5 as authoritative. Recorded that **C2 was reverted**, that the work is now **committed**, and that hot deploy has since been manually exercised. Removed the `ssh` boot feature from the verification table. |
| 2026-08-11 (rev 6) | **Final review for commit.** Audited every staged hunk against "required for DET cold start"; confirmed `pentaho-det` has zero changes and the `nodejs` hack is unstaged. Added **45 unit tests** (`SpringContextFactoryTest`, `CompositeClassLoaderTest`, `ContentGeneratorServletTest`, `SpringFileHandlerTest`, `PdiPlatformActivatorVfsRegistrationTest`, plus a `WebjarsURLConnectionTest` case with a new no-pom fixture). Corrected stale javadoc on `CompositeClassLoader` and documented the `LinkageError` rationale on `ContentGeneratorServlet`. Re-verified by full rebuild + cold start: 181 bundles, empty `bundle:diag`, all endpoints 200, functional pass confirmed. |
| 2026-08-11 (rev 5) | **C4 and C6 closed.** The Mondrian `DriverManager` defect is **disproven** by direct instrumentation — mondrian auto-registers from PDI's `lib/` via `META-INF/services/java.sql.Driver`, so the servlet's guard always short-circuits and the registration branch never executes (confirmed unchanged across a real undeploy→redeploy). C6 bisected into three hunks: the output-handler swap is **required** (reverting it throws `LinkageError` on every content generator), the Mondrian block is **dead** (removed, −39 lines), the composite class loader shows no measurable effect (kept). Re-verified on a pristine install: 181 bundles, empty `bundle:diag`, analyzer 4 servlets / **0** `ApplicationContext` services, geo 4 servlets. |
| 2026-08-11 (rev 4) | **Options A1, B5, C1, C3, C7, C8b applied, built and verified.** Added [§I](#i-what-was-applied). B5 regressed `/content/geojson` (the deleted extender had been silently supplying `pentaho-geo-agile-bi`'s Spring context); fixed by applying the same factory bean to geo. Result: no new modules, no new bundles, `pentaho-det` out of the delivery, 181 active bundles, clean cold start, full functional pass. |
| 2026-08-10 (rev 3) | **All §C candidates built and run.** C3 and C7 cleared (earlier 🔴 over-claimed); C5 and C8a promoted to **required** (C5 discovered via a UI pivot-table test after every automated signal stayed green); C8b and C1 cleared under hot deploy; C4 **confirmed** by forcing a bundle refresh. Mondrian `DriverManager` defect **not reproduced** — downgraded to a code-review note. New finding: a failed deploy poisons the container until PDI restarts (§C.6). Scorecard added at §C.5. |
| 2026-08-10 (rev 2) | Added [§E](#e-hot-deploy-impact-of-every-errata-item): every item marked 🟢/🟡/🔴 for hot-deploy impact. A1 additionally **measured** under hot deploy, undeploy and redeploy (safe). Three candidates reclassified to *keep* (C3, C4, C7). Documented a file-level collision between C3 and the hot-deploy defect-3 fix, and a **new hot-deploy defect** found by code reading (§C.3, Mondrian `DriverManager` leak across redeploys). |
| 2026-08-10 | Initial version. Establishes that the `ILineageClient` stub is unnecessary (proven by removal), that the rhino bundle line is redundant (proven), and sets out five options for the mini Spring extender including one that removes it entirely. |
