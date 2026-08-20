**Reopening: the underscore 1.13.8 bump also needs the webjar override folder renamed**

The bump from 1.13.7 to 1.13.8 (CVE-2026-27601) was merged in `maven-parent-poms` only:

| Ticket | Branch | PR |
|---|---|---|
| PPP-6439 | `master` | maven-parent-poms #897 |
| SP-7588 | `10.2` | maven-parent-poms #991 |
| SP-7589 | `11.0` | maven-parent-poms #988 |

It missed a companion change in `pentaho-osgi-bundles`. `pentaho-webjars-deployer` wraps underscore's UMD file so it can be loaded through RequireJS. It finds that wrapper by the **exact** webjar version, in `pentaho-webjars-deployer/src/main/resources/overrides/org.webjars.npm/underscore/<version>/overrides.json`, and that folder is still named `1.13.7`. The 1.13.8 file is therefore served unwrapped, and every RequireJS module that requires `underscore` gets `undefined`. The previous underscore bump (SP-6838) renamed the folder in the same way.

**Impact found.** On PDI 10.2.0.11 QAT build 443, the DET Flat Table renders no columns (`TypeError: Cannot read properties of undefined (reading 'each')`). The same RequireJS mapping is used by `@pentaho/marketplace` (its controllers and services require `underscore`) and `@pentaho/di-core-ui`, so the PDI Marketplace may be affected too. QA should check it.

**Fix.** In `pentaho-osgi-bundles`, rename `…/overrides/org.webjars.npm/underscore/1.13.7` to `…/1.13.8`. This is a rename only; the file content does not change. It is needed on every branch where underscore is 1.13.8:
* `10.2` (SP-7588): fixes DET in PDI 10.2. ✅ Verified on 10.2.0.11-443: after the rename, the Flat Table renders and the served `underscore-umd.js` is wrapped again.
* `11.0` (SP-7589) and `master` (PPP-6439): needed for consistency, even though DET is no longer supported in those versions. Any other component loading the underscore webjar through RequireJS is affected in the same way.

Tracked from PDI-20686 / SP-7619. See https://pentaho-eng.atlassian.net/browse/SP-7619?focusedCommentId=465777
