# PDI 10.2 — SSH access to the embedded Karaf instance is broken

> **Scope.** This document is self-contained and **not** part of the PDI-20686 delivery. It exists
> because the Karaf shell is the only way to run `bundle:diag`, `feature:list` and `bundle:services`,
> which several checks in [`PDI-20686.md` §10](./PDI-20686.md#10-verify) rely on.
>
> **Status:** root-caused, fixed, and ✅ verified end-to-end on a clean install.

---

## 0. How to read this document

| Audience | Start here |
|---|---|
| **Deciding whether to ship it** | [§1 Executive summary](#1-executive-summary), [§11 Residual risks](#11-residual-risks-and-known-issues) |
| **Engineer reviewing the change** | [§3 Root cause](#3-root-cause), [§5 Change footprint](#5-change-footprint-per-repository), [§6 Design decisions](#6-design-decisions-and-why-the-alternatives-were-rejected) |
| **Engineer applying it to an install** | [§9 Deploy](#9-deploy), [§10 Verify](#10-verify) |
| **An AI model asked to replicate this analysis** | [§12 Replication protocol](#12-replication-protocol-for-an-ai-agent) — it is self-contained |

Every claim in [§3](#3-root-cause) and [§6](#6-design-decisions-and-why-the-alternatives-were-rejected)
is annotated with the evidence that supports it, so a reviewer can challenge any individual line
without re-doing the whole investigation.

| Marker | Meaning |
|---|---|
| ✅ **Verified** | Observed directly in this environment (log line, listening socket, Karaf shell output). |
| 📄 **Sourced** | Traced to a specific commit, file or upstream ticket. |
| ⚠️ **Inferred** | Consistent with all observations, but not directly proven. Challenge these first. |

---

## 1. Executive summary

**The symptom.** In PDI 10.2.0.x the embedded Karaf SSH console never accepts a connection. The port
announced in `SpoonDebug.txt` is never bound and `system/karaf/etc/host.key` is never generated.

**Why it broke.** Two independent defects, both of which must be fixed:

1. **The `ssh` feature is not booted.** 📄 `featuresBoot` in the *client* assemblies does not list
   `ssh`, so on PDI the SSH server is never installed in the first place.
2. **When it *is* booted, its activator crashes.** PDI runs Karaf **embedded**, and since SP-6960 /
   PDI-20311 the PDI main class loader ships Apache SSHD **2.16.0** (`lib/`) while the Karaf `ssh`
   feature uses Apache SSHD **2.12.1** (`system/karaf/system`). Apache SSHD resolves classes through
   the **thread context class loader**, deliberately bypassing OSGi wiring, so the two class spaces
   mix and `org.apache.karaf.shell.ssh.Activator` dies with a `ClassCastException` before
   `SshServer#start()` is ever reached.

**The fix.** Boot-delegate `org.apache.sshd.*` to the class loader that loads the OSGi framework, which
for the embedded case is the PDI class loader containing `lib/`. That collapses the two class spaces
into one. Plus add `ssh` to `featuresBoot` for the client assemblies.

**The result.** ✅ Verified on `pdi-ee-client-10.2.0.0-20260818.001718-1758-osgi`: the port is bound,
`host.key` is generated, `sshpass … "system:version"` returns `4.4.6`, `feature:list -i` reports
`ssh │ 4.4.6.2026_05_12 │ x │ Started`, and `bundle:diag` / `bundle:list` / `bundle:services` were used
to verify the whole PDI-20686 cold-start **and** hot-deploy change set.

> **Not a BouncyCastle problem.** The failure is unrelated to the `bcprov-jdk15on` blacklisting or the
> `jdk18on` 1.84 upgrade, and it is **not** fixed by aligning the two Apache SSHD versions — see
> [§6.1](#61-why-boot-delegation-and-not-a-version-alignment).

---

## 2. Symptom

```
$ sshpass -p karaf ssh -p 8802 karaf@127.0.0.1 -oHostKeyAlgorithms=+ssh-rsa -oStrictHostKeyChecking=no
ssh: connect to host 127.0.0.1 port 8802: Connection refused
```

✅ The port *is* announced in `SpoonDebug.txt`:

```
*** Karaf Port:8802                                                         ***
*** OSGI Service Port:9051                                                  ***
```

but **nothing ever listens on it**, and `system/karaf/etc/host.key` is never created.

| Observation | Meaning |
|---|---|
| `ssh` missing from `featuresBoot` in `system/karaf/etc/org.apache.karaf.features.cfg` | Defect 1 — the feature is never installed |
| With `ssh` added: feature installs, all its bundles resolve and are `Active`, but still no socket | Defect 2 — the activator crashes |

✅ With `ssh` booted, all of the feature's bundles are `Active`, so the problem is **not** a missing or
wrongly blacklisted encryption dependency:

* `mvn:org.apache.sshd/sshd-osgi/2.12.1`, `sshd-scp/2.12.1`, `sshd-sftp/2.12.1`
* `mvn:org.bouncycastle/bcprov-jdk18on/1.84`, `bcpkix-jdk18on/1.84`, `bcutil-jdk18on/1.84`
* `mvn:org.apache.karaf.shell/org.apache.karaf.shell.ssh/4.4.6`

---

## 3. Root cause

### 3.1 The activator crashes

✅ `org.apache.karaf.shell.ssh.Activator` fails while building the SSH server:

```
WARN [activator-1-thread-1] org.apache.karaf.shell.ssh.Activator - Error starting activator
java.lang.ClassCastException: Cannot cast org.apache.sshd.common.util.security.bouncycastle.BouncyCastleSecurityProviderRegistrar to org.apache.sshd.common.util.security.SecurityProviderRegistrar
	at java.base/java.lang.Class.cast(Class.java:3606)
	at org.apache.sshd.common.util.ReflectionUtils.newInstance(ReflectionUtils.java:74)
	at org.apache.sshd.common.util.threads.ThreadUtils.createDefaultInstance(ThreadUtils.java:168)
	at org.apache.sshd.common.util.security.SecurityUtils.register(SecurityUtils.java:431)
	at org.apache.sshd.common.util.security.SecurityUtils.isBouncyCastleRegistered(SecurityUtils.java:396)
	at org.apache.sshd.common.util.security.SecurityUtils.getRandomFactory(SecurityUtils.java:554)
	at org.apache.sshd.common.BaseBuilder.fillWithDefaultValues(BaseBuilder.java:166)
	at org.apache.sshd.server.ServerBuilder.build(ServerBuilder.java:151)
	at org.apache.sshd.server.SshServer.setUpDefaultServer(SshServer.java:442)
	at org.apache.karaf.shell.ssh.Activator.createSshServer(Activator.java:184)
	at org.apache.karaf.shell.ssh.Activator.createAndRunSshServer(Activator.java:121)
	at org.apache.karaf.shell.ssh.Activator.doStart(Activator.java:116)
	at org.apache.karaf.util.tracker.BaseActivator.run(BaseActivator.java:312)
```

Because the activator throws, `SshServer#start()` is never reached: the port is never bound and the
host key is never created. Karaf keeps running normally, which is why the failure is invisible (see
[§11.1](#111-the-failure-is-silent)).

### 3.2 Two Apache SSHD class spaces in the same JVM

PDI runs Karaf **embedded** in the Spoon/Kitchen/Pan JVM, so the OSGi framework and the PDI application
class loader coexist:

| Class space | Apache SSHD | Files (relative to the install) | Origin |
|---|---|---|---|
| PDI main class loader | **2.16.0** | `lib/sshd-common-2.16.0.jar`, `lib/sshd-core-2.16.0.jar`, `lib/sshd-sftp-2.16.0.jar`, `lib/bcprov-jdk15to18-1.84.jar`, `lib/bcpkix-jdk15to18-1.84.jar`, `lib/bcutil-jdk15to18-1.84.jar` | 📄 `kettle-engine` (`org.pentaho.di.core.ssh.mina.MinaSshConnection`, `MinaSftpSession`), declared in `~/github/pentaho-kettle/engine/pom.xml`, versioned by `apache-sshd.version` in `~/github/maven-parent-poms/pom.xml`. Added by **SP-6960 / PDI-20311** when `trilead-ssh2` was replaced by Apache SSHD (MINA) |
| Karaf OSGi | **2.12.1** | `system/karaf/system/org/apache/sshd/{sshd-osgi,sshd-scp,sshd-sftp}/2.12.1/*.jar` | 📄 `ssh` feature of the Karaf fork `4.4.6-2026.05.12`, defined in `system/karaf/system/org/hitachivantara/karaf/features/standard/4.4.6-2026.05.12/standard-4.4.6-2026.05.12-features.xml` |

📄 `org.apache.sshd.common.util.threads.ThreadUtils#iterateDefaultClassLoaders` resolves classes in this
order:

1. `Thread.currentThread().getContextClassLoader()` (**TCCL**)
2. the anchor class' class loader
3. the system class loader

and `ThreadUtils#createDefaultInstance(Iterable, Class, String)` only catches `ClassNotFoundException`,
so **the first class loader that *can* load the class wins**.

In the embedded scenario the TCCL of the Karaf activator threads is the PDI launcher class loader
(`launcher/launcher.jar`, whose `libraries` entry in `launcher/launcher.properties` points at `lib/`),
which contains Apache SSHD **2.16.0**. Therefore:

* `SecurityProviderRegistrar` (the anchor / target type) is loaded from the OSGi bundle
  `sshd-osgi-2.12.1.jar` — that is where `SecurityUtils` runs from — while
* `BouncyCastleSecurityProviderRegistrar` is loaded from `lib/sshd-common-2.16.0.jar` through the TCCL.

Two different `java.lang.Class` objects for the same type name produce a `ClassCastException`, and OSGi
cannot prevent it because Apache SSHD deliberately bypasses the OSGi wiring.

### 3.3 Proof

✅ Starting Spoon with `lib/sshd-common-2.16.0.jar`, `lib/sshd-core-2.16.0.jar` and
`lib/sshd-sftp-2.16.0.jar` temporarily moved away, and `system/karaf/caches/spoon` deleted, makes the
`ClassCastException` disappear: the port is bound, `host.key` is generated and the SSH login succeeds.
Putting the three jars back reproduces the failure. This isolates the PDI-side Apache SSHD as the
cause, independently of BouncyCastle.

---

## 4. The solution in one picture

```
                 BEFORE                                    AFTER
                 ──────                                    ─────
 PDI class loader        OSGi bundles             PDI class loader        OSGi bundles
 lib/sshd-*-2.16.0       sshd-osgi-2.12.1         lib/sshd-*-2.16.0       sshd-osgi-2.12.1
        │                       │                        │                       │
        │  TCCL lookup          │ anchor class           │  org.apache.sshd.* is boot delegated
        └───────► Registrar     └──► Registrar           └──────────┬────────────┘
                  (2.16.0)           (2.12.1)                       ▼
                        ✗ ClassCastException              ONE class space (2.16.0)
                                                                    ▼
                                                    SshServer.start() → port bound, host.key written
```

`org.osgi.framework.bundle.parent=framework` is already set in `etc/config.properties`, so the boot
delegation parent *is* the class loader that owns `lib/`. Felix consults it first and, when the package
is not found there, falls back to the normal OSGi wiring — so assemblies that do **not** ship Apache
SSHD in `lib/` keep using `sshd-osgi` exactly as before ([§6.2](#62-why-boot-delegation-is-safe-here)).

---

## 5. Change footprint per repository

Both commits are on branch **`10.2-ssh`**, message `fix: karaf ssh connection`.

| Repository | Commit | Files |
|---|---|---|
| `pentaho-karaf-assembly` | [`658986c77`](https://github.com/pentaho/pentaho-karaf-assembly/commit/658986c77) | 3 — [§5.1](#51-pentaho-karaf-assembly), [§5.3](#53-the-dead-bouncycastle-property) |
| `pentaho-karaf-ee-assembly` | [`fb877cb7`](https://github.com/pentaho/pentaho-karaf-ee-assembly/commit/fb877cb7) | 1 — [§5.2](#52-pentaho-karaf-ee-assembly) |

> ⚠️ **Not yet committed:** adding `ssh` to `featuresBoot` for the client assemblies
> ([§5.4](#54-still-uncommitted--ssh-in-featuresboot)). Without it the fix above is necessary but not
> sufficient on a pristine PDI client.

### 5.1 `pentaho-karaf-assembly`

Both files are byte-for-byte duplicates of each other and **must stay in sync**:

* `assemblies/common-resources/src/main/resources/etc/config.properties`
* `assemblies/client/src/main/resources/etc/config.properties`

```properties
# javax.security.cert is needed by pax-transx-tm-narayana as it uses URLClassLoader and otherwise is unable to create
# object store.
#
# org.apache.sshd.* must be boot delegated because Karaf runs embedded in the Pentaho application and Apache SSHD
# resolves its own classes through the thread context class loader (ThreadUtils#iterateDefaultClassLoaders tries the
# TCCL first). Since the products ship Apache SSHD in their main class loader (lib/), the Karaf 'ssh' feature would
# otherwise mix the SSHD classes of the OSGi bundles with the SSHD classes of the main class loader and fail with
# "java.lang.ClassCastException: Cannot cast org.apache.sshd.common.util.security.bouncycastle
# .BouncyCastleSecurityProviderRegistrar to org.apache.sshd.common.util.security.SecurityProviderRegistrar",
# which prevents the Karaf SSH server from ever binding its port. Boot delegation keeps a single SSHD class space.
# When Apache SSHD is not present in the main class loader, Felix silently falls back to the OSGi bundles.
#
org.osgi.framework.bootdelegation = \
    com.sun.*, \
    javax.transaction, \
    javax.transaction.xa, \
    javax.xml.crypto, \
    javax.xml.crypto.*, \
    javax.security.cert, \
    jdk.nashorn.*, \
    sun.*, \
    jdk.internal.reflect, \
    jdk.internal.reflect.*, \
    org.apache.karaf.jaas.boot, \
    org.apache.karaf.jaas.boot.principal, \
    org.apache.sshd.*
```

### 5.2 `pentaho-karaf-ee-assembly`

`assemblies/client/src/main/resources/etc/config.properties` — identical change, same sync rule. This
is the copy that ends up as `system/karaf/etc/config.properties` in the **PDI EE client**.

### 5.3 The dead BouncyCastle property

Removed from `pentaho-karaf-assembly/assemblies/common-resources/src/main/resources/etc/custom.system.properties`:

```properties
# disables the BouncyCastle registrar, preventing it to wrongly assume it is
# supported, due to the presence of bcprov-jdk14-138.jar in the main classloader;
# enabling it requires an upgrade (>= 1.58) and extra configuration
# (https://karaf.apache.org/manual/latest/security#_security_providers)
org.apache.sshd.config.org.apache.sshd.security.provider.BC.enabled = false
```

📄 It has never had any effect: Apache SSHD's registrar configuration namespace is
`org.apache.sshd.security.provider.` (constant `CONFIG_PROP_BASE` in `SecurityProviderRegistrar`);
there is no `org.apache.sshd.config.` prefix anywhere in `sshd-common`. Its comment refers to
`bcprov-jdk14-138.jar`, which has not shipped for years. Leaving it in place is **dangerous**, because
"fixing" the typo would disable BouncyCastle for the Karaf SSH server. It would also not have avoided
the `ClassCastException`, because Apache SSHD instantiates the registrar *before* asking it whether it
is enabled.

### 5.4 Still uncommitted — `ssh` in `featuresBoot`

✅ `featuresBoot` in `assemblies/client/src/main/resources-filtered/etc/org.apache.karaf.features.cfg`
of **both** assemblies does not list `ssh`:

```properties
featuresBoot=\
  config,\
  pentaho-base,\
  pentaho-client-minimal,\
  pentaho-big-data-plugin-osgi,\
  pentaho-fasterxml
```

so on a pristine PDI client the SSH server is never installed, whatever `config.properties` says. The
`ssh` entry must be added — first in the list, as the `server` assembly already does:

```properties
featuresBoot=\
  ssh,\
  config,\
  ...
```

⚠️ **Decide before shipping.** Booting an SSH server by default in a *desktop client* is a security
posture change, not just a bug fix. Options: ship it enabled (matching the server assembly), ship it
commented out with documentation, or leave it a support-only manual step. This document does not
choose; [§9](#9-deploy) applies it to the local test install only.

---

## 6. Design decisions and why the alternatives were rejected

### 6.1 Why boot delegation, and not a version alignment

Aligning the Karaf `ssh` feature onto Apache SSHD 2.16.0 looks like the obvious fix and **does not
work**: identical bytecode loaded by two different class loaders still yields incompatible
`java.lang.Class` objects. The defect is the *duplication* of the class space, not the version skew.
Version alignment remains worthwhile as hardening ([§6.4](#64-optional-hardening-not-required-for-the-fix)),
not as the fix.

### 6.2 Why boot delegation is safe here

* 📄 Apache Felix only consults the boot delegation parent **first**; when the package is not found
  there it falls back to the normal OSGi wiring (only `java.*` is terminal). Assemblies that do not
  ship Apache SSHD in their main class loader — such as the one produced from
  `pentaho-karaf-assembly/assemblies/server` — therefore keep using `sshd-osgi`, `sshd-scp` and
  `sshd-sftp` exactly as before.
* ✅ `org.apache.sshd.scp.server` is **not** in `lib/` and consequently still comes from the OSGi bundle
  `sshd-scp-2.12.1.jar`. Because that bundle's super types (`org.apache.sshd.server.command`,
  `org.apache.sshd.common`) are boot delegated as well, the `ScpCommandFactory` built by
  `org.apache.karaf.shell.ssh.Activator` remains type-compatible. Verified with **and** without
  `sshd-scp-2.16.0.jar` present in `lib/`; the SSH server starts and accepts logins in both cases.
* ✅ BouncyCastle is left untouched. With the fix, Apache SSHD uses `lib/bcprov-jdk15to18-1.84.jar`,
  `lib/bcpkix-jdk15to18-1.84.jar` and `lib/bcutil-jdk15to18-1.84.jar`, and `hostKeyFormat = simple` in
  `system/karaf/etc/org.apache.karaf.shell.cfg` does not require the BouncyCastle-based OpenSSH key
  parser.

### 6.3 Files deliberately not changed

| File | Why |
|---|---|
| `pentaho-karaf-assembly/assemblies/{common-resources,server}/src/main/resources-filtered/etc/custom.properties` | `org.osgi.framework.bootdelegation` is not defined there, and defining it would require duplicating the whole Karaf default list |
| `.../etc/org.apache.karaf.features.xml` (both assemblies) | The BouncyCastle blacklists, bundle replacements and the `ssh` feature definition are correct as they are; the `ssh` feature installs and resolves without errors |
| `~/github/maven-parent-poms/pom.xml` | `apache-sshd.version=2.16.0` and the BouncyCastle versions are correct |
| `~/github/pentaho-kettle/engine/pom.xml` | The `sshd-core` / `sshd-sftp` dependencies stay as they are |

### 6.4 Optional hardening (not required for the fix)

Keep the Apache SSHD used by the Karaf `ssh` feature aligned with `apache-sshd.version` from
`maven-parent-poms` (currently `2.16.0`), so `sshd-scp` is never a different minor version from the
Apache SSHD in `lib/`. Either:

1. Add `bundleReplacements` entries for `mvn:org.apache.sshd/{sshd-osgi,sshd-scp,sshd-sftp}` in
   `pentaho-karaf-assembly/assemblies/common-resources/src/main/resources-filtered/etc/org.apache.karaf.features.xml`; or
2. Add an `org.apache.sshd:sshd-scp` dependency to `~/github/pentaho-kettle/engine/pom.xml` next to the
   existing `sshd-core` and `sshd-sftp`, so the whole stack lives in `lib/` at a single version.

---

## 7. Environment

| Purpose | Path |
|---|---|
| **Pristine 10.2.0.X** | `~/pentaho/pdi/10.2.0.0-SNAPSHOT/20260818/pdi-ee-client-10.2.0.0-20260818.001718-1758-osgi_clean/data-integration` *(read-only)* |
| **Test / debug install** | `~/pentaho/pdi/10.2.0.0-SNAPSHOT/20260818/pdi-ee-client-10.2.0.0-20260818.001718-1758-osgi/data-integration` |

Source repositories, both on branch `10.2-ssh`, under `~/github`: `pentaho-karaf-assembly`,
`pentaho-karaf-ee-assembly`. Referenced but unchanged: `pentaho-kettle`, `maven-parent-poms`.

Versions observed in the test install: PDI class loader Apache SSHD **2.16.0** + BouncyCastle
`jdk15to18` **1.84**; Karaf OSGi Apache SSHD **2.12.1** + BouncyCastle `jdk18on` **1.84**; Karaf
**4.4.6** (fork `4.4.6-2026.05.12`); JDK Corretto **11.0.26**.

---

## 8. Build

```bash
for repo in pentaho-karaf-assembly pentaho-karaf-ee-assembly; do
  ( cd ~/github/$repo && mvn clean install -DskipTests ) || break
done
```

The changed files are plain resources; the only artifact that matters downstream is the
`etc/config.properties` embedded in the client assembly. For a **local** test install it is faster to
edit the install in place — [§9](#9-deploy).

---

## 9. Deploy

Applying the fix to an existing install, without rebuilding:

```bash
PDI="$HOME/pentaho/pdi/10.2.0.0-SNAPSHOT/20260818/pdi-ee-client-10.2.0.0-20260818.001718-1758-osgi/data-integration"
ETC="$PDI/system/karaf/etc"

# 1. Boot-delegate Apache SSHD  (§5.1 / §5.2) - append to the bootdelegation list
#    org.apache.karaf.jaas.boot.principal   ->   org.apache.karaf.jaas.boot.principal, \
#                                                    org.apache.sshd.*
$EDITOR "$ETC/config.properties"

# 2. Remove the dead BouncyCastle property  (§5.3)
$EDITOR "$ETC/custom.system.properties"

# 3. Boot the ssh feature  (§5.4) - add "ssh,\" as the first entry of featuresBoot
$EDITOR "$ETC/org.apache.karaf.features.cfg"

# 4. Reset to a genuine cold-start state - MANDATORY, see §11.2
rm -rf "$PDI/system/karaf/caches"; rm -f "$PDI/SpoonDebug.txt" "$PDI/logs/"*.log

# 5. Launch (answer Y to both prompts)
cd "$PDI" && ./SpoonDebug.sh
```

---

## 10. Verify

```bash
PDI="$HOME/pentaho/pdi/10.2.0.0-SNAPSHOT/20260818/pdi-ee-client-10.2.0.0-20260818.001718-1758-osgi/data-integration"
SSH="sshpass -p karaf ssh -p 8802 karaf@127.0.0.1 -oHostKeyAlgorithms=+ssh-rsa -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null"

grep "Karaf Port" "$PDI/SpoonDebug.txt"
lsof -nP -iTCP:8802 -sTCP:LISTEN
ls -l "$PDI/system/karaf/etc/host.key"
$SSH "system:version"
$SSH "feature:list -i | grep ssh"
```

| # | Check | ✅ Result observed |
|---|---|---|
| 1 | Port announced | `*** Karaf Port:8802 ***` |
| 2 | Port bound | `java … TCP *:8802 (LISTEN)` |
| 3 | Host key generated | `-rw-------  1 andjorge  staff  1704 … etc/host.key` (+ `host.key.pub`) |
| 4 | Login works | `system:version` → **`4.4.6`** |
| 5 | Feature started | `ssh │ 4.4.6.2026_05_12 │ x │ Started │ standard-4.4.6-2026.05.12` |
| 6 | Diagnostics usable | `bundle:diag` empty, `bundle:list -s` → 181 `Active`, `bundle:services <id>` lists the servlets — this is what [`PDI-20686.md` §10](./PDI-20686.md#10-verify) needs |

Before the fix the same commands produce no listening socket, no `host.key`, and
`ssh: connect to host 127.0.0.1 port 8802: Connection refused`.

> 📌 **Use `127.0.0.1`, not `localhost`.** Unrelated to SSH, but it bites in the same session: this
> build's Jetty `IPAccessHandler` rejects IPv6 literals, so an HTTP probe resolved to `::1` returns
> **HTTP 500 `Invalid IP address: 0:0:0:0:0:0:0:1`** instead of the real response.

---

## 11. Residual risks and known issues

### 11.1 The failure is silent

PDI only installs `pax-logging-api` (wrapped as `mvn:org.hitachivantara/pax-logging-api-wrap/2.2.12`,
declared in `system/karaf/etc/startup.properties`) and **no pax-logging backend**. OSGi logging is
therefore routed to the PDI Log4j2 configuration `classes/log4j2.xml`, whose root level is `ERROR`. The
Karaf activator logs the failure at `WARN`, so nothing appears in `SpoonDebug.txt`, `logs/karaf.log` or
`logs/pdi.log`.

To make it visible, add this appender inside `<Appenders>` of `classes/log4j2.xml`:

```xml
<File name="karaf-debug-appender" fileName="logs/karaf-debug.log" append="false">
    <PatternLayout>
        <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5p [%t] %c - %m%n%throwable</Pattern>
    </PatternLayout>
</File>
```

and these loggers inside `<Loggers>` of the same file:

```xml
<Logger name="org.apache.karaf" level="DEBUG" additivity="false">
    <appender-ref ref="karaf-debug-appender"/>
</Logger>
<Logger name="org.apache.sshd" level="DEBUG" additivity="false">
    <appender-ref ref="karaf-debug-appender"/>
</Logger>
<Logger name="org.apache.aries.blueprint" level="DEBUG" additivity="false">
    <appender-ref ref="karaf-debug-appender"/>
</Logger>
```

The stack trace of [§3.1](#31-the-activator-crashes) is then written to `logs/karaf-debug.log`.

⚠️ This is a **diagnostic aid, not part of the fix**, and it is noisy. Revert it afterwards.

### 11.2 Stale Karaf caches hide or alter the behaviour

The feature service state is persisted per Karaf instance, e.g. in
`system/karaf/caches/spoon/data-1/cache/bundle19/data/state.json`. An instance provisioned *before*
`ssh` was added to `featuresBoot` never installs the feature at all
(`feature:ssh/[4.4.6.2026_05_12,4.4.6.2026_05_12]` missing from that file's `features` entry), so the
port is missing for a completely different reason. **Always delete `system/karaf/caches` before
validating any SSH change.**

### 11.3 Open questions

| # | Risk | Assessment |
|---|---|---|
| 1 | Booting `ssh` by default on a desktop client is a security posture change | Unresolved by design — see [§5.4](#54-still-uncommitted--ssh-in-featuresboot). The port binds on `0.0.0.0` (`sshHost` in `org.apache.karaf.shell.cfg`) with the default `karaf/karaf` credentials |
| 2 | Boot delegation makes PDI's Apache SSHD version authoritative for Karaf | ⚠️ Accepted. A future PDI upgrade of `apache-sshd.version` silently becomes the version the Karaf shell runs on. [§6.4](#64-optional-hardening-not-required-for-the-fix) keeps them aligned deliberately rather than accidentally |
| 3 | Other libraries duplicated between `lib/` and OSGi could fail the same way | ⚠️ Not audited. The failure mode is specific to libraries that resolve through the TCCL; Apache SSHD is the only one known to do so here |

---

## 12. Replication protocol for an AI agent

A self-contained recipe to reproduce this analysis from scratch.

### 12.1 Ground rules

1. Treat the `_clean` install as a **read-only reference**; do all testing in the test install
   ([§7](#7-environment)).
2. Delete `system/karaf/caches` and `SpoonDebug.txt` before **every** run — otherwise you are testing a
   warm start and [§11.2](#112-stale-karaf-caches-hide-or-alter-the-behaviour) will mislead you.
3. Never kill processes by name. Resolve PIDs against the **install directory**
   (`pgrep -f "<install-dir>"`), then `kill`, then **poll until gone**, escalating to `kill -9`.
   `SIGTERM` alone has never been observed to stop this JVM.
4. Probe over `127.0.0.1`, never `localhost` ([§10](#10-verify)).

### 12.2 Investigation sequence

| Step | Question | How to answer it |
|---|---|---|
| 1 | Is the feature even installed? | `grep -A6 featuresBoot etc/org.apache.karaf.features.cfg`. If `ssh` is absent, that is defect 1 — add it and restart before going further |
| 2 | Is the port bound? | `lsof -nP -iTCP:<karaf-port> -sTCP:LISTEN`; `ls etc/host.key`. Both empty ⇒ the server never started |
| 3 | Did the bundles resolve? | `bundle:list -s \| grep -i sshd` — if they are all `Active`, it is **not** a dependency or blacklist problem |
| 4 | Why did the activator fail? | The failure is logged at `WARN` and swallowed. Add the Log4j2 appender of [§11.1](#111-the-failure-is-silent) and re-run |
| 5 | Where do the two classes come from? | The exception names both types. `ls lib/ \| grep sshd` and `find system/karaf/system/org/apache/sshd -name '*.jar'` — different versions ⇒ two class spaces |
| 6 | Which one does the TCCL reach? | Read `ThreadUtils#iterateDefaultClassLoaders`: TCCL first. The TCCL is the PDI launcher loader (`launcher/launcher.properties` → `libraries=…/lib`) |
| 7 | Prove it | Move `lib/sshd-*.jar` aside, clear the caches, restart. Failure disappears; restore, it returns ([§3.3](#33-proof)) |
| 8 | Is version alignment enough? | No — reason about class identity, not bytecode ([§6.1](#61-why-boot-delegation-and-not-a-version-alignment)) |

### 12.3 Diagnostic cheat-sheet

```bash
PDI=<install>/data-integration
SSH="sshpass -p karaf ssh -p 8802 karaf@127.0.0.1 -oHostKeyAlgorithms=+ssh-rsa \
     -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null"

grep -E "Karaf Port|OSGI Service Port" "$PDI/SpoonDebug.txt"   # ports actually assigned
lsof -nP -iTCP:8802 -sTCP:LISTEN                                # is the SSH port bound?
ls -l "$PDI/system/karaf/etc/host.key"                          # generated only after start()
grep -A6 featuresBoot "$PDI/system/karaf/etc/org.apache.karaf.features.cfg"
grep -A16 bootdelegation "$PDI/system/karaf/etc/config.properties"
ls "$PDI/lib" | grep -iE "sshd|bcprov|bcpkix|bcutil"            # PDI class space
find "$PDI/system/karaf/system/org/apache/sshd" -name "*.jar"   # OSGi class space

$SSH "system:version"
$SSH "feature:list -i | grep ssh"
$SSH "bundle:list -s | grep -i sshd"
$SSH "bundle:diag"
```

### 12.4 Traps discovered the hard way

| Trap | Symptom | Correct response |
|---|---|---|
| Stale Karaf cache | The feature is not installed even though `featuresBoot` lists it | `rm -rf system/karaf/caches` before every run |
| Assuming a dependency problem | All `sshd`/`bc` bundles are `Active`, so the blacklists "must" be wrong | They are fine; look at the activator, not the wiring |
| Trusting the logs | No error anywhere | The backend swallows `WARN`; add the appender of [§11.1](#111-the-failure-is-silent) |
| Fixing the version skew | Same `ClassCastException` after aligning to 2.16.0 | Class *identity*, not version, is the problem |
| "Fixing" the BC property typo | SSH still broken, now with BouncyCastle disabled | Delete the property, do not repair it ([§5.3](#53-the-dead-bouncycastle-property)) |
| Probing over `localhost` | HTTP 500 `Invalid IP address: 0:0:0:0:0:0:0:1` | Use `127.0.0.1` |

---

## 13. Glossary

| Term | Meaning |
|---|---|
| **TCCL** | Thread context class loader — the loader Apache SSHD consults *first*, bypassing OSGi wiring |
| **Boot delegation** | Felix mechanism (`org.osgi.framework.bootdelegation`) that sends listed packages to the framework's parent class loader before the OSGi wiring |
| **Embedded Karaf** | Karaf running inside the Spoon/Kitchen/Pan JVM rather than as its own process — the reason two class spaces coexist |
| **`featuresBoot`** | Karaf features installed at startup, in `etc/org.apache.karaf.features.cfg` |
| **`host.key`** | The SSH server host key; generated by `SshServer#start()`, so its absence proves the server never started |
| **`hostKeyFormat = simple`** | Karaf setting that avoids the BouncyCastle-based OpenSSH key parser |

---

## 14. Change log of this document

| Date | Change |
|---|---|
| 2026-08-18 (rev 2) | **Restructured** to mirror [`PDI-20686.md`](./PDI-20686.md) / [`PDI-20686_hot.md`](./PDI-20686_hot.md): audience guide and evidence markers, executive summary, root cause, solution diagram, change footprint, design decisions, environment/build/deploy/verify, residual risks, an AI replication protocol, a glossary and this change log. Paths moved to the `…-20260818.001718-1758-osgi` install and reduced to install-relative form. **New finding:** `ssh` is absent from `featuresBoot` in the *client* assemblies of both `pentaho-karaf-assembly` and `pentaho-karaf-ee-assembly`, so boot delegation alone is not sufficient on a pristine PDI client — documented as a second, still-uncommitted change in [§5.4](#54-still-uncommitted--ssh-in-featuresboot) with its security caveat. Added the `localhost` → IPv6 HTTP 500 trap. ✅ Re-verified end-to-end: port 8802 bound, `host.key` generated, `system:version` → 4.4.6, `ssh` feature `Started`, and `bundle:diag` / `bundle:list` / `bundle:services` used to verify the PDI-20686 cold-start and hot-deploy runs. |
| 2026-08-10 | Initial version: root cause (two Apache SSHD class spaces reached through the TCCL), the boot-delegation fix in `pentaho-karaf-assembly` [`658986c77`](https://github.com/pentaho/pentaho-karaf-assembly/commit/658986c77) and `pentaho-karaf-ee-assembly` [`fb877cb7`](https://github.com/pentaho/pentaho-karaf-ee-assembly/commit/fb877cb7), removal of the dead `org.apache.sshd.config.…BC.enabled` property, the isolation proof, the silent-failure and stale-cache notes, and the verification commands. |
