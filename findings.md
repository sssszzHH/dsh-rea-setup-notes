# Findings — two failures and what actually caused them

## Finding 1 — a non-ASCII `JAVA_HOME` silently disables Ghidra

### Symptom

`mcp__rea__binary_session` responded normally, but the analysis engine was
unavailable, so every native-binary tool was disabled:

```json
"availability": {
  "status": "unavailable",
  "code": "unsupported_version",
  "reason": "Update JAVA_HOME to the root of a 64-bit full JDK 21 or newer installation.",
  "diagnostics": {
    "java_home": "C:\\Users\\????\\.dsh\\plugins\\rea-bundle\\vendor\\jdk",
    "java_version": "21.0.12.1",
    "checks": [{ "name": "java", "ok": false,
      "detail": "21.0.12.1; 64-bit; runtime only; ... accepted a 64-bit full JDK 21 or newer" }]
  }
}
```

Two red herrings in that message:

* **"runtime only" was false.** It was a complete JDK — `bin/javac.exe`, `jmods/`,
  and a `release` file containing `IMAGE_TYPE="JDK"`; `java --list-modules` listed
  `jdk.compiler@21.0.12.1` among 70 modules.
* **The `????` was the real clue**: `C:\Users\保安` losing its non-ASCII bytes.

### Cause

Ghidra's launcher is a cmd batch file, and `support/launch.bat` resolves the JVM by
capturing another Java process's stdout:

```bat
for /f "delims=*" %%i in ('call "%JAVA_CMD%" -cp "%LS_CPATH%" LaunchSupport "%INSTALL_DIR%" %JAVA_TYPE_ARG% -save') do set LS_JAVA_HOME=%%i
```

`LaunchSupport` prints the JDK path and cmd captures that line. This machine's
console code page is **936 (GBK)**, so a path containing non-ASCII characters is
mangled during that capture, and the launch then fails:

```
analyzeHeadless.bat : The system cannot find the path specified.
```

REA's own JDK probe reads the same corrupted output — which is exactly why it
classified a perfectly good JDK as "runtime only".

### It is the path *string*, not the directory

Measured on this machine. A directory junction to an ASCII name does **not** help,
because `LaunchSupport` emits the string it was configured with:

| `JAVA_HOME` set to | `analyzeHeadless.bat` | provider |
|---|---|---|
| `C:\Users\保安\...\vendor\jdk` | cannot find the path | `available: false` |
| `C:\rea-jdk-ascii-j` (junction → that same dir) | cannot find the path | `available: false` |
| `C:\Java\jdk` (real ASCII directory) | prints usage — works | `available: true` |

So the **string** must be ASCII. No aliasing trick can substitute.

After relocating the bundle, the same probe that previously said "runtime only"
now reports:

```json
{ "name": "java", "ok": true,
  "detail": "21.0.12.1; 64-bit; JDK; C:\\dsh-rea\\vendor\\jdk; accepted a 64-bit full JDK 21 or newer" }
```

Note `64-bit; JDK`. Fix procedure: [install-notes.md](install-notes.md#relocating-the-bundle).

---

## Finding 2 — `github.com` is unreachable

```
[ERROR] git ls-remote "https://github.com/czcdr/dsh-plugin-rea.git" HEAD "HEAD^{}"
fatal: unable to access ...: Failed to connect to github.com:443 after 21210 ms
```

Failed identically on three consecutive retries (~21 s each), so it is a stable
network block rather than a transient error.

| Host | Reachable |
|---|---|
| `github.com` (git, release downloads) | **no** |
| `api.github.com`, `codeload.github.com` | yes |
| `release-assets.githubusercontent.com`, `raw.githubusercontent.com` | yes |
| `registry.npmjs.org`, `nodejs.org`, `dl.google.com` | yes |

> **Caution:** `Test-NetConnection github.com -Port 443` reported **`True`** while
> real HTTP and git connections failed. Do not trust a TCP-connect probe here —
> test the actual protocol.

### Workaround A — install from the npm registry (preferred)

The plugin is published to npm, so the identical code is reachable without GitHub.
The published tarball's `gitHead` equals the GitHub `main` SHA:

```
registry  dsh-plugin-rea@2.0.1   gitHead = 3138bd15c3c8651380f3ef4406da0d1c08ec2fe1
GitHub    commits/main                    = 3138bd15c3c8651380f3ef4406da0d1c08ec2fe1
```

It also carries SLSA provenance. Check the two values yourself rather than trusting
this file.

```powershell
dsh plugin --profile desktop add dsh-plugin-rea@2.0.1
```

### Workaround B — fetch release assets through the API

The engine's setup script downloads Ghidra, a JDK, apktool and jadx from
`github.com/.../releases/download/...`, which is blocked. The `api.github.com`
asset endpoint is not, and redirects to a Range-capable CDN:

```
GET https://api.github.com/repos/NationalSecurityAgency/ghidra/releases/assets/579530646
    Accept: application/octet-stream
 -> 302 release-assets.githubusercontent.com/...
 -> 200  569649598 bytes, accept-ranges: bytes      (~9.8 MiB/s; 6 parallel chunks)
```

The setup script reuses anything already in `<root>/cache` after verifying size and
sha256, so pre-seed that cache and it skips the blocked download:

1. Resolve each asset id from `/repos/{owner}/{repo}/releases/tags/{tag}`
   (`setup.mjs --dry-run` prints the exact tags and URLs it wants).
2. `GET /repos/{owner}/{repo}/releases/assets/{id}` with
   `Accept: application/octet-stream`.
3. Save under the **basename of the original release URL** — that is the cache key.
4. Verify sha256 against `manifest/bundle.manifest.json`, which lists both the hash
   and its provenance (`sha256Source`).
5. Re-run setup; it logs `using cached <name>` per file.

Blocked on this machine: the JDK zip, Ghidra zip, apktool jar, jadx jar.
Not blocked: Node (nodejs.org), the apktool `.bat` (raw.githubusercontent.com),
platform-tools (dl.google.com).

---

## Finding 3 — the plugin's documented override shape is broken on this DSH

`examples/cordis.patch.with-env.yml` shows overriding the row like this:

```yaml
- id: mcp-rea
  config:
    env:
      GHIDRA_INSTALL_DIR: '<your Ghidra directory>'
```

On DSH 0.2.0-rc.2 a profile-level patch **replaces** the row's whole `config` — it
does not merge into it. That shape produced a row carrying only `env`, with
`command`, `args` and `transport` gone, so the row could not launch. Verified in an
isolated profile:

```
$ dsh --profile headless --patch <that shape>.yml --dump-config
- id: mcp-rea
  name: '@deepseek-ai/dsh-mcp-client'
  config:
    env:
      JAVA_HOME: C:\Java\jdk
```

DSH's own config schema agrees: *"A patch config replaces the whole config."*

So the row must be restated in full — see [rea-cordis-row.yml](rea-cordis-row.yml),
which also records a second trap: putting `JAVA_HOME` in the row's `env` has **no
effect**, because `bin/rea-shim.mjs` overwrites it from `bundle.json`:

```js
env = { ...process.env };
set("JAVA_HOME", paths.jdk);   // unconditional
```

`REA_BUNDLE_ROOT` is read *before* any `set()` call, which is why that is the
override that works.
