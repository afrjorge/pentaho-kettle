**QAT 10.2.0.11-443: two follow-up fixes for DET (Flat Table and Pivot table)**

After merging the four PRs (pentaho/pentaho-osgi-bundles#561, pentaho/pentaho-analyzer#2820, pentaho/pentaho-platform-plugin-geo#455, pentaho/pentaho-det-ee#743), QAT build 10.2.0.11-443 cold-starts DET correctly. The Model view, charts and geo map work. Two visualizations failed:

1. **Flat Table shows no columns.** Browser console: `TypeError: Cannot read properties of undefined (reading 'each')` in `dataUtils.js`.
2. **Pivot table shows the popup** _"Cannot get default options … Unrecognized field "filters" (class com.pentaho.analyzer.report.y)"_.

**The merged commits are complete and correct.** All four are on `10.2`, and every change is present in the 443 PDI assembly and DET KAR. Both failures only happen in release (obfuscated) builds, and earlier validation used SNAPSHOT/daily builds.

**1. Flat Table: underscore 1.13.8 (not caused by PDI-20686)**
* Cause: SP-7588, the 10.2 backport of PPP-6439 (maven-parent-poms PR #991, merged 2026-10-12), bumped underscore from 1.13.7 to 1.13.8. The `pentaho-webjars-deployer` override that wraps underscore for RequireJS is looked up by exact version, and only `1.13.7` existed. underscore 1.13.8 is therefore served unwrapped, and the module resolves to `undefined`.
* Fix (`pentaho-osgi-bundles`): rename `pentaho-webjars-deployer/src/main/resources/overrides/org.webjars.npm/underscore/1.13.7` to `.../1.13.8`. The previous underscore bump (SP-6838) did the same.
* Note: branches 11.0 and master are also on underscore 1.13.8 with only the 1.13.7 override, so they need the same fix for consistency even if DET is no longer supported in those versions.

**2. Pivot table: missing ProGuard keep rule in the agile-bi jar (latent bug exposed by PDI-20686)**
* Cause: release builds of `paz-plugin-ce` ship the obfuscated `pentaho-analyzer-agile-bi-*-obf.jar`, and in PDI its classes are the ones that run. BACKLOG-38003 kept `PreQualifiedFilters` out of obfuscation only in `server/core/src/assembly/proguard.pro`. In the agile-bi jar it became `report.y` with renamed accessors, so Jackson cannot read `filters.json`. Before pentaho-analyzer PR #2820 this code path failed earlier (`Provider "bundle" not installed`), so the problem never showed.
* Fix (`pentaho-analyzer`): add the same three BACKLOG-38003 keep rules (`PreQualifiedFilters`, `PreQualifiedFilterManager`, `ResourceFileManager`) to `server/agile-bi/src/assembly/proguard.pro`. The Pentaho Server is not affected, because its rules already keep these classes.
* Note: branches 11.0 and master do not need this fix. DEVO-11981 removed obfuscation there, both in `pentaho-analyzer` and in the EE parent POM (`maven-parent-poms-ee`), so no `-obf` jar is built or shipped. The agile-bi `proguard.pro` left on those branches is no longer used.

**Verification (JDK 17, PDI 10.2.0.11-443 cold start)**
* Reproduced both failures on the unmodified 443 install.
* Rebuilt `pentaho-webjars-deployer` (35/35 tests) and the agile-bi jars with the release/obfuscation profile against the 443 obfuscation maps. Deployed them into the 443 install.
* Cold start plus a manual DET pass: Flat Table columns render, and the Pivot table works with no popup. `filters.json` is created and read. There were 0 filter errors in the logs, and all DET/Analyzer/geo endpoints answered 200.

**Next steps**
* `10.2`: open PRs for both commits (local branch `PDI-20686` in each repository; not pushed yet):
  * `pentaho-osgi-bundles`: `fix: DET flat table does not render with underscore 1.13.8 [PDI-20686][SP-7619]`
  * `pentaho-analyzer`: `fix: pivot table in DET cannot load pre-qualified filters in release builds [PDI-20686][SP-7619]`
* `11.0` and `master`: port only the `pentaho-osgi-bundles` underscore override rename, for consistency. The `pentaho-analyzer` fix does not apply there (no obfuscation).
* Retest DET (Model view, Flat Table, charts, geo map, Pivot table) on the next 10.2.0.11 QAT build.
