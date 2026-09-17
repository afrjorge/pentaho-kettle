---
type: pattern
title: "Data Service Step Cache Divergence in Spoon (PDI 11.0 - 11.1)"
description: "Two PDI Data Service defects reproduced on 11.0.0.2, 11.0.0.3 and 11.1: a split step cache that leaves the step badge and right-click menu stale, and SP-7071's file:// filename normalisation on open, which both orphans published ServiceTrans pointers and can make a transformation impossible to save."
tags: [architecture, spring, pdi, data-services, spoon, cache, extension-point, regression, metastore, cross-version, vfs]
timestamp: 2026-09-21T20:55:00Z
---

# Data Service Step Cache Divergence in Spoon (PDI 11.0 - 11.1)

## 1. Symptom

PDI 11.0.0.3 with `plugins/pdi-dataservice-ee-plugin` installed:

1. Right-click a step -> **Data Services** -> **New...** -> fill in the dialog -> **OK**.
2. The step canvas does **not** paint the bottom-left Data Service badge.
3. Re-opening the step's **Data Services** submenu still shows **New...** enabled and **Edit... / Delete... / Test...** disabled.
4. Saving the transformation does not change anything.
5. The View panel *does* list the service under **Configurations -> Data Services**.
6. Selecting / right-clicking -> **Edit...** on that tree node (and closing the dialog) makes the badge appear and the menu items become correct.

## 2. Components Involved

| Class | Cache it reads/writes |
|---|---|
| `org.pentaho.di.trans.dataservice.TransPainterStepExtensionPointPlugin` (draws the badge, EP `TransPainterStep`) | `DataServiceContext.getInstance().getMetaStoreUtil()` — **singleton** cache |
| `org.pentaho.di.trans.dataservice.ui.menu.DataServiceStepMenuExtension` (enables/disables `dataservices-new/edit/delete/test`, EP `TransStepRightClick`) | `DataServiceContext.getInstance().getMetaStoreUtil()` — **singleton** cache |
| `org.pentaho.di.trans.dataservice.ui.menu.DataServiceTreeDelegateExtension` (EP `SpoonTreeDelegateExtension`) | `DataServiceContext.getInstance().getMetaStoreUtil()` — **singleton** cache |
| `org.pentaho.di.trans.dataservice.ui.menu.DataServiceStepHandler` -> `DataServiceDelegate` (New/Edit/Delete/Test actions) | **its own private** cache |
| `org.pentaho.di.trans.dataservice.ui.menu.DataServicePopupMenuExtension`, `DataServiceViewTreeExtension`, `DataServiceFolderProvider`, `DataServiceStepDeleteExtensionPointPlugin` | **their own private** caches |

## 3. Root Cause

`DataServiceMetaStoreUtil` keeps a `javax.cache.Cache<Integer, String>` (`stepCache`) mapping
`hash(transName, stepName) -> dataServiceName`, where an **empty string is a negative marker**
("this step has no data service").

`DataServiceMetaStoreUtil.getDataServiceByStepName()` short-circuits on that marker:

```java
// DataServiceMetaStoreUtil.java:208-238
for ( Map.Entry<Integer, String> entry : stepCache.getAll( cacheKeys ).entrySet() ) {
  String serviceName = entry.getValue();
  if ( serviceName.isEmpty() ) {
    return null;                       // <-- never falls through to the embedded metastore
  }
  ...
}
// ... otherwise scan the embedded metastore, then:
stepCache.putAll( Maps.asMap( cacheKeys, Functions.constant( "" ) ) );  // negative caching
```

The cache is created **per instance**, with a random name, so two instances never share state:

```java
// DataServiceMetaStoreUtil.java:84-88
private static Cache<Integer, String> initCache( PentahoCacheManager cacheManager ) {
  String name = DataServiceMetaStoreUtil.class.getName() + UUID.randomUUID().toString();
  PentahoCacheTemplateConfiguration template = cacheManager.getTemplates().get( Constants.DEFAULT_TEMPLATE );
  return template.createCache( name, Integer.class, String.class );
}
```

And the write path used by the UI is a *different* instance from the read path used by the painter/menu:

* `DataServiceDelegate extends DataServiceFactory extends DataServiceMetaStoreUtil`
  (`DataServiceDelegate.java:39`, `DataServiceFactory.java:32`), and its constructors call `super()`
  (`DataServiceDelegate.java:54-68`), i.e. `initCache(...)` -> **a brand new cache**.
* `DataServiceContext.getDataServiceDelegate()` returns `DataServiceDelegate.withDefaultSpoonInstance( this )`
  — **a new delegate (and therefore a new cache) on every single call** (`DataServiceContext.java:174-176`).
* `DataServiceContext.getMetaStoreUtil()` returns the singleton `DataServiceMetaStoreUtil.getInstance()`
  (`DataServiceContext.java:113,149-151`, `DataServiceMetaStoreUtil.java:67-73`).

### Failing sequence

1. Right-click on the step fires `TransStepRightClick` -> `DataServiceStepMenuExtension` ->
   `getDataServiceByStepName()` on the **singleton** cache -> not found -> **negative marker `""` stored in the singleton cache**.
   (`TransPainterStep` repaints store the same marker.)
2. **New...** -> `DataServiceStepHandler` -> `DataServiceDelegate` (fresh instance, fresh cache) ->
   `DataServiceDialogController.saveAndClose()` -> `delegate.save(...)` ->
   `EmbeddedMetaStoreFactory.saveElement()` -> `saveToCache()` writes the positive entry into the
   **delegate's private cache only** (`EmbeddedMetaStoreFactory.java:63-71`).
3. `spoon.refreshGraph()` repaints -> `TransPainterStepExtensionPointPlugin` reads the **singleton** cache ->
   still `""` -> no badge. `DataServiceStepMenuExtension` does the same -> wrong enablement.
4. Saving the transformation does not help: the stale negative entry is never invalidated, and the embedded
   metastore is never consulted while the marker is present.

### Why the View-tree "Edit..." workaround works

Selecting the service node in the View tree fires `SpoonTreeDelegateExtension` ->
`DataServiceTreeDelegateExtension` (case 3) -> `metaStoreUtil.getDataService( path[2], transMeta )` on the
**singleton** util -> `EmbeddedMetaStoreFactory.loadElement()` -> `saveToCache()` **overwrites the stale `""`
with the real service name in the singleton cache** (`DataServiceTreeDelegateExtension.java:41,64`,
`EmbeddedMetaStoreFactory.java:44-51`). From that point the painter and the step menu see the service.

### Regression origin

Introduced by `3398405 [BACKLOG-40276] Moved PDI data services out of OSGi and Merged Pdi Data Service
client / Server into one repo`. Under OSGi/Blueprint, `DataServiceContext`, `DataServiceMetaStoreUtil` and
`DataServiceDelegate` were wired as shared singletons with a single injected `stepCache`. The de-OSGi
migration replaced injection with `getInstance()` singletons for the *read* side but left `new`-based
construction for the *write* side, splitting the cache.

Secondary defect: because `getDataServiceDelegate()` creates a new delegate on every call, every call also
creates a new named cache in the `PentahoCacheManager` — an unbounded cache/resource leak.

## 4. What Needs to Be Fixed

> **Status (v2.0.0):** items 1, 2, 3 and 4 are **implemented and verified** — see sections 6 and 7.
> Item 6 (regression tests) was delivered in v2.1.0; only item 5 is still open — see section 9. Note that item 3's "orphan pointer" aspect was ultimately
> fixed in `ServiceTrans`, not by eager deletion — see section 5.

Repository: `pdi-dataservice-ee-plugin` (branch `practitest_5375`, cut from `11.0`), module `core`.

1. **Single shared step cache (primary fix).**
   Make every `DataServiceMetaStoreUtil` subclass share one cache instance. Options, in order of preference:
   * Make the cache `static` / lazily created once in `DataServiceMetaStoreUtil` (`initCache` with a fixed
     cache name instead of `UUID.randomUUID()`), so `DataServiceDelegate`/`DataServiceFactory` inherit it; or
   * Add a protected constructor chain so `DataServiceFactory`/`DataServiceDelegate` pass
     `DataServiceMetaStoreUtil.getInstance().getStepCache()` to `super(cache)`.
   Files: `serialization/DataServiceMetaStoreUtil.java`, `serialization/DataServiceFactory.java`,
   `ui/DataServiceDelegate.java`.

2. **Stop creating a new delegate (and a new cache) per call.**
   `DataServiceContext.getDataServiceDelegate()` must memoize a single `DataServiceDelegate` instance
   (`DataServiceContext.java:174-176`). This also removes the cache leak.

3. **Explicit invalidation on create/save/rename/delete.**
   Even with a shared cache, add defensive invalidation of the step keys for the affected
   `(transName, stepName)` in `DataServiceMetaStoreUtil.save()` / `removeDataService()` /
   `SynchronizationListener.onStepChange()`, so a rename or a cross-instance change cannot leave a stale
   positive/negative entry. Note `createCacheKeys()` hashes `transMeta.getName()` — renaming the
   transformation also orphans entries.

4. **Refresh the UI after the write, on the same data.**
   `DataServiceDelegate.save()` already calls `spoon.refreshTree(...)` + `spoon.refreshGraph()`
   (`ui/DataServiceDelegate.java:176-182`); once (1) is fixed this becomes sufficient. Verify the same for
   `removeDataService()` so the badge disappears immediately on delete.

5. **Consider dropping negative caching** in `getDataServiceByStepName()`, or bounding it with a short TTL,
   since the embedded metastore lookup is cheap relative to the staleness risk it creates.

6. **Regression tests.**
   * `DataServiceStepMenuExtensionTest` / `TransPainterStepExtensionPointPluginTest`: assert that a service
     saved through a `DataServiceDelegate` obtained from `DataServiceContext` is immediately visible through
     `DataServiceContext.getInstance().getMetaStoreUtil().getDataServiceByStepName(...)`.
   * New test asserting `context.getDataServiceDelegate() == context.getDataServiceDelegate()` and that both
     share the same `getStepCache()` instance as the singleton util.

### Manual verification steps

Install the rebuilt plugin into
`/Users/andjorge/pentaho/pdi/11.0.0.3/pdi-ee-client-11.0.0.3-312/data-integration/plugins/pdi-dataservice-ee-plugin`,
then:

1. Right-click a step **before** creating the service (this is what poisons the cache) -> **Data Services** -> **New...**.
2. Save the dialog. **Expected:** badge appears immediately; submenu shows **New...** disabled and
   **Edit... / Delete... / Test...** enabled without touching the View tree.
3. **Delete...** the service. **Expected:** badge disappears immediately and **New...** is re-enabled.
4. Rename the step. **Expected:** badge follows the step, menu state stays correct.

## 5. Orphan `ServiceTrans` Reference Files (Secondary Defect)

### Symptom

`~/.pentaho/metastore/pentaho/Data Service Transformation/` accumulates one `<serviceName>.xml` file per data
service ever created, and the file is **never** removed — not when the data service is deleted, and not when
the transformation is subsequently saved.

Observed state before the fix:

```
2test.xml   2ttest.xml   4test.xml   5test.xml
test 11 0 0 2.xml   test 11 0 0 2 v3.xml   test post.xml   testtt.xml   tttest.xml
```

Eight of those referenced `/Users/andjorge/Downloads/dataservices_11002.ktr`, whose embedded metastore held a
single data service.

On `master` the same delete-then-save sequence **does** remove the file. That difference is the key clue and
is explained below — it is not a code difference between the branches.

### The two stores, and who reaps what

A data service is persisted in **two** places:

| Store | Element | Written by |
|---|---|---|
| Transformation's **embedded** metastore (inside the `.ktr`) | `DataServiceMeta` | `EmbeddedMetaStoreFactory` via `DataServiceMetaStoreUtil.save()` |
| **Global** metastore (`~/.pentaho/metastore/pentaho/Data Service Transformation/`) | `ServiceTrans` (a pointer: service name -> `.ktr` location) | `DataServiceReferenceSynchronizer.sync()`, driven by `SynchronizationListener.contentSafe()` on transformation **save** |

By design the published pointers describe the **saved** state of the `.ktr`, so they are reaped at save time,
not at delete time:

```
Spoon save -> AbstractMeta.clearChanged() -> fireContentChangedListeners( false )
           -> SynchronizationListener.contentSafe()
           -> DataServiceReferenceSynchronizer.sync()
              toDelete = publishedInTrans - servicesInTrans   ->  serviceTransFactory.deleteElement( name )
```

`SynchronizationListener` is installed on every `TransMeta` by
`TransOpenedExtensionPointPlugin` (extension point `TransAfterOpen`) and again by
`DataServiceDialogController.saveAndClose()`.

### Root Cause: `SP-7071` normalises opened filenames to `file://`, which breaks the synchronizer

`ServiceTrans.references( TransMeta )` stores `TransMeta.getFilename()` **verbatim**
(`ServiceTrans.java:73`). Both forms are found side by side in the metastore folder:

```
blo.xml      -> file:///Users/andjorge/Downloads/dataservices_11100_3.ktr
bla11003.xml -> /Users/andjorge/Downloads/dataservices_11103_1.ktr
```

**What decides the form is `pentaho-kettle` commit `f560af820d6`
`fix[SP-7071]: Preserve trans/job filename in Carte remote execution` (Bruno Cavaleiro, 2026-04-23).**
It added `KettleVFS.normalizeFilePath( String )`, which prefixes any scheme-less local path with `file://`:

```java
// KettleVFS.java (added by SP-7071)
if ( hasSchemePattern( result ) ) {
  return result;                       // already file:// / hdfs:// / s3:// -> unchanged
}
File file = new File( result );
String absolutePath = file.getAbsolutePath();
if ( absolutePath.startsWith( "/" ) ) {
  result = "file://" + absolutePath;   // /Users/a/b.ktr  ->  file:///Users/a/b.ktr
} else {
  result = "file:///" + absolutePath.replace( '\\', '/' );   // Windows
}
```

and wired it into Spoon's **open** path:

```java
// TransFileListener.java, SP-7071
- transMeta.setFilename( fname );
+ transMeta.setFilename( KettleVFS.normalizeFilePath( fname ) );
```

The same commit also started **persisting** the filename into the `.ktr`
(`TransMeta.getXML()`, `<transformation><info><filename>`), with a matching read in `TransMeta.loadXML()`
that only applies when the in-memory filename is empty. `JobMeta` got the identical treatment.

This produces an asymmetry between the two ways a `.ktr` acquires a filename:

| How the filename is set | Goes through `normalizeFilePath()`? | `TransMeta.getFilename()` | `<info><filename>` persisted | `transformation_location` written |
|---|---|---|---|---|
| **New transformation -> Save / Save As** | no | `/Users/.../x.ktr` (plain) | `/Users/.../x.ktr` | `/Users/.../x.ktr` |
| **Close -> Open -> Save** (`TransFileListener.open()`) | **yes** | `file:///Users/.../x.ktr` | `file:///Users/.../x.ktr` | `file:///Users/.../x.ktr` |

So the `file://` form is not a property of a particular dialog — **every** Spoon open path is affected,
because they all funnel through `TransFileListener.open()`. That is why the section 10 matrix sees the URI
form from the VFS browser, from Finder drag & drop **and** from `spoon.sh -file=` alike.

Two places in `ServiceTrans` then fail on the URI form:

1. `ServiceTrans.StorageMethod.FILE.exists()` resolved the location with `new File( location ).exists()`,
   which is **always false** for a `file://` URI:

   ```
   new File("/Users/andjorge/Downloads/dataservices_11002.ktr").exists()       -> true
   new File("file:///Users/andjorge/Downloads/dataservices_11002.ktr").exists() -> false
   ```

   `DataServiceReferenceSynchronizer.getAllPublishedServiceTrans()` keeps only pointers that pass that
   check, so URI-based pointers were filtered out of `allPublished`, hence out of `publishedInTrans`, hence
   could never appear in `toDelete`. **The synchronizer literally could not see them.**

2. `Reference.equals()` / `hashCode()` compared the **raw** location strings, so a pointer stored in one form
   never matched the same transformation expressed in the other form — affecting both
   `getPublishedServicesFromTransMeta()` and `ServiceTrans.isReferenceTo()`.

### Affected versions, established by build inspection

`KettleVFS.normalizeFilePath` and its use in `TransFileListener` are present in exactly the builds that leak:

| PDI build | `KettleVFS.normalizeFilePath` | `TransFileListener` uses it | `<info><filename>` in saved `.ktr` | Pointer reaped on delete + save |
|---|---|---|---|---|
| 11.0.0.0 (237) | absent | no | absent | **yes** |
| 11.0.0.1 (259) | absent | no | absent | **yes** |
| 11.0.0.2 (294) | present | yes | present | **no** (after reopen) |
| 11.0.0.3 (312) | present | yes | present | **no** (after reopen) |
| 11.1.0.0-SNAPSHOT (20260916-335) | present | yes | present | **no** (after reopen) |
| `master` | **absent** (SP-7071 is not on `master`) | no | absent | **yes** |

`git branch -a --contains f560af820d6` lists 26 branches — `11.0`, `practitest_5375` and various feature
branches — but **not** `master`. That, not the file dialogs, is the whole explanation of the
`master` vs `11.0` difference. **The data-service source is functionally identical on both branches** —
verified by diffing `master` against `11.0` for the whole `core` module, where the only differences are
licence headers and `commons-lang` vs `commons-lang3` imports; `pentaho-kettle`'s `AbstractMeta` is
identical too.

### Diagnosing an affected `.ktr`

Because SP-7071 persists the filename, the `.ktr` itself tells you which form it carries, and therefore
whether its data services will leak:

```bash
grep -m1 "<filename>" <path-to>.ktr
```

* absent -> pre-11.0.0.2 build, or written by `master`; pointers reap normally.
* `<filename>/Users/...</filename>` -> plain form; the pointer will be reaped correctly.
* `<filename>file:///Users/...</filename>` -> URI form; **the pointer for every data service saved from this
  state will leak.**

The value always matches the `transformation_location` of the pointers published from that transformation.

### Two failure modes, selected by which form the pointer was published in

The raw-string comparison produces **two opposite symptoms**, depending on whether the pointer was published
before or after the transformation was first reopened. Both are driven by
`DataServiceReferenceSynchronizer.sync()`:

```java
// DataServiceReferenceSynchronizer.java:67-75
List<String> toDelete      = publishedInTrans - servicesInTrans;
List<String> nameConflicts = servicesInTrans
                               .filter( allPublished.contains( name ) && !publishedInTrans.contains( name ) );
List<String> toSave        = servicesInTrans.filter( overwrite || !nameConflicts.contains( name ) );
```

* `allPublished` = pointers whose `reference.exists( repository )` is true (`new File( location ).exists()`).
* `publishedInTrans` = of those, the ones whose `Reference` **equals** one of `ServiceTrans.references( transMeta )`
  — a raw-string comparison of the location.

| Pointer `transformation_location` | `TransMeta.getFilename()` now | `exists()` | in `allPublished` | in `publishedInTrans` | Outcome |
|---|---|---|---|---|---|
| `file:///p.ktr` (published *after* a reopen) | `file:///p.ktr` | **false** | no | no | pointer is invisible -> never in `toDelete` -> **orphan leak** |
| `/p.ktr` (published *before* the first reopen) | `file:///p.ktr` | **true** | yes | **no** | -> **`nameConflicts`** -> `DataServiceAlreadyExistsException` -> **save deadlock** |
| `/p.ktr` | `/p.ktr` (never reopened) | true | yes | yes | correct: saved, and reaped on delete |

The second row is the more damaging of the two and is described next.

### Third symptom: unrecoverable "data service already exists" save loop

**Repro:** create a transformation **and** its data service in one session and save (pointer published with
the plain `/...` location) -> close the transformation -> reopen it (filename becomes `file:///...`) -> make
any change so it is dirty -> **Ctrl+S**.

The save writes the file, then `AbstractMeta.clearChanged()` fires `contentSafe()` -> `sync()`. The pointer
is in `allPublished` (its plain location passes `new File(...).exists()`) but not in `publishedInTrans` (its
location no longer string-matches the now-normalised filename), so the service lands in `nameConflicts`:

1. `toSave` **excludes** it, so the pointer is never rewritten with the new location — the mismatch can
   never self-heal.
2. `exceptionHandler` receives a `DataServiceAlreadyExistsException`, and
   `SynchronizationListener.contentSafe()` routes it to `DataServiceDelegate.suggestEdit()`.
3. `suggestEdit()` runs because `hasChanged()` is false (the save just cleared it) — and **both** of its
   branches call `serviceTrans.setChanged()`:

   ```java
   // DataServiceDelegate.java:120-136
   if ( !serviceTrans.hasChanged() ) {
     if ( showPrompt( title, text ) ) {
       editDataService( dataServiceMeta );
       serviceTrans.setChanged();                              // dirty again
     } else {
       // "can not exist duplicate, delete dataService and save transformation (PDI-15584)"
       deleteDataServiceElementAndCleanCache( dataServiceMeta, serviceTrans );
       serviceTrans.setChanged();                              // dirty again
     }
   }
   ```

So every save ends with the transformation marked dirty again and the same prompt on the next attempt —
**the transformation can never be returned to a clean state** unless the user renames the data service in
the edit dialog (which ends the loop but silently renames a service that was never in conflict, and still
leaves the original pointer orphaned — see Annex B and Annex C), exactly as reported.

**Answering "No" is worse, not an escape.** The `else` branch calls
`deleteDataServiceElementAndCleanCache()`, which does exactly two things:

```java
// DataServiceMetaStoreUtil.java:330-333
getDataServiceFactory( transMeta ).deleteElement( dataService.getName() );  // embedded DataServiceMeta only
invalidateStepCache( transMeta, dataService.getStepname() );
```

It removes the **embedded** `DataServiceMeta` and nothing else — the global `ServiceTrans` pointer XML is
only ever deleted by `sync()`'s `toDelete` list. And `toDelete` is derived from `publishedInTrans`:

```java
// DataServiceReferenceSynchronizer.java:67-69
List<String> toDelete = publishedInTrans.keySet().stream()
  .filter( name -> !servicesInTrans.keySet().contains( name ) )
```

`publishedInTrans` is empty here — that is the whole reason the conflict arose — so `toDelete` is empty on
this save and on every save afterwards. The result matches the observation exactly: **the data service is
deleted but its XML file is not**, and because the embedded service is now gone there is no longer anything
that could ever republish or reap that pointer. The user loses the data service *and* is left with a
permanent orphan, from a single click on a dialog they had no reason to trust.

This makes "No" a strictly destructive answer **while the mismatch exists**: it does not resolve the
duplicate it claims to resolve (the pointer it is nominally deduplicating is precisely the one it leaves
behind), it destroys the user's work, and it converts the save loop into the first failure mode. The
`serviceTrans.setChanged()` at the end of the branch also leaves the transformation dirty, so the user must
still save again — this time with the data service silently missing.

**Why the branch is nevertheless left unchanged by the fix.** It is tempting to also delete the pointer in
this branch, but that would be wrong once the scheme mismatch is repaired. `nameConflicts` is by definition
*"in `allPublished` but not in `publishedInTrans`"*, and the `publishedInTrans` filter
(`DataServiceReferenceSynchronizer.java:97-101`) and `ServiceTrans.isReferenceTo()` (`ServiceTrans.java:79-87`)
compute the *same* predicate — a non-empty intersection of references. So whenever this branch runs, the
pointer provably does **not** reference the current transformation; post-fix that means it genuinely belongs
to a different one, and deleting it would destroy a valid pointer. A guard of the form "delete it only if it
references this transformation" is therefore unreachable by construction. The orphan reported here is
entirely an artefact of the `file://` mismatch, which `canonicalLocation()` removes at source by preventing
the bogus conflict in the first place — so **leaving the pointer in place is the correct behaviour once the
root cause is fixed**. (Such a guard was implemented and unit-tested on 2026-09-18, shown to be dead code by
this reasoning, and reverted; `ae39a92` remains the complete fix.)

### Consequence

`DataServiceMetaStoreUtil.checkConflict()` scans `getServiceTransMap( serviceTrans )` (the global pointers)
and `ServiceTrans.isValid( repository )` only checks that the referenced `.ktr` still exists, so an orphan
pointing at a live `.ktr` survives forever:

* recreating a data service with a previously used name raises a bogus
  `DataServiceAlreadyExistsException` / "suggest edit" prompt — note `checkConflict()` line 299 tests
  `getServiceTransMap( serviceTrans ).containsKey( name )`, which is keyed on name only and is **not**
  filtered by `isReferenceTo`, so any same-named pointer anywhere in the metastore triggers it;
* a transformation that was saved with its data service and then reopened can become **impossible to save
  cleanly** (previous subsection); no answer to the prompt is a clean exit — "Yes" without a rename
  re-dirties the transformation and loops, "Yes" with a rename escapes only by renaming a service that was
  never in conflict (breaking JDBC/Carte clients) and still orphans the original pointer, and "No" deletes
  the data service while **leaving its pointer XML behind**;
* `getDataService( name, repository, metaStore )` can resolve the stale pointer, load the referenced `.ktr`
  and fail, or resolve to the wrong service — a service can inherit state belonging to a deleted namesake;
* the JDBC/Carte side lists services that no longer exist.

### Fix

Make a `FILE` location scheme-independent before it is used, in `ServiceTrans`:

* new `StorageMethod.canonicalPath( String )` maps `file:///a/b.ktr` -> `/a/b.ktr` (percent escapes decoded
  via `java.net.URI.getPath()`), and leaves plain paths and non-local VFS URIs (`hdfs://`, `s3://`, ...)
  untouched. No new dependency: `java.net.URI`, not `commons-vfs2`, which is not on `core`'s compile path.
* `StorageMethod.FILE.exists()` resolves through `canonicalPath()`; `load()` now reuses `exists()` instead of
  repeating its own `java.io.File` check.
* `Reference.equals()` / `hashCode()` compare canonical locations, so the two forms are interchangeable.

The existing save-time mechanism then does the reaping, exactly as it does on `master`. **Pre-existing
orphans are repaired automatically**: the next time their transformation is saved, the synchronizer finally
sees them and deletes them (verified — see section 7).

**The same two lines fix both failure modes**, which is why no separate change is needed for the save loop:

* `exists()` through `canonicalPath()` puts a `file://` pointer back into `allPublished`, so it can reach
  `toDelete` -> the **orphan leak** is fixed.
* `Reference.equals()` through `canonicalLocation()` makes a plain-path pointer match a normalised
  `file://` filename, so it lands in `publishedInTrans` instead of `nameConflicts` -> no
  `DataServiceAlreadyExistsException`, no `suggestEdit()`, no `setChanged()` -> the **save deadlock** is
  fixed, and the service is included in `toSave` so the pointer is rewritten with the current location.

Both are already implemented on `practitest_5375` (commit `ae39a92`, see section 6): `canonicalLocation()`
at `ServiceTrans.java:194-195` and `exists()` at `:213`.

### Rejected alternative: eager deletion on delete

The first attempt (commit later rewritten) deleted the pointer synchronously inside
`DataServiceMetaStoreUtil.deleteDataServiceElementAndCleanCache()`, via a
`deleteServiceTransReference( DataServiceMeta, TransMeta )` helper. It made the symptom disappear but was
**rejected as the wrong layer**:

* it treats the symptom, leaving the underlying `file://` matching bug in place — `isReferenceTo()`,
  `isValid()` and `sync()` all stay broken for URI locations, and orphans keep accumulating via every other
  route (rename, `suggestEdit`, crash before save);
* it **breaks the design invariant** that published pointers describe what is on disk. Deleting a data
  service is an in-memory edit until the transformation is saved; unpublishing it immediately means that
  closing without saving leaves a `.ktr` that still contains the data service while the pointer is gone, so
  the service silently stops being resolvable over JDBC/Carte;
* it cannot repair the orphans already on disk, whereas the canonical-path fix does.

## 6. Code Changes Applied

Repository `pdi-dataservice-ee-plugin`, branch **`practitest_5375`** (cut from `11.0`), module `core`.
Two local commits, one per defect. **Nothing pushed.**

### Commit 1 — `[PRACTITEST-5375] Fix stale Data Service step icon and right-click menu in Spoon`

`core/.../serialization/DataServiceMetaStoreUtil.java`

1. Added `private static volatile Cache<Integer, String> sharedStepCache` plus `getSharedStepCache()`
   (double-checked lazy init). The no-arg constructor uses it instead of allocating a private cache.
2. `initCache()` no longer appends `UUID.randomUUID()` to the cache name; it uses the fixed name
   `org.pentaho.di.trans.dataservice.serialization.DataServiceMetaStoreUtil` and reuses an already
   registered cache from the `PentahoCacheManager`. Removed the now-unused `java.util.UUID` import.
3. `save()` calls the new `invalidateStepCache( transMeta, stepName )` **before** `saveElement()`, so the
   negative marker cannot survive the write (`saveElement` -> `saveToCache` repopulates the positive entry).
4. `deleteDataServiceElementAndCleanCache()` replaces the conditional `stepCache.replace( key, name, "" )`
   with `invalidateStepCache(...)` (`stepCache.remove( key )`); the conditional form was a no-op whenever the
   cache held a negative marker, a different name, or no entry at all.
5. Added `protected void invalidateStepCache( TransMeta, String )`.

`core/.../DataServiceContext.java`

6. Added `private volatile DataServiceDelegate dataServiceDelegate` and memoized `getDataServiceDelegate()`
   with double-checked locking. Previously every call returned
   `DataServiceDelegate.withDefaultSpoonInstance( this )` — a new delegate **and a new named cache in the
   `PentahoCacheManager`** per invocation (unbounded resource leak, and the direct cause of the cache split).

Tests shipped in the same commit:

* `core/src/test/.../serialization/DataServiceMetaStoreUtilStepCacheTest.java` (new, 10 tests) —
  `getSharedStepCache()` is a singleton; `DataServiceMetaStoreUtil.getInstance()` and a `DataServiceFactory`
  subclass (the UI writer) share that exact instance; both `initCache()` branches (reuse an already
  registered cache vs create it from the default template); the three `invalidateStepCache()` null guards
  (`stepCache`, `transMeta`, `stepName`) and its removal path over every `createCacheKeys(...)` key;
  `save()` invalidating **before** `saveElement()` (`InOrder`); and
  `deleteDataServiceElementAndCleanCache()` removing instead of conditionally replacing.
* `core/src/test/.../DataServiceContextTest.java` — added `testGetDataServiceDelegateIsMemoized`.

### Commit 2 — `[PRACTITEST-5375] Match published ServiceTrans references stored as VFS URIs`

`core/.../serialization/ServiceTrans.java` (+44/-4)

7. Added `static String StorageMethod.canonicalPath( String )` — `file:///a/b.ktr` -> `/a/b.ktr` via
   `java.net.URI.getPath()` (decodes `%20`), other forms returned unchanged.
8. `StorageMethod.FILE.exists()` resolves through `canonicalPath()`; `load()` now delegates to `exists()`.
9. `Reference.equals()` / `hashCode()` compare `canonicalLocation()` (canonical for `FILE`, raw otherwise).
10. Added the `java.net.URI` import.

Tests shipped in the same commit:

* `core/src/test/.../serialization/ServiceTransFileLocationTest.java` (new, 15 tests) —
  `canonicalPath()` for a `file://` URI, an upper-case scheme, `%20` decoding, plain paths, non-local
  schemes, a `file:` URI with no path, `null`/empty, and the malformed-URI fallback;
  `FILE.exists()`/`FILE.load()` for both the URI and the plain-path form of a real file plus the
  missing/`null`/empty cases; and `Reference.equals()`/`hashCode()` across the two forms plus the identity,
  `null`, foreign-type, different-location, different-storage-method and non-`FILE` branches.

### Not changed (deliberately)

* Negative caching in `getDataServiceByStepName()` was kept — with a single shared cache plus explicit
  invalidation it is now correct, and removing it would change hot-path behaviour for Carte/JDBC.
* `DataServiceReferenceSynchronizer` and `SynchronizationListener` are untouched: the save-time reaping
  design is correct, it was only being fed unmatchable locations.
* Each commit carries its own tests, so the history stays bisectable: `mvn -pl core test` is green at
  commit 1 and at commit 2. Full `core` suite: **798 tests, 0 failures, 1 skipped** (a pre-existing
  `@Ignore`).

## 7. AI Reproduction / Verification Runbook

Deterministic steps for an agent to rebuild, deploy, run and verify. Paths are for this workstation.

```bash
# 0. Repos. The fix branch is `practitest_5375` in pdi-dataservice-ee-plugin (cut from 11.0);
#    the other repos stay on branch 11.0.
#    /Users/andjorge/github/pdi-dataservice-plugin          (11.0)
#    /Users/andjorge/github/pdi-dataservice-ee-plugin       (practitest_5375)  <- the fix lives here
#    /Users/andjorge/github/pdi-plugins-ee                  (11.0)
#    /Users/andjorge/github/pentaho-kettle                  (11.0)
#    /Users/andjorge/github/pdi-assemblies-ee               (11.0)

# 1. Build (online; -o/offline fails on maven-shade-plugin deps)
cd /Users/andjorge/github/pdi-dataservice-ee-plugin
mvn clean install -DskipTests

# 2. Back up the shipped jar ONCE, outside the plugin dir (a *.jar.orig inside it is ignored by the
#    plugin classloader, but keeping it outside avoids any ambiguity)
P=/Users/andjorge/pentaho/pdi/11.0.0.3/pdi-ee-client-11.0.0.3-312/data-integration/plugins/pdi-dataservice-ee-plugin
cp "$P/pdi-dataservice-ee-core-11.0.0.3-312.jar" <backup-dir>/pdi-dataservice-ee-core-11.0.0.3-312.jar.orig

# 3. Deploy the patched jar UNDER THE SHIPPED FILENAME (the classloader picks up every *.jar in the folder;
#    copying the SNAPSHOT jar alongside the original would load BOTH and give non-deterministic results)
cp /Users/andjorge/github/pdi-dataservice-ee-plugin/core/target/pdi-dataservice-ee-core-11.0.0.0-SNAPSHOT.jar \
   "$P/pdi-dataservice-ee-core-11.0.0.3-312.jar"

# 4. ALWAYS clear the debug log before each run
D=/Users/andjorge/pentaho/pdi/11.0.0.3/pdi-ee-client-11.0.0.3-312/data-integration
rm -f "$D/SpoonDebug.txt"

# 5. Launch Spoon (foreground, so Ctrl+C can stop it). SpoonDebug.sh asks two questions:
#    "Set logging level to Debug?" -> Y ; "Redirect console output to SpoonDebug.txt?" -> Y
cd "$D" && (sleep 2; echo "Y"; sleep 2; echo "Y"; sleep 30) | ./SpoonDebug.sh
#    Wait ~50s for the UI to come up.

# 6. Inspect the log
grep -iE "ERROR|Exception|Caused by" "$D/SpoonDebug.txt"
#    Benign/expected noise on a clean run:
#      SLF4J(E): ... org.apache.logging.slf4j.SLF4JServiceProvider not a subtype
#      SLF4J(W): No SLF4J providers were found. / Defaulting to no-operation (NOP) logger

# 7. Inspect the published pointers
ls -la "/Users/andjorge/.pentaho/metastore/pentaho/Data Service Transformation/"
```

### Manual actions to request from the user (Spoon is a GUI, these cannot be automated here)

Test transformation: `/Users/andjorge/Downloads/dataservices_11002.ktr`

**A — step badge + submenu state (primary bug)**

1. Open the `.ktr`.
2. Right-click the target step -> **Data Services**. Close the menu. *(Belt and braces only — it is not
   strictly required: the negative cache marker is already written when the canvas paints the step, and
   again by the right-click in step 3.)*
3. Right-click the step -> **Data Services** -> **New...**, use a name not present in
   `~/.pentaho/metastore/pentaho/Data Service Transformation/`, save the dialog.
4. **Without touching the View tree**, check: the bottom-left Data Service badge must appear on the step
   immediately, and the submenu must show **New...** disabled with **Edit... / Delete... / Test...** enabled.
5. Right-click -> **Data Services** -> **Delete...**: the badge must disappear immediately and **New...** must
   become enabled again.

*Pre-fix behaviour:* nothing changes at step 4, not even after saving the transformation; the badge only
appears after right-clicking on the service in **View -> Configurations -> Data Services** and opening its options menu (which routes through
`DataServiceTreeDelegateExtension` and the singleton cache).

**B — orphan pointer cleanup (secondary bug)**

> **The transformation must have been opened from disk at least once in this Spoon session.** A brand-new
> transformation that has only ever been saved keeps a plain `/...` filename and is reaped correctly even
> on an unpatched build, so it will not reproduce the defect. If in doubt, close the tab, reopen the `.ktr`
> and confirm with `grep -m1 "<filename>" <path>.ktr` that the value now starts with `file:///`.

1. Create a data service on the step.
2. **Save the transformation (Ctrl+S)** — required, the `ServiceTrans` pointer is only published on save.
   Verify `<serviceName>.xml` now exists under `.../Data Service Transformation/`.
3. Right-click the step -> **Data Services** -> **Delete...**
4. check: the bottom-left Data Service badge must disappear from the step
   immediately, and the submenu must show **New...** enabled with **Edit... / Delete... / Test...** disabled. Do **not** save.
5. The `<serviceName>.xml` file must still exist.
6. Save the transformation.
7. The `<serviceName>.xml` file must be gone.

*Pre-fix behaviour:* the file survives step 7 too — it is never removed, because the pointer's
`file://` location makes it invisible to `DataServiceReferenceSynchronizer.sync()` (section 5).

To confirm you are reproducing the pre-fix case, check the stored location:

```bash
grep -A2 transformation_location "/Users/andjorge/.pentaho/metastore/pentaho/Data Service Transformation/<name>.xml"
```

A `file:///...` value leaks on the unpatched build; a plain `/Users/...` value is reaped even unpatched. The
form is decided by SP-7071's `KettleVFS.normalizeFilePath()` in `TransFileListener.open()` — present from
11.0.0.2 onward, absent on `master` and on 11.0.0.0/11.0.0.1, which is why the defect does not show up
there (section 5).

**C — automatic repair of pre-existing orphans (secondary bug)**

1. Make sure `.../Data Service Transformation/` contains an orphan `<name>.xml` whose
   `transformation_location` is a `file:///...` URI pointing at a `.ktr` that no longer defines that service.
2. Open that exact `.ktr`, make any trivial change so it becomes dirty, and save it (Ctrl+S).
3. The orphan `<name>.xml` must be gone.

> Watch out when interpreting this test: only the orphans belonging to the transformation you actually saved
> are reaped. Cross-check the `.ktr` mtime against the pointer's `transformation_location` before concluding
> that a file "was not removed".

**D — unrecoverable "data service already exists" save loop (secondary bug, second failure mode)**

This scenario needs the pointer published in the **plain** form and the transformation reopened, so the
order matters and it cannot be started from an existing `.ktr`.

1. In Spoon, create a **new** transformation, add a step that can feed a data service, and create the data
   service on it in the same session.
2. Save (Ctrl+S) to a new file, e.g. `/tmp/loop_demo.ktr`. Verify both paths are plain:

   ```bash
   grep -m1 "<filename>" /tmp/loop_demo.ktr
   grep -A2 transformation_location "$HOME/.pentaho/metastore/pentaho/Data Service Transformation/<name>.xml"
   ```

   Both must start with `/`, not `file:///`.
3. Close the transformation tab, then reopen `/tmp/loop_demo.ktr`.
4. Make any trivial change so the tab is marked dirty.
5. **Ctrl+S.**

*Pre-fix behaviour (confirmed empirically on 11.0.0.2+):* a dialog appears reading *Data Service
"&lt;name&gt;" already exists. Do you want to edit this data service?*

* Clicking **Yes** opens the data service edit dialog; pressing **OK** *without changing the name* returns
  to a **still-dirty** transformation, and pressing Ctrl+S reproduces the same dialog — indefinitely. The
  transformation can never be brought to a clean state. (Renaming the service in that dialog does end the
  loop — do **not** do it in this scenario, it changes what you are measuring; see Annex C.)
* Clicking **No** takes `DataServiceDelegate.suggestEdit()`'s `else` branch: the data service is **deleted**
  from the transformation (check the step badge), but its pointer XML under
  `.../Data Service Transformation/` is **left behind**, because `deleteDataServiceElementAndCleanCache()`
  only removes the embedded element and `sync()`'s `toDelete` is empty in this state. Verify with:

  ```bash
  ls "$HOME/.pentaho/metastore/pentaho/Data Service Transformation/"
  ```

  The `<name>.xml` file is still present, and no later save will ever remove it.

*Expected (and post-fix):* the save completes silently, the tab goes clean, no dialog appears, and the
pointer's `transformation_location` is rewritten to the current `file:///...` value.

### Result of the 2026-09-18 run of scenario D (patched jar)

Verified on **11.0.0.3-312** with the `ae39a92` jar deployed. The save after reopening produced **no dialog**
and the tab went clean; the data service still executed correctly. Spoon's log shows the two saves with no
exception between them:

```
21:09:39 - Data Services - Saving 'dloop2' to 'loop_demo2'   <- create + first save (pointer written plain)
21:12:10 - Data Services - Saving 'dloop2' to 'loop_demo2'   <- after close/reopen/dirty; no exception
```

The decisive evidence is the pointer file, not the absence of the dialog: `dloop2.xml` ended the run with
`transformation_location` = `file:///tmp/loop_demo2.ktr` and an mtime of **21:12**. It had been written in
the plain `/tmp/...` form at 21:09, so the reopened save **rewrote** it — which only happens when the service
is in `toSave`. Pre-fix it would have been classified as a `nameConflict`, excluded from `toSave`, and left
untouched with its 21:09 mtime.

> A first attempt at this scenario was discarded as invalid: an extra *Save As* had also written the same
> data service to `~/Downloads/dataservices_11003_dloop.ktr`, so the two
> `DataServiceAlreadyExistsException`s it logged were a **genuine** cross-file duplicate rather than the
> defect. Archived under `scenarioD/run1-contaminated/`. Note also that creating a data service requires the
> transformation to have been saved first (`DataServiceDelegate.NewTransChanged.Message`), so the scenario
> necessarily involves two saves to the *same* path.
>
> That discarded run did expose a **separate, still-open issue**, later confirmed by a controlled test and
> written up as **Annex C**: for a *genuine* cross-file duplicate the prompt **does** appear, but answering
> it never resolves anything, because **both** branches of `DataServiceDelegate.suggestEdit()` call
> `serviceTrans.setChanged()` — the transformation is dirty again after every save and the prompt returns
> indefinitely. `canonicalLocation()` cannot help there: the conflict is real. (An earlier reading of this
> run — that the conflict raised *no* prompt at all because `suggestEdit()` was deferred through
> `syncExec` — was retracted in v3.3.0; see the changelog.)

### Full lifecycle re-test on the patched jar (2026-09-18, restarted session)

After restarting Spoon (see the hot-swap warning in section 8), the whole data service lifecycle was walked
end to end on **11.0.0.3-312 + `ae39a92`**: create -> save -> close -> reopen -> dirty -> save -> delete ->
recreate. Every step behaved correctly and **the session log contains zero errors and zero exceptions** - no
`DataServiceAlreadyExistsException`, no save loop, no prompt.

Final on-disk state, which is the part that matters:

| Artefact | State | Meaning |
|---|---|---|
| `dloop2.xml` | absent | the delete reaped the pointer even though its location was `file:///...` - the orphan leak is fixed |
| `loop_demo2.ktr` | zero `dloop2` *and* zero `dloop3` references | the embedded element was removed too; no residue |
| `dloop3.xml` | `file:///tmp/loop_demo3.ktr`, mtime 21:28 | the recreated service republished cleanly under the normalised form |
| `loop_demo3.ktr` | `<filename>file:///tmp/loop_demo3.ktr`, 2 `dloop3` refs | pointer and transformation agree |

Pointer and transformation ended in agreement in both directions: nothing left behind for the deleted
service, and a correct, matching pointer for the recreated one. This exercises both failure modes of the
secondary defect (the invisible-orphan leak and the `nameConflicts` save loop) plus defect A's create/delete
badge path in a single pass.

### Result of the 2026-09-17 run

Scenarios A-C passed with the patched jar, and `SpoonDebug.txt` contained no errors beyond the SLF4J NOP
warnings:

* **A** — badge and submenu update immediately on create and on delete, with no View-tree interaction.
* **B** — the pointer is published on save and removed on the save that follows the delete.
* **C** — two pre-existing `file://` orphans were repaired by simply saving their transformations:
  `t8` (referencing `dataservices_11100_1.ktr`) and `t9` (referencing `dataservices_11002.ktr`).
  `.../Data Service Transformation/` ended the session containing only `.type.xml`.

## 8. Files Modified Outside the Git Repos (restore points)

Anything below is **not** under version control. Restore these to get back to a pristine 11.0.0.3 install.

### PDI installation — `/Users/andjorge/pentaho/pdi/11.0.0.3/pdi-ee-client-11.0.0.3-312/data-integration/`

| Path | Change | How to restore |
|---|---|---|
| `plugins/pdi-dataservice-ee-plugin/pdi-dataservice-ee-core-11.0.0.3-312.jar` | **Overwritten** with the locally built `core/target/pdi-dataservice-ee-core-11.0.0.0-SNAPSHOT.jar` (~565 KB; original 568,161 bytes) | Copy the `.orig` backup back over it |
| `SpoonDebug.txt` | Deleted before every run, recreated by `SpoonDebug.sh` | Disposable; delete it |

Backup of the original jar for this session:
`/Users/andjorge/.copilot/session-state/0b28d23a-b656-44d8-9f58-8f04348c2a6f/files/pdi-dataservice-ee-core-11.0.0.3-312.jar.orig`

Re-deployed on 2026-09-18 for scenario D, rebuilt from a clean `ae39a92` working tree (565,740 bytes).
Second backup of the pristine 11.0.0.3 jar:
`~/.copilot/session-state/8c340e03-f402-454e-9617-691cc0436220/files/scenarioD/pdi-dataservice-ee-core-11.0.0.3-312.jar.orig`

> The rest of `plugins/pdi-dataservice-ee-plugin/` (`lib/`, `Data Service JDBC Driver/`, `pluginmeta.json`)
> was **not** touched. Only the `core` module changed, so only that one jar needs redeploying.

### User state — `/Users/andjorge/.pentaho/` and `/Users/andjorge/.kettle/`

| Path | Note |
|---|---|
| `~/.pentaho/metastore/pentaho/Data Service Transformation/*.xml` | One `ServiceTrans` pointer per published data service. The 9 pre-fix orphans (`2test`, `2ttest`, `4test`, `5test`, `test 11 0 0 2`, `test 11 0 0 2 v3`, `test post`, `testtt`, `tttest`) were **deleted manually on 2026-09-17** to establish a clean baseline (backup: `~/.copilot/session-state/0b28d23a-b656-44d8-9f58-8f04348c2a6f/files/orphan-servicetrans-backup/`). The later orphans `t8`/`t9` were then reaped **automatically** by the commit-2 fix. The folder now holds only `.type.xml`. Never delete `.type.xml`. |
| `~/.pentaho/metastore/pentaho/Database connection/`, `Local VFS Connection/`, `SharedDimensionGroup/` | Not touched |
| `/Users/andjorge/Downloads/dataservices_11002.ktr`, `dataservices_11100_1.ktr` | Modified by the tests — the embedded metastore gains/loses `DataServiceMeta` entries on save. Keep pristine copies before testing. |
| `~/.kettle/repositories.xml` | Read only |

> **Never redeploy the jar while Spoon is running.** The JVM keeps an open handle on the original file, so
> overwriting it invalidates the cached zip directory: already-loaded **classes** keep working, but
> **resource** lookups start failing. Observed on 2026-09-18 — after a 21:16 hot-swap, deleting a data
> service still worked while *Data Services -> New...* silently did nothing, logging
> `XulException: Can not locate Xul document [.../ui/xul/dataservice-dialog.xul]` from
> `DataServiceDialog.loadXul()`. The jar was intact (all 12 `.xul` entries present); restarting Spoon cleared
> it. Always stop Spoon, swap the jar, then relaunch.

### Clean-baseline checklist between test rounds

1. **Stop Spoon first**, then restore or redeploy `pdi-dataservice-ee-core-11.0.0.3-312.jar`.
2. `rm -f "$D/SpoonDebug.txt"`.
3. Delete every `*.xml` (not `.type.xml`) under `~/.pentaho/metastore/pentaho/Data Service Transformation/`.
4. Restore pristine copies of the test `.ktr` files (`dataservices_11002.ktr`, `dataservices_11100_1.ktr`).

## 9. Remaining Work

* ~~Unit/regression tests~~ — **done**, see section 6. Each commit contains the tests covering the code it
  changes, and the full `core` suite passes (798 tests). Coverage of the changed/added lines is complete:
  every new branch (shared-cache lazy init, both `initCache()` paths, all three `invalidateStepCache()`
  guards, `canonicalPath()`'s URI / non-URI / no-path / null-empty / exception paths, both
  `Reference.equals()` location forms) has a dedicated assertion.
* Optional extra coverage, not required by the fix: `DataServiceStepMenuExtensionTest` /
  `TransPainterStepExtensionPointPluginTest` end-to-end assertions that a service saved through the delegate
  is immediately visible via the singleton util, and a `DataServiceReferenceSynchronizerTest` case for a
  `file://` published location.
* Consider whether `createCacheKeys()` hashing `transMeta.getName()` should hash the filename/`ObjectId`
  instead — renaming a transformation currently orphans every cached entry.
* A one-off cleanup of pre-existing orphans is **no longer needed**: commit 2 makes the existing save-time
  synchronizer see them again, so they are removed the next time their transformation is saved (verified).
* `ServiceTrans.references()` still persists `TransMeta.getFilename()` verbatim, so the metastore keeps
  mixing `file://` and plain-path locations. Comparison is now form-independent, but normalising at write
  time would be tidier — deferred because it changes the on-disk format.
* **Raise the asymmetry with the SP-7071 author.** `TransFileListener.open()` normalises the filename to
  `file://` but `Spoon`'s save/save-as path does not, so the same transformation carries two different
  filename forms depending on how it entered the session — and now persists whichever one it happened to
  have into `<transformation><info><filename>`. The `ServiceTrans` fix in this branch makes data services
  immune, but any other consumer of `TransMeta.getFilename()` that assumes a plain OS path has the same
  latent bug. Normalising consistently on both paths (or not persisting `<filename>` at all for local
  files) would remove the class of defect rather than one instance of it.
* **Separate, unfixed defect: the prompt loop also happens for *genuine* duplicates, and survives this
  branch's fix.** Verified on 2026-09-18 with `ae39a92` deployed — see **Annex C**. Saving a transformation
  whose data service name is already published by a *different* transformation correctly raises
  `DataServiceAlreadyExistsException` and correctly prompts, but **both** branches of
  `DataServiceDelegate.suggestEdit()` call `serviceTrans.setChanged()`, so the transformation is dirty again
  after every save and the prompt returns indefinitely. `canonicalLocation()` cannot help here: the conflict
  is real. **Renaming the data service in the edit dialog does resolve it** — the loop only persists while
  the duplicate name is kept — but nothing tells the user that; accepted for now, see the scope marker in
  Annex C. Needs its own ticket.
* Decide whether SP-7071 should also be assessed for `master`: the commit is not on `master`, so `master`
  currently lacks the Carte remote-execution fix it delivered. Whoever forward-ports it must carry the
  `ServiceTrans` canonical-path fix with it, or `master` will acquire this defect.

## 10. Cross-Version Evaluation of Delete Behaviour (2026-09-18)

Question asked: **does the behaviour of "delete a data service" differ between PDI versions**, specifically
whether the `ServiceTrans` pointer XML under `~/.pentaho/metastore/pentaho/Data Service Transformation/`
is removed?

**Answer: no. 11.0.0.2, 11.0.0.3 and 11.1 behave identically — the pointer file always leaks.** There is no
version divergence between them, and the defect is confirmed present in the current 11.1 line.

> **Scope correction (see section 5).** Every run below opened a pre-existing `.ktr` from disk, i.e. they are
> all **reopen** cases. Follow-up investigation established that the variable that actually decides the
> outcome is not the PDI version and not the dialog, but **whether the transformation was opened from disk at
> least once**: `TransFileListener.open()` normalises the filename to `file://` (SP-7071, 11.0.0.2+), while a
> brand-new transformation saved for the first time keeps the plain path and **is** reaped correctly. The
> matrix below is therefore a matrix of reopen cases, and the version boundary is 11.0.0.1 -> 11.0.0.2,
> not 11.0.x -> 11.1.

### Builds under test (all with their **original, unpatched** `pdi-dataservice-ee-plugin`)

| PDI | Install root | Plugin jar |
|---|---|---|
| 11.0.0.2 | `/Users/andjorge/pentaho/pdi/11.0.0.2/pdi-ee-client-11.0.0.2-294/data-integration` | `pdi-dataservice-ee-core-11.0.0.2-294.jar` (568 163 B) |
| 11.0.0.3 | `/Users/andjorge/pentaho/pdi/11.0.0.3/pdi-ee-client-11.0.0.3-312/data-integration` | `pdi-dataservice-ee-core-11.0.0.3-312.jar` (568 161 B) |
| 11.1.0.0-SNAPSHOT (build 20260916.215745-335) | `/Users/andjorge/pentaho/pdi/11.1.0.0-SNAPSHOT/20260916/pdi-ee-client-11.1.0.0-20260916.215745-335/data-integration` | `pdi-dataservice-ee-core-11.1.0.0-SNAPSHOT.jar` (568 182 B) |

Unpatched status verified by class inspection: none of the three jars contains
`ServiceTrans$StorageMethod.canonicalPath` or `DataServiceMetaStoreUtil.sharedStepCache` /
`invalidateStepCache`.

### Static pre-check: the plugin code is byte-identical across the three versions

Extracting and diffing

* `org/pentaho/di/trans/dataservice/serialization/**`
* `org/pentaho/di/trans/dataservice/ui/**`
* `org/pentaho/di/trans/dataservice/DataServiceContext.class`

from the three jars gives **zero differing files** (`diff -rq`, 11.0.0.2 vs 11.0.0.3 vs 11.1). Spoon's
open-file wiring is also identical — `ui/menubar.xul` binds `file-open` to `spoon.openFileNew()` in all
three. Therefore any behavioural difference could only come from PDI core, via the form of
`TransMeta.getFilename()`. That is exactly what the test matrix probed.

### Test setup

* Baseline cleared: the five orphan pointers left over from the previous session (`1102_1`, `1102_2`,
  `111_1`, `111_2`, `11_1` — all `file:///...` locations) were backed up and deleted, leaving only
  `.type.xml`. Backup: `~/.copilot/session-state/8c340e03-.../files/baseline-backup/servicetrans/`.
* Six pristine transformations were generated from a service-free copy of `dataservices_11002.ktr`, each
  with its internal `<name>` matched to its filename, under `/Users/andjorge/Downloads/dstest/`:
  `ds_1102_vfs`, `ds_1102_os`, `ds_1103_vfs`, `ds_1103_os`, `ds_1110_vfs`, `ds_1110_os`. Both steps
  present: `Generate Rows` (used for every service) and `Delay row`.
* Per run: right-click the step **before** creating the service (redundant belt-and-braces poisoning of the
  cache — the painter does it anyway), create the service,
  `Ctrl+S`, verify the pointer, delete the service, `Ctrl+S`, re-verify.

### Result matrix

| # | PDI | How the `.ktr` was opened | `transformation_location` written | Badge on create | Badge on delete | Pointer after delete + save |
|---|---|---|---|---|---|---|
| 1 | 11.0.0.2 | VFS browser (`File > Open...`) | `file:///Users/andjorge/Downloads/dstest/ds_1102_vfs.ktr` | **not painted** (View-tree workaround needed) | disappears immediately | **`t1102v.xml` survives — LEAK** |
| 2 | 11.0.0.2 | Drag & drop from Finder onto canvas | `file:///.../ds_1102_os.ktr` | n/a | n/a | **`t1102o.xml` survives — LEAK** |
| 3 | 11.0.0.2 | `./spoon.sh -file=/.../ds_1102_os.ktr` | `file:///.../ds_1102_os.ktr` | n/a | n/a | **`t1102o.xml` survives — LEAK** |
| 4 | 11.0.0.3 | `./spoon.sh -file=/.../ds_1103_os.ktr` | `file:///.../ds_1103_os.ktr` | n/a | n/a | **`t1103o.xml` survives — LEAK** |
| 5 | 11.0.0.3 | VFS browser | `file:///.../ds_1103_vfs.ktr` | n/a | n/a | **`t1103v.xml` survives — LEAK** |
| 6 | 11.1 | `./spoon.sh -file=/.../ds_1110_os.ktr` | `file:///.../ds_1110_os.ktr` | n/a | n/a | **`t1110o.xml` survives — LEAK** |
| 7 | 11.1 | VFS browser | `file:///.../ds_1110_vfs.ktr` | n/a | n/a | **`t1110v.xml` survives — LEAK** |

In every one of the seven runs the embedded `DataServiceMeta` **was** correctly removed from the `.ktr`
(the `Embedded MetaStore Elements` attribute group ends up with no keys), while the global pointer file
**was not**. Final state of `.../Data Service Transformation/`: `t1102o.xml`, `t1102v.xml`, `t1103o.xml`,
`t1103v.xml`, `t1110o.xml`, `t1110v.xml` — six orphans, one per test transformation (runs 2 and 3 reused the
same `.ktr`, and therefore the same pointer name). No `ERROR`/`Exception` in any of the
three Spoon logs (only the known SLF4J NOP warnings); each closed with a normal `Spoon has ended.`

The logs are also informative by omission: they contain the `Data Services - Saving '<name>' to '<trans>'`
line for every creation, but **no corresponding entry, warning or error on delete**. The pointer is not
failing to be deleted — deletion of the global element is never even attempted, because
`DataServiceReferenceSynchronizer.sync()` never puts the pointer in its `toDelete` set.

### Key findings

1. **No behavioural divergence between 11.0.0.2, 11.0.0.3 and 11.1.** For a reopened transformation the
   defect is uniform. 11.1 is *not* fixed; the regression carries forward into the current development line.
2. **The open method is not a variable.** All three entry points — the Pentaho VFS browser, Finder
   drag & drop, and the `-file=` command-line argument — put the **`file://` URI form** into
   `TransMeta.getFilename()`, and therefore into `ServiceTrans`'s `transformation_location`. Follow-up
   investigation explains why: they all funnel through `TransFileListener.open()`, which SP-7071 changed to
   call `KettleVFS.normalizeFilePath( fname )`. This **strengthens** the section 5 root cause:
   `StorageMethod.FILE.exists()`'s `new File( location )` check is always false for these values, so
   `DataServiceReferenceSynchronizer.sync()` can never see, let alone reap, such a pointer.
3. **The plain-path form is still reachable — it just never survives an open.** A transformation created in
   Spoon and saved for the first time keeps `/Users/...`, and its pointers *are* reaped correctly. The
   leak starts the moment that transformation is closed, reopened and saved again. See section 5.
4. **`master` differs because SP-7071 is not on it**, not because of dialog history.
   `git branch -a --contains f560af820d6` lists 26 branches but not `master`, so `TransFileListener` there
   still does `transMeta.setFilename( fname )` and the plain path satisfies `new File(...)`. This supersedes
   the v2.0.0 "historical artefact of older, non-VFS file dialogs" explanation, and still confirms that the
   eager-delete workaround was correctly rejected in favour of the `canonicalPath()` fix.
5. **Defect A (stale step badge) also reproduces on 11.0.0.2** (run 1): after `New...` the badge was not
   painted and the View-tree workaround was required. Consistent with the byte-identical `ui/` classes, so
   it is present on all three versions.
6. Because every reopened-transformation pointer leaks and `DataServiceMetaStoreUtil.checkConflict()` scans
   those pointers, **name reuse breaks over time on a stock 11.0.0.2+ install**: recreating a data service
   with a previously used name raises a bogus `DataServiceAlreadyExistsException` / "suggest edit" prompt.

### Follow-up investigation (2026-09-18, evening): create vs reopen

Prompted by the observation that `<transformation><info><filename>` is a **new** element, the corpus of
`.ktr` files on this workstation was inspected. It splits cleanly three ways and the `<filename>` value
predicts the pointer form exactly:

| `.ktr` | `<info><filename>` | Written by |
|---|---|---|
| `dataservices.ktr`, `dataservices_11000.ktr`, `dataservices_11000_1.ktr`, `dataservices_11001.ktr`, `dataservices_11001_1.ktr` | **absent** | 11.0.0.0 / 11.0.0.1 — no SP-7071 |
| `dataservices_11002.ktr`, `dataservices_11003.ktr`, `dataservices_11100.ktr`, `dataservices_11100_1.ktr`, `dataservices_11103_1.ktr`, `dataservices_2.ktr` | `/Users/...` (plain) | 11.0.0.2+ — created and saved, never reopened |
| `dataservices_11002_1.ktr`, `dataservices_11100_2.ktr`, `dataservices_11100_3.ktr`, `dataservices_1.ktr`, and all six `dstest/ds_*.ktr` | `file:///Users/...` | 11.0.0.2+ — reopened, then saved |

Matching pointers, confirming `transformation_location` == `<info><filename>` verbatim:

```
bla11003.xml -> /Users/andjorge/Downloads/dataservices_11103_1.ktr          (plain  -> reaped)
blo.xml      -> file:///Users/andjorge/Downloads/dataservices_11100_3.ktr   (URI    -> leaks)
```

Build inspection then pinned the boundary precisely (`unzip -p <jar> <class> | strings | grep
normalizeFilePath`): `KettleVFS.normalizeFilePath` and its call from `TransFileListener` are **absent** in
11.0.0.0 (237) and 11.0.0.1 (259), and **present** in 11.0.0.2 (294), 11.0.0.3 (312) and
11.1.0.0-SNAPSHOT (335) — exactly the builds in which `<filename>` appears and the pointer leaks.

**Corrected behavioural rule:**

* Create a transformation, create a data service, save — **without ever closing and reopening it**: the path
  stays plain `/...` in both the `.ktr` and the pointer, and deleting the data service + saving **does**
  remove the pointer XML. This holds on all of 11.0.0.0 - 11.1.
* Close it, reopen it, (re)create the data service, save: both paths become `file:///...` and deleting the
  data service **no longer** removes the pointer XML. This holds on 11.0.0.2+ and is the defect.
* The same applies to the delete-then-recreate order — *delete the data service -> close -> reopen ->
  create the data service -> save* also yields `file:///...`, because the form is decided solely by whether
  the current filename came through `TransFileListener.open()`, not by anything the data service does.
* **Mixing the two orders is what produces the save deadlock.** If the pointer was published in the plain
  form (created and saved in one session) and the transformation is subsequently reopened, the pointer is
  visible to the synchronizer but no longer matches it, which raises a spurious
  `DataServiceAlreadyExistsException` on every save. See section 5, "Third symptom", and scenario D in
  section 7.

Note that the six `dstest/ds_*.ktr` fixtures were generated by script from a copy of `dataservices_11002.ktr`
and still carried that file's plain `<filename>` value. Spoon ignored it — `TransMeta.loadXML()` only adopts
the persisted `<filename>` when the in-memory filename is empty — and wrote the normalised URI for the
actual path instead. This independently confirms that `<info><filename>` is a *record* of
`TransMeta.getFilename()`, not an input to it, in the Spoon open path.

### Evidence archive

`~/.copilot/session-state/8c340e03-f402-454e-9617-691cc0436220/files/`

| Path | Contents |
|---|---|
| `baseline-backup/servicetrans/` | The five pre-run orphans removed to establish the clean baseline |
| `ktr-pristine/` | The six generated test transformations, before any run |
| `run-2026-09-18/` | Post-run copies of the six `.ktr` files, the resulting `servicetrans-after/` folder, and the three Spoon logs (`spoon1102_file.log`, `spoon1103.log`, `spoon1110.log`) |

### Cleanup still owed on this workstation

The six orphan pointers (`t1102o`, `t1102v`, `t1103o`, `t1103v`, `t1110o`, `t1110v`) and
`/Users/andjorge/Downloads/dstest/` are **left in place deliberately** as the reproduction corpus for
re-testing the fix. To re-run the verification of section 5's automatic repair, deploy the patched
`core` jar and simply re-save each of the six transformations; each pointer must then disappear.
To discard the corpus instead:

```bash
rm -rf /Users/andjorge/Downloads/dstest
find "/Users/andjorge/.pentaho/metastore/pentaho/Data Service Transformation" \
  -name 't11*.xml' -delete        # never delete .type.xml
```

## 11. Changelog

| Version | Date | Author | Change |
|---|---|---|---|
| 1.0.0 | 2026-09-17 | @afrjorge (investigation) | Initial analysis: identified the split `stepCache` between the singleton `DataServiceMetaStoreUtil` and per-instance `DataServiceDelegate` as root cause of the missing step badge and wrong Data Services submenu state; documented the View-tree workaround path and the proposed fixes. |
| 1.1.0 | 2026-09-17 | @afrjorge (implementation) | Implemented the fix in `pdi-dataservice-ee-plugin` `core` (shared step cache, memoized `DataServiceDelegate`, explicit cache invalidation on save/delete). Added section 5 (orphan `ServiceTrans` reference files + fix), section 6 (code changes applied), section 7 (AI reproduction/verification runbook), section 8 (files modified outside the git repos / restore points) and section 9 (remaining work). Both scenarios verified manually against PDI 11.0.0.3 with a clean `SpoonDebug.txt`. |
| 1.1.1 | 2026-09-17 | @afrjorge | Spoon closed; the 9 pre-fix orphan `ServiceTrans` pointer files were deleted (backed up) to establish a clean baseline. Section 8 updated accordingly. |
| 2.0.0 | 2026-09-17 | @afrjorge | **Root cause of defect 2 corrected.** Investigating why `master` reaps the pointer on save while `11.0` does not showed the data-service sources are functionally identical on both branches; the real cause is that `ServiceTrans.StorageMethod.FILE.exists()` used `new File( location )`, which is always false for the `file://` URIs that 11.0's VFS-aware file dialogs put in `TransMeta.getFilename()`, so `DataServiceReferenceSynchronizer.sync()` filtered those pointers out and could never delete them; `Reference.equals()` had the same raw-string problem. The eager-delete fix was **rejected and rewritten** as a canonical-path fix in `ServiceTrans` (see section 5, "Rejected alternative"). Section 5 rewritten, sections 6-9 updated, scenario C added to section 7. Branch references changed from `11.0` to `practitest_5375`. Verified: pre-existing orphans are now repaired automatically on save. |
| 2.1.0 | 2026-09-18 | @afrjorge | Unit tests added to **both** commits (history rewritten so each commit carries its own tests and stays bisectable): `DataServiceMetaStoreUtilStepCacheTest` (new) and a memoization test in `DataServiceContextTest` for commit 1; `ServiceTransFileLocationTest` (new) for commit 2. Full `core` suite green (798 tests, 0 failures). Sections 6 and 9 updated. |
| 2.2.0 | 2026-09-18 | @afrjorge | **Cross-version evaluation added (new section 10; old section 10 renumbered to 11).** Ran the create/delete cycle on unpatched 11.0.0.2, 11.0.0.3 and 11.1.0.0-SNAPSHOT across three open methods (VFS browser, Finder drag & drop, `spoon.sh -file=`) — 7 runs. Result: **no divergence, the `ServiceTrans` pointer leaks in all 7**, so 11.1 is not fixed either. Also established that the `pdi-dataservice-ee-core` `serialization/`, `ui/` and `DataServiceContext` classes are **byte-identical** across the three builds, and that every 11.x open path yields a `file://` URI in `TransMeta.getFilename()` — confirming the section 5 root cause and that no plain-path form remains reachable in 11.x. Defect A (stale badge) also reproduced on 11.0.0.2. |
| 2.2.1 | 2026-09-18 | @afrjorge | Frontmatter `title`/`description`/`tags` and the H1 widened from "PDI 11.0" to "PDI 11.0 - 11.1" to match section 10's scope. Repaired a blank line that was splitting the changelog table in two. Evidence archive re-snapshotted after the final run so it includes `t1110v.xml`. |
| 2.2.2 | 2026-09-18 | @afrjorge | Section 10: added the log-omission corroboration — the Spoon logs carry a `Data Services - Saving '<name>' to '<trans>'` line per creation but **no entry, warning or error on delete**, showing the pointer deletion is never attempted rather than attempted-and-failed. All three sessions exited cleanly (`Spoon has ended.`). |
| 2.3.0 | 2026-09-18 | @afrjorge | Added **Annex A** (Jira draft for defect 1, the stale step badge / right-click menu, derived from sections 1-4) and **Annex B** (Jira draft for defect 2, the orphan `ServiceTrans` pointer files, derived from sections 5 and 10). Each annex carries Summary, Affects, Description, Preconditions, Steps to reproduce, Actual result and Expected result, paste-ready for Jira. No analysis changed. |
| 3.0.0 | 2026-09-18 | @afrjorge | **Trigger of defect 2 identified; the v2.0.0 "VFS-aware file dialogs / historical artefact" explanation is superseded and removed.** The `file://` form comes from `pentaho-kettle` commit `f560af820d6` *fix[SP-7071]: Preserve trans/job filename in Carte remote execution*, which added `KettleVFS.normalizeFilePath()` and changed `TransFileListener.open()` to `setFilename( KettleVFS.normalizeFilePath( fname ) )`. Consequences documented: a **create-and-save** transformation keeps the plain `/...` path and its pointers **are** reaped, while **close -> open -> save** turns the path into `file:///...` and the pointer leaks. The same commit persists `<transformation><info><filename>`, giving a one-command diagnostic. Version boundary re-established by build inspection as **11.0.0.1 -> 11.0.0.2** (`normalizeFilePath` absent in 11.0.0.0/237 and 11.0.0.1/259, present in 294/312/11.1), and `master` differs simply because SP-7071 is not on it. Section 5 root-cause subsection rewritten with the affected-version and diagnostic tables; section 7 scenario B gained the mandatory reopen precondition; section 9 gained two follow-up items; section 10 gained a scope correction, a "create vs reopen" subsection and corrected key findings 2-4 (the claim that no 11.x entry point can produce a plain path was **wrong**); Annex B rewritten end to end, including a passing contrast case. |
| 3.1.0 | 2026-09-18 | @afrjorge | **Second failure mode of defect 2 documented: an unrecoverable "data service already exists" save loop.** When the pointer was published in the plain `/...` form (transformation *and* data service created in one session and saved) and the transformation is later reopened, its location passes `new File(...).exists()` so it enters `allPublished`, but no longer string-matches the normalised filename so it is absent from `publishedInTrans` — `DataServiceReferenceSynchronizer.sync()` therefore classifies it as a `nameConflict`, excludes it from `toSave` (so the mismatch can never self-heal) and raises `DataServiceAlreadyExistsException`. `SynchronizationListener.contentSafe()` routes that to `DataServiceDelegate.suggestEdit()`, whose **both** branches call `serviceTrans.setChanged()`, re-dirtying the transformation after every save and looping forever; the "No" branch additionally **deletes the data service** (data loss). Added to section 5 a two-failure-mode truth table and a "Third symptom" subsection, plus a note that the existing `canonicalLocation()`/`exists()` fix in `ae39a92` repairs both modes; added scenario **D** to the section 7 runbook; extended the section 10 create-vs-reopen rules with the delete-then-recreate order and the mixed-order deadlock; extended Annex B with a "Second failure mode" section, its own repro steps and an updated Summary. Also confirmed `checkConflict()` line 299 matches on name only, without an `isReferenceTo` filter. |
| 3.1.1 | 2026-09-18 | @afrjorge | Save loop **confirmed empirically on 11.0.0.2+** (previously derived from code analysis only), and the behaviour of the prompt's **No** answer corrected and expanded. Answering No does not merely delete the data service: `deleteDataServiceElementAndCleanCache()` (`DataServiceMetaStoreUtil.java:330-333`) removes only the **embedded** `DataServiceMeta`, while the global pointer XML is deleted solely by `sync()`'s `toDelete` list, which is built from `publishedInTrans` - empty in precisely this state. The branch therefore **destroys the data service and leaves the orphan pointer behind**, without removing the duplicate it claims to resolve, and once the embedded service is gone nothing can republish or reap that pointer. Section 5's "Third symptom" gained this analysis with both code excerpts; the Consequence bullet now states that both dialog answers are dead ends; section 7 scenario D gained the observed dialog text, per-answer outcomes and an `ls` verification step, and the "Result of the 2026-09-17 run" heading now notes that D postdates that run and is not yet re-verified against the patched jar; Annex B's second-failure-mode section and its Actual result were updated to match. |
| 3.2.0 | 2026-09-18 | @afrjorge | **Scenario D verified empirically against the patched jar on 11.0.0.3-312**: after close/reopen/dirty the save produced no dialog, the tab went clean and the data service still executed; the decisive evidence recorded is that `dloop2.xml` was *rewritten* from the plain form to `file:///tmp/loop_demo2.ktr` at the reopened save, proving the service reached `toSave` instead of `nameConflicts`. A first attempt was discarded as invalid (a stray *Save As* had created a real cross-file duplicate) and archived under `scenarioD/run1-contaminated/`. Section 5 gained a subsection explaining **why the destructive "No" branch is deliberately left unchanged**: `nameConflicts` and `ServiceTrans.isReferenceTo()` evaluate the same reference-intersection predicate, so a "delete the pointer only if it references this transformation" guard is unreachable by construction, and post-fix a non-matching pointer genuinely belongs to another transformation - leaving it is correct, the orphan being purely an artefact of the mismatch that `canonicalLocation()` removes at source. Such a guard was built, unit-tested and then reverted; `ae39a92` remains the complete fix and the deployed jar was rebuilt from it. Section 9 gained a **new, separate open defect** found during the run: genuine duplicates raise no prompt at all, because `suggestEdit()` is dispatched through `syncExec` and its `!hasChanged()` guard is false by the time it executes. |
| 3.2.1 | 2026-09-18 | @afrjorge | Section 8 gained a **test-procedure hazard warning**: redeploying the plugin jar under a running Spoon corrupts resource loading for the rest of the session. After a 21:16 hot-swap, *Data Services -> New...* silently did nothing while delete still worked, because the JVM's open handle on the replaced jar invalidates its cached zip directory - loaded classes keep working, resources do not (`XulException: Can not locate Xul document [.../ui/xul/dataservice-dialog.xul]` from `DataServiceDialog.loadXul()`). The jar itself was intact; a restart cleared it. The clean-baseline checklist now starts with "stop Spoon first". Also recorded on the same restarted session: deleting the scenario D data service **reaped its pointer correctly** (`dloop2.xml` removed and zero `dloop2` references left in `loop_demo2.ktr`), confirming the orphan-leak fix end to end on a `file:///` path. |
| 3.2.2 | 2026-09-18 | @afrjorge | Section 7 gained a **full-lifecycle re-test on the patched jar** (11.0.0.3-312 + `ae39a92`, restarted session): create -> save -> close -> reopen -> dirty -> save -> delete -> recreate, with **zero errors and zero exceptions in the session log**. Final on-disk state tabulated: `dloop2.xml` reaped despite its `file:///` location, `loop_demo2.ktr` left with no embedded residue, and the recreated `dloop3` republished cleanly as `file:///tmp/loop_demo3.ktr` in agreement with its transformation. This single pass exercises both failure modes of the secondary defect and defect A's create/delete path. |
| 3.3.0 | 2026-09-18 | @afrjorge | **Retraction and replacement of the section 9 "silently ignored" claim, plus new Annex C.** The previous claim - that genuine duplicates raise no prompt because `suggestEdit()` is deferred through `syncExec` - was **wrong on three counts** and has been removed: `DataServiceDelegate.syncExec()` is `Display.syncExec` (blocking, and the sync already runs on the UI thread), `AbstractMeta.clearChanged()` clears `changedFlag` *before* `fireContentChangedListeners( false )` so the `!hasChanged()` guard passes, and `setPrompt( false )` is never called in main code. A controlled duplicate test on 2026-09-18 (open `/tmp/loop_demo3.ktr`, *Save As* `/tmp/dup_test.ktr`) confirmed the prompt **does** appear. What it revealed instead is a genuinely separate defect that **survives `ae39a92`**: for a real cross-file duplicate the conflict is legitimate, so `canonicalLocation()` cannot help, and both branches of `suggestEdit()` call `serviceTrans.setChanged()` - leaving the transformation dirty after every save and looping the prompt indefinitely, with the only exits being the destructive "No" branch or manual deletion. Documented as **Annex C** with a full repro; section 9's bullet rewritten to match. |
| 3.3.1 | 2026-09-21 | @afrjorge | Consistency pass, no analysis changed. Section 4's status note said items 5 **and 6** were still open although item 6 (regression tests) was delivered in v2.1.0 and is struck through in section 9 — corrected to item 5 only. Section 7 scenario D carried a duplicated *"Expected (and post-fix)"* paragraph — removed. Section 7's note on the discarded scenario D run still asserted the v3.3.0-retracted claim that genuine duplicates raise **no** prompt (`syncExec` deferral) — replaced with the confirmed finding and a pointer to Annex C. Section 10 said "six orphans, one per run" for a 7-run matrix — clarified that runs 2 and 3 reused the same `.ktr` and hence the same pointer name. SP-7071 claims re-verified against `pentaho-kettle`: `f560af820d6` exists, `git branch -a --contains` lists 26 branches, `master` not among them. |
| 3.3.2 | 2026-09-21 | @afrjorge | **Annex A repro simplified: the separate "poisoning" right-click is redundant and was wrongly labelled mandatory.** Source inspection (`practitest_5375`) shows both `TransPainterStepExtensionPointPlugin.callExtensionPoint()` and `DataServiceStepMenuExtension.callExtensionPoint()` call `metaStoreUtil.getDataServiceByStepName(...)` on the singleton util, so the empty-string negative marker is written by **any** miss — on every canvas repaint of the step, and again on the right-click that opens the **Data Services** submenu to choose **New...**. The old step 2 therefore added nothing that the old step 3 did not already do. Annex A's steps were merged (7 -> 6 steps), the Actual/Expected result step references renumbered, and a note added explaining where the marker really comes from. The same overstated claim was softened in section 7 scenario A and in section 10's per-run procedure; both keep the extra right-click as harmless belt-and-braces. No analysis or root cause changed. |
| 3.3.3 | 2026-09-21 | @afrjorge | **Annex summaries reworked for use as Jira ticket Summary fields.** All three now share the `Data Services: <symptom>` form and fit on one line (86-101 chars): A *"step badge and right-click menu not refreshed after creating a data service"*; B *"`file://` filename mismatch orphans ServiceTrans pointers and blocks saving"*, replacing a 213-character two-clause sentence that bundled both failure modes, with a note offering the split-ticket alternative and why a single ticket is preferred; C *"duplicate service name causes an endless 'already exists' prompt on save"*. Annex C also gained the `**Summary:**` and `**Affects:**` lines it was missing, making the three annexes structurally identical, and its now-duplicated version note was dropped from Preconditions. Annex headings realigned with the summaries. No analysis changed. |
| 3.4.0 | 2026-09-21 | @afrjorge | **Annex C scoped: the loop only occurs while the duplicate name is kept.** Title marked `Annex C (*)` and a scope note added at the top. Verified in code that **renaming the data service in the edit dialog fully resolves the conflict**: `DataServiceDialogController.saveAndClose()` saves under the new name via `delegate.save( delegate.checkConflict( ..., existing ) )` and, when the name changed, drops the old element with `delegate.removeDataService( dataService )`, so the next save finds no conflict, publishes the pointer and leaves the transformation clean — at the cost of one extra save, because `suggestEdit()` calls `setChanged()` on the way out (and provided the new name is itself unpublished). Recorded as **accepted for now** that no warning tells the user a rename is what makes the transformation savable; that guidance is explicitly out of scope. Summary reworded to *"...when a duplicate service name is not renamed"*; repro step 4 now says "without changing the service name"; Expected result narrowed to the non-rename paths; the rename escape added to Additional notes and to the section 9 bullet, which previously claimed the only exits were destructive or manual. |
| 3.4.1 | 2026-09-21 | @afrjorge | **Annex B and section 5 aligned with the v3.4.0 rename finding.** Both asserted that *"neither answer is an escape"* / *"both answers to the prompt are dead ends"*, which is now known to be too strong: **Yes + rename** does end the save loop. Corrected to *"no answer is a **clean** escape"*, with the rename listed as a third outcome and its specific cost in the Annex B case spelled out — because there the conflict is **spurious**, renaming silently changes the name of a service that was never in conflict (breaking JDBC/Carte clients querying the old name) and still leaves the original pointer orphaned, so it trades one defect for two. Updated: section 5's "Third symptom" conclusion and its Consequence bullet, Annex B's second-failure-mode bullet list (now three outcomes, closing on "every route costs the user either the service name, the service itself, or an endless prompt — for a conflict that does not exist") and its save-loop Actual result. Section 7 scenario D's pre-fix **Yes** bullet gained a warning not to rename during that measurement. Neither the root cause nor the `ae39a92` fix is affected: `canonicalLocation()` prevents the spurious conflict from arising, so none of these routes is reached post-fix. |

### Document conventions

* Bump **patch** for wording/typo fixes, **minor** for new findings or new fix items, **major** when the root
  cause analysis itself changes.
* Update `timestamp` in the frontmatter on every edit (ISO 8601 UTC).
* Add one row per revision; never rewrite existing rows.

## Annex A — Jira ticket draft: step badge and right-click menu not refreshed

> Paste-ready. Derived from sections 1-4.

**Summary:** Data Services: step badge and right-click menu not refreshed after creating a data service

**Affects:** PDI 11.0.0.0, 11.0.0.1, 11.0.0.2, 11.0.0.3, 11.1

### Description

After creating a data service on a step via the step's right-click menu, Spoon does not repaint the step. The
bottom-left Data Service badge is not drawn and the **Data Services** submenu still offers **New...** with
**Edit... / Delete... / Test...** disabled, as if no service existed. Saving the transformation does not help.
The service *is* created correctly — it appears under **View -> Configurations -> Data Services**, and
selecting/editing it there makes the badge and menu correct.

Root cause: `DataServiceMetaStoreUtil` holds a per-instance `stepCache` created with a random name
(`initCache` uses `UUID.randomUUID()`), so the read path and the write path use different caches. The painter
(`TransPainterStepExtensionPointPlugin`) and the menu (`DataServiceStepMenuExtension`) read the singleton
`DataServiceContext.getInstance().getMetaStoreUtil()` cache, while the create action writes through a
`DataServiceDelegate` that `DataServiceContext.getDataServiceDelegate()` re-creates — with a brand new cache —
on every call. The first right-click stores an empty-string negative marker in the singleton cache;
`getDataServiceByStepName()` short-circuits on that marker and never falls through to the embedded metastore,
so the newly created service stays invisible.

Secondary impact: because a new delegate (and therefore a new named cache in the `PentahoCacheManager`) is
created per call, this is also an unbounded cache/resource leak.

Regressed in `3398405 [BACKLOG-40276] Moved PDI data services out of OSGi...`, which replaced Blueprint's
shared singletons with `getInstance()` on the read side but left `new`-based construction on the write side.

### Preconditions

* PDI client with the stock `plugins/pdi-dataservice-ee-plugin` installed
  (no patches).
* A transformation saved on the local filesystem containing at least one step that can feed a data service
  (e.g. `Generate Rows`).
* No data service currently defined on that step.

### Steps to reproduce

1. Open the transformation in Spoon.
2. Right-click the target step -> **Data Services** -> **New...**
3. Enter a service name not already present in `~/.pentaho/metastore/pentaho/Data Service Transformation/`
   and click **OK**.
4. Without clicking anything in the **View** tree, look at the step on the canvas.
5. Right-click the step -> **Data Services** again and inspect the item states.
6. Save the transformation (Ctrl+S) and repeat steps 4-5.

> No extra "poisoning" step is needed: the negative cache marker is written by any
> `getDataServiceByStepName()` miss, which happens both when the canvas paints the step
> (`TransPainterStepExtensionPointPlugin`) and on the step 2 right-click itself
> (`DataServiceStepMenuExtension`) — i.e. before the service is created, in every case.

### Actual result

* Step 4: no Data Service badge is painted on the step.
* Step 5: **New...** is still enabled; **Edit... / Delete... / Test...** are still disabled.
* Step 6: unchanged after saving.
* The service is nevertheless listed under **View -> Configurations -> Data Services**; double-clicking
  or right-clicking that node and clicking on the step again makes the badge appear and the submenu states become correct.

### Expected result

The badge appears and the submenu switches to **New...** disabled / **Edit... / Delete... / Test...** enabled
immediately after step 3, with no View-tree interaction required. Deleting the service should likewise remove
the badge and re-enable **New...** immediately.

## Annex B — Jira ticket draft: orphan `ServiceTrans` pointers and blocked saves after reopen

> Paste-ready. Derived from sections 5 and 10.

**Summary:** Data Services: `file://` filename mismatch orphans ServiceTrans pointers and blocks saving

> Alternative, if the two failure modes are split into separate tickets:
> *"Data Services: ServiceTrans pointer not deleted after a transformation is reopened"* (orphan leak) and
> *"Data Services: reopened transformation cannot be saved — spurious 'data service already exists'"*
> (save loop). They share one root cause and one fix, so a single ticket is preferred.

**Affects:** PDI 11.0.0.2, 11.0.0.3, 11.1.0.0 — **not** 11.0.0.0 / 11.0.0.1

### Description

A data service is persisted in two places: the `DataServiceMeta` inside the transformation's embedded
metastore, and a `ServiceTrans` pointer (`<serviceName>.xml`, service name -> `.ktr` location) in the global
metastore under `~/.pentaho/metastore/pentaho/Data Service Transformation/`. The pointer is published when
the transformation is saved and is meant to be reaped by `DataServiceReferenceSynchronizer.sync()` on the
save that follows a delete.

From PDI 11.0.0.2 onward that reaping stops working **once the transformation has been closed and
reopened**. Deleting the data service correctly removes the `DataServiceMeta` from the `.ktr`, but the
`<serviceName>.xml` file survives indefinitely, so the folder accumulates one orphan per data service ever
created.

Root cause: `ServiceTrans.references( TransMeta )` stores `TransMeta.getFilename()` verbatim.
`pentaho-kettle` commit `f560af820d6` *"fix[SP-7071]: Preserve trans/job filename in Carte remote
execution"* added `KettleVFS.normalizeFilePath()`, which prefixes any scheme-less local path with `file://`,
and changed `TransFileListener.open()` from `transMeta.setFilename( fname )` to
`transMeta.setFilename( KettleVFS.normalizeFilePath( fname ) )`. Every Spoon open path funnels through that
listener, so any reopened transformation now carries `file:///Users/...`. Saving a newly created
transformation for the first time does not go through it, so it keeps the plain `/Users/...` form.

`ServiceTrans.StorageMethod.FILE.exists()` resolves the location with `new File( location ).exists()`, which
is always `false` for a URI, so `DataServiceReferenceSynchronizer.getAllPublishedServiceTrans()` filters
those pointers out and they can never reach the synchronizer's `toDelete` set. `Reference.equals()` /
`hashCode()` compare raw location strings and have the same problem. The Spoon log confirms this by
omission: it records `Data Services - Saving '<name>' to '<trans>'` on every creation but produces no entry,
warning or error on delete — the deletion is never attempted rather than attempted and failed.

The same commit also began persisting the filename into the `.ktr` as `<transformation><info><filename>`,
which makes the defect trivially diagnosable: `grep -m1 "<filename>" <path>.ktr` returns a plain path for a
healthy transformation, `file:///...` for one whose data services will leak, and nothing at all on
pre-11.0.0.2 builds. `KettleVFS.normalizeFilePath` is absent from 11.0.0.0 (237) and 11.0.0.1 (259), present
in 11.0.0.2 (294), 11.0.0.3 (312) and 11.1.0.0, and **not present on `master`** — exactly the set of builds
in which the leak occurs. The data-service plugin code itself is unchanged: the `serialization/`, `ui/` and
`DataServiceContext` classes are byte-identical across the 11.0.0.2, 11.0.0.3 and 11.1.0.0 plugin jars.

Consequences:

* `DataServiceMetaStoreUtil.checkConflict()` scans the global pointers, so recreating a data service with a
  previously used name raises a bogus `DataServiceAlreadyExistsException` / "suggest edit" prompt.
* `getDataService( name, repository, metaStore )` can resolve a stale pointer and load the wrong
  transformation, so a new service can inherit state belonging to a deleted namesake.
* JDBC/Carte clients list data services that no longer exist.
* **A transformation can become impossible to save cleanly** — see "Second failure mode" below.

### Preconditions

* PDI client 11.0.0.2, 11.0.0.3 or 11.1.0.0 with the stock `plugins/pdi-dataservice-ee-plugin` installed
  (no patches).
* A transformation saved on the local filesystem with at least one step that can feed a data service
  (e.g. `Generate Rows`).
* **The transformation must be opened from disk**, not merely created and saved in the current session —
  this is what puts the `file:///` form into `TransMeta.getFilename()`.
* `~/.pentaho/metastore/pentaho/Data Service Transformation/` contains no `<serviceName>.xml` for the name
  you are about to use. *(Never delete `.type.xml`.)*

### Steps to reproduce

1. In Spoon, create a transformation containing a step that can feed a data service (e.g. `Generate Rows`)
   and save it to disk as `/tmp/orphan_demo.ktr`.
2. **Close the transformation tab and reopen `/tmp/orphan_demo.ktr`.** Any open method reproduces it —
   `File > Open...`, drag-and-drop from the OS file manager, or launching with `spoon.sh -file=<path>`.
3. Right-click the step -> **Data Services** -> **New...**, name it `orphan_test`, click **OK**.
4. Save the transformation (Ctrl+S).
5. Confirm the normalised filename and the published pointer location:

   ```bash
   grep -m1 "<filename>" /tmp/orphan_demo.ktr
   ls "$HOME/.pentaho/metastore/pentaho/Data Service Transformation/"
   grep -A2 transformation_location "$HOME/.pentaho/metastore/pentaho/Data Service Transformation/orphan_test.xml"
   ```

   Both values start with `file:///`.
6. Right-click the step -> **Data Services** -> **Delete...** and confirm.
7. Save the transformation (Ctrl+S).
8. List the folder again and inspect the `.ktr`:

   ```bash
   ls "$HOME/.pentaho/metastore/pentaho/Data Service Transformation/"
   grep -c "Embedded MetaStore Elements" /tmp/orphan_demo.ktr
   ```
9. Repeat steps 3-7 reusing the same service name `orphan_test`.

### Actual result

* Step 8: `orphan_test.xml` is still present, even though the `.ktr`'s `Embedded MetaStore Elements`
  attribute group no longer contains the service. No error or warning is logged.
* Step 9: creating the service again with the same name raises a "data service already exists" error /
  "suggest edit" prompt, referring to a service that no longer exists anywhere.

### Expected result

The save in step 7 removes `orphan_test.xml` from `~/.pentaho/metastore/pentaho/Data Service Transformation/`,
and the name is immediately reusable in step 9 — i.e. the same behaviour as when step 2 is skipped, and the
same behaviour as on 11.0.0.0, 11.0.0.1 and `master`.

### Second failure mode: unrecoverable "data service already exists" save loop

The same raw-string comparison produces a second, more severe symptom when the pointer was published
**before** the first reopen, i.e. in the plain `/...` form:

| Pointer `transformation_location` | `TransMeta.getFilename()` now | `new File(loc).exists()` | in `allPublished` | in `publishedInTrans` | Outcome |
|---|---|---|---|---|---|
| `file:///p.ktr` | `file:///p.ktr` | false | no | no | orphan leak (above) |
| `/p.ktr` | `file:///p.ktr` | true | **yes** | **no** | **`nameConflicts` -> save loop** |
| `/p.ktr` | `/p.ktr` | true | yes | yes | correct behaviour |

In the second row `DataServiceReferenceSynchronizer.sync()` classifies the service as a name conflict, so
(a) it is excluded from `toSave`, meaning the pointer is never rewritten and the mismatch can never
self-heal, and (b) a `DataServiceAlreadyExistsException` reaches
`SynchronizationListener.contentSafe()`, which calls `DataServiceDelegate.suggestEdit()`. Both branches of
`suggestEdit()` call `serviceTrans.setChanged()`, so the transformation is marked dirty again immediately
after every save:

```java
// DataServiceDelegate.java:120-136
if ( !serviceTrans.hasChanged() ) {
  if ( showPrompt( title, text ) ) {
    editDataService( dataServiceMeta );
    serviceTrans.setChanged();                              // dirty again
  } else {
    // "can not exist duplicate, delete dataService and save transformation (PDI-15584)"
    deleteDataServiceElementAndCleanCache( dataServiceMeta, serviceTrans );
    serviceTrans.setChanged();                              // dirty again
  }
}
```

The user is therefore stuck in an endless prompt/save cycle, and **no answer to the dialog is a clean
escape**:

* **Yes**, then OK **without renaming**, returns to a still-dirty transformation and the next save
  re-raises the same prompt.
* **Yes**, then **renaming** the service in the edit dialog, *does* break the loop — the new name is
  unpublished, so the next save has no conflict (see Annex C). But it is not a resolution here: the conflict
  was **spurious**, so the user silently ends up with a differently named data service — breaking any JDBC
  or Carte client that queries the old name — while the original pointer stays on disk as an orphan,
  unreferenced by anything.
* **No** takes the `else` branch, which **deletes the data service** from the transformation but **leaves
  its pointer XML in place**. `deleteDataServiceElementAndCleanCache()` only removes the embedded element
  (`DataServiceMetaStoreUtil.java:330-333`); the global pointer is deleted solely by `sync()`'s `toDelete`
  list, which is built from `publishedInTrans` — empty in exactly this state, which is what caused the
  conflict in the first place. So the branch destroys the user's work *without* removing the duplicate it
  claims to be resolving, and because the embedded service is now gone nothing can ever republish or reap
  that pointer again: the loop is traded for a permanent orphan plus data loss.

Every route therefore costs the user either the service name, the service itself, or an endless prompt —
for a conflict that does not exist.

#### Steps to reproduce the save loop

1. In Spoon, create a **new** transformation, add a step that can feed a data service, and create the data
   service on it **in the same session**.
2. Save (Ctrl+S) as `/tmp/loop_demo.ktr`. Confirm both paths are plain:

   ```bash
   grep -m1 "<filename>" /tmp/loop_demo.ktr
   grep -A2 transformation_location "$HOME/.pentaho/metastore/pentaho/Data Service Transformation/<name>.xml"
   ```

   Both start with `/`, not `file:///`.
3. Close the transformation tab and reopen `/tmp/loop_demo.ktr`.
4. Make any trivial change so the tab is marked dirty.
5. Press Ctrl+S.

**Actual result:** a dialog appears reading *Data Service "&lt;name&gt;" already exists. Do you want to edit
this data service?*

* Clicking **Yes** and pressing **OK** without changing the name returns to a still-dirty transformation,
  and pressing Ctrl+S reproduces the dialog, indefinitely. The transformation can never be brought to a
  clean state.
* Clicking **Yes** and **renaming** the service in the edit dialog does end the loop, but only by silently
  changing the name of a service that was never really in conflict — breaking JDBC/Carte clients that query
  the old name — and the original pointer is left behind as an orphan.
* Clicking **No** removes the data service from the transformation, but its XML file under
  `~/.pentaho/metastore/pentaho/Data Service Transformation/` remains on disk and is never cleaned up by
  any later save.

**Expected result:** the save completes silently, the tab goes clean, no dialog appears, and the pointer's
`transformation_location` is updated to the current value.

### Additional notes

**Contrast case (passes, and is the cleanest proof).** Perform steps 1 and 3-7 while **skipping step 2**.
The filename stays plain `/tmp/orphan_demo.ktr`, and `orphan_test.xml` *is* correctly removed by the save in
step 7. The data-service code is therefore fine; the regression is in the filename normalisation.

Verified across 7 runs — three open methods on 11.0.0.2 and two each on 11.0.0.3 and 11.1.0.0 — every
reopened transformation leaked. Full matrix and evidence in section 10 of this document.

## Annex C (*) — Jira ticket draft: endless "already exists" prompt for duplicate service names

> **(*) Scope marker — this only bites when the duplicate name is *kept*.** The loop described below
> happens when the user answers **Yes** and closes the edit dialog **without changing the service name**
> (or answers **No**). **Renaming the data service in that edit dialog resolves it completely:**
> `DataServiceDialogController.saveAndClose()` saves the service under the new name and, because the name
> changed, removes the old-named element (`delegate.removeDataService( dataService )`), so the next save
> finds no conflict, publishes the pointer and leaves the transformation clean. The only cost is one extra
> save, since `suggestEdit()` calls `setChanged()` on the way out. (The new name must itself be unpublished,
> otherwise the conflict simply moves.)
>
> **Accepted for now:** the dialog gives no hint that renaming is the action required to make the
> transformation savable. That missing guidance is **not** part of this ticket — it is considered acceptable
> for the time being, and is recorded here only so the gap is known.

> Independent of the two defects above. Verified on **PDI 11.0.0.3-312 with the PRACTITEST-5375 fix already
> applied**, so it is not a regression of that work and is not fixed by it.

**Summary:** Data Services: endless "already exists" prompt on save when a duplicate service name is not renamed

**Affects:** PDI 11.0.0.3 (reproduced on build 312). The mechanism is version-independent and unrelated to
the `file://` normalisation of Annex B, so earlier 11.0.x and 11.1 are expected to be affected too.

### Description

If a transformation is saved while it contains a data service whose name is already published by a
*different* transformation, Spoon raises the expected "already exists" prompt — but answering it never
resolves anything. `DataServiceDelegate.suggestEdit()` calls `serviceTrans.setChanged()` in **both** of its
branches, so the transformation is marked dirty again immediately after the save that triggered the prompt.
The next save repeats the conflict and the prompt reappears, indefinitely. The transformation can never be
brought to a clean state while the duplicate name exists.

The clean way out is to **rename** the data service in the edit dialog (see the scope marker above); leaving
the name unchanged loops, and the remaining alternatives are destructive or manual — answering **No**
silently deletes the data service from the transformation (see Annex B), or the user deletes the duplicate
service by hand and saves again.

```java
// DataServiceDelegate.java:120-136
if ( !serviceTrans.hasChanged() ) {
  if ( showPrompt( title, text ) ) {
    editDataService( dataServiceMeta );
    serviceTrans.setChanged();                              // dirty again -> prompt returns next save
  } else {
    // "can not exist duplicate, delete dataService and save transformation (PDI-15584)"
    deleteDataServiceElementAndCleanCache( dataServiceMeta, serviceTrans );
    serviceTrans.setChanged();                              // dirty again
  }
}
```

Note that editing the data service (the **Yes** branch) only resolves a *name* conflict if the user actually
renames the service — which works, but which the dialog neither asks for nor hints at, so the prompt's
suggested action is ineffective by default. Adding that hint is explicitly **out of scope for now** (scope
marker above); this ticket covers the loop itself.

### Preconditions

* PDI 11.0.0.3 (see **Affects** above).
* Any transformation containing a data service, saved to a file, so that its pointer is published under
  `~/.pentaho/metastore/pentaho/Data Service Transformation/<name>.xml`.

### Steps to reproduce

1. In Spoon, create a transformation with any step that can feed a data service, save it as
   `/tmp/loop_demo3.ktr`, and create a data service named `dloop3` on that step. Save.
2. Confirm the pointer is published and references this file:

   ```bash
   grep -A2 transformation_location \
     "$HOME/.pentaho/metastore/pentaho/Data Service Transformation/dloop3.xml"
   ```

3. With `/tmp/loop_demo3.ktr` open, choose **File -> Save As...** and save it as `/tmp/dup_test.ktr`.
   Two transformations now contain a data service called `dloop3`, which is a genuine duplicate.
4. Observe the prompt, click **Yes**, then click **OK** in the data service edit dialog **without changing
   the service name**. *(Changing it here is the documented way out — see the scope marker at the top of
   this annex.)*
5. Save the transformation again.

### Actual result

Step 4 shows *Data Service "dloop3" already exists. Do you want to edit this data service?*. Clicking **Yes**
opens the edit dialog; clicking **OK** returns to a **dirty** transformation, and the next save shows the
same prompt again. The cycle repeats indefinitely and the transformation can never be saved clean.

The log records the conflict on each attempt — two full cycles from the verification run, the second after
answering **Yes** and confirming the edit dialog:

```
21:39:29.288 - Data Services - Saving 'dloop3' to 'dup_test'
21:39:29.323 - Data Services - Data Service "dloop3" already exists      <- cycle 1
21:41:48.490 - Data Services - Data Service "dloop3" already exists      <- cycle 2, after Yes + OK
```

Note that the second cycle has no preceding `Saving 'dloop3' to ...` line: the embedded element is already
present, so nothing new is written. The save accomplishes nothing except re-running the conflict check and
re-dirtying the transformation.

### Expected result

A duplicate name should be reported once and resolvable. Renaming the service in the edit dialog already
achieves that, so the minimum fix is to stop the *non-rename* paths from looping: closing the dialog with
the name unchanged (or declining the prompt) should leave the transformation saved and clean with the data
service simply left unpublished — not dirty, and not silently deleted. Prompting the user to rename would be
a nice-to-have on top; it is **not** required by this ticket.

### Additional notes

* The transformation *file* is written correctly on every attempt; only the published pointer is skipped
  (`DataServiceReferenceSynchronizer.sync()` excludes name conflicts from `toSave`). The looping is purely
  the `setChanged()` calls in `suggestEdit()`.
* The published pointer of the original transformation is correctly left untouched, as it should be.
* Renaming the data service in the edit dialog is a working escape, verified in code:
  `DataServiceDialogController.saveAndClose()` calls `delegate.save( delegate.checkConflict( ..., existing ) )`
  and then, when `!model.getServiceName().equals( existing )`, `delegate.removeDataService( dataService )` —
  so the duplicate element is gone and the following save publishes the new name cleanly.
* Related but distinct: the same `suggestEdit()` loop is reachable **without** any real duplicate on
  PDI 11.0.0.2+ through the `file://` path mismatch — that is the subject of the Annex B ticket, and is
  fixed by making the reference comparison scheme-independent. This ticket remains after that fix.
