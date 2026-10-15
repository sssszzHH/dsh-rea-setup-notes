# Install and maintenance notes

Operational detail for installing `dsh-plugin-rea` and provisioning the REA engine,
including the workarounds this machine needed. See [README.md](README.md) for the
two root-cause investigations.

## 1. Install the plugin

Preferred, and what was used here:

```powershell
dsh plugin --profile desktop add dsh-plugin-rea@2.0.1
```

Only fall back to the git form if the registry is unreachable:

```powershell
dsh plugin --profile desktop add github:czcdr/dsh-plugin-rea
```

Both resolve to the same commit. Verify that yourself rather than trusting it:

* npm: `https://registry.npmjs.org/dsh-plugin-rea/latest` → read `gitHead`
* GitHub: `https://api.github.com/repos/czcdr/dsh-plugin-rea/commits/main` → read `sha`

The two must be equal.

**Do not** run `npm install -g rea-agents` as a prerequisite. It is not needed and is
actively unhelpful here: Node/npm are not on `PATH` on this machine, so a global
`rea` would never be found by the harness, while the plugin vendors its own Node
(including npm) and REA inside the bundle. A global install only adds an unreachable
second copy.

## 2. Provision the engine

```powershell
# Run from a directory unrelated to the bundle, and with a Node OUTSIDE the bundle.
& "C:\path\to\external\node.exe" C:\dsh-rea\scripts\setup.mjs --root C:\dsh-rea
```

Two requirements, both learned the hard way:

* **`--root` is mandatory.** Omit it and the script derives the root from `DSH_HOME`,
  rebuilding at the original non-ASCII path and reintroducing the `JAVA_HOME`
  problem described in [README.md](README.md#finding-1-the-non-ascii-java_home).
* **Use an external Node.** Running it with the bundle's own
  `vendor/node/node.exe` fails partway through, because the script deletes and
  re-extracts `vendor/node` while executing from it:

  ```
  Error: EPERM, Permission denied: \\?\C:\dsh-rea\vendor\node
  ```

Useful flags:

| Flag | Effect |
|---|---|
| `--dry-run` | print the plan, download nothing |
| `--reuse` | adopt existing node/ghidra/jdk found on the machine instead of duplicating |
| `--offline` | refuse network; satisfy only from cache/reuse |
| `--only <a,b,c>` | restrict to named components |
| `--json` | machine-readable result |

## 3. When `github.com` is blocked

`api.github.com` is reachable even when `github.com` is not, and its release-asset
endpoint 302-redirects to a CDN that supports Range requests. The setup script
accepts anything already in `<root>/cache` after verifying size + sha256, so
pre-seeding the cache skips the blocked download entirely.

For each asset:

```powershell
# 1. Find the asset id and expected size for the pinned tag.
#    GET /repos/{owner}/{repo}/releases/tags/{tag}
#    (the exact tags the installer wants are printed by setup.mjs --dry-run)

# 2. Download through the API route, NOT the browser_download_url.
#    GET https://api.github.com/repos/{owner}/{repo}/releases/assets/{id}
#        Accept: application/octet-stream     -> 302 -> CDN, Range-capable

# 3. Save as the BASENAME of the original release URL — that is the cache key.
#    e.g. .../releases/download/Ghidra_12.1.4_build/ghidra_12.1.4_PUBLIC_20260921.zip
#         -> <root>\cache\ghidra_12.1.4_PUBLIC_20260921.zip

# 4. Verify sha256 against manifest/bundle.manifest.json before trusting it.
#    The manifest gives both the hash and its provenance ("sha256Source").
```

Then re-run setup; it logs `using cached <name>` for each pre-seeded file.

Assets the plan needed on this machine (win32-x64, plugin 2.0.1):

| Asset | Source host | Blocked? |
|---|---|---|
| `node-v24.21.0-win-x64.zip` | nodejs.org | no |
| `OpenJDK21U-jdk_x64_windows_hotspot_21.0.12.1_1.zip` | github.com releases | **yes** |
| `ghidra_12.1.4_PUBLIC_20260921.zip` | github.com releases | **yes** |
| `apktool_3.0.3.jar` | github.com releases | **yes** |
| `apktool.bat` | raw.githubusercontent.com | no |
| `jadx-headless-mcp-0.7.1-all.jar` | github.com releases | **yes** |
| `platform-tools-latest-windows.zip` | dl.google.com | no |

Observed throughput through the API route: ~9.8 MiB/s, with six parallel chunks.

## 4. Relocating the bundle to an ASCII root

Only needed when the default root contains non-ASCII characters. The default is
`<DSH_HOME>/plugins/rea-bundle`; on this machine `DSH_HOME` is
`C:\Users\保安\.dsh`, so the default root is unusable for the reason in the README.

Outline of what was done, in a safe order:

1. Copy the bundle's `bin/`, `scripts/`, `manifest/`, `cache/` to the new root
   (`robocopy /E`).
2. Move `vendor/*` to the new root.
3. Run setup with `--root <new root>` using an **external** Node, so `bundle.json`
   is regenerated with the new paths. It reuses the cache, so this is fast.
4. Confirm every path the shim will export is ASCII:

   ```powershell
   $n = "C:\dsh-rea\bin\rea-shim.mjs"
   & "<node>" $n --shim-status      # inspect "paths" — all must be ASCII
   ```

5. Add the override from [rea-cordis-row.yml](rea-cordis-row.yml) to the profile.
6. Restart the harness, then confirm via `mcp__rea__binary_session`.
7. Only after that, delete the old bundle (it is ~930 MiB including its cache).

The old bundle can be deleted safely once step 6 passes; nothing else references it.

## 5. Health checks

```powershell
# Full protocol handshake through the shim — the authoritative engine check.
# Verify inventory counts and that request_flow.binary_session is ok.
& "<node>" C:\dsh-rea\bin\rea-shim.mjs mcp doctor

# Environment/provider detail without a full handshake.
& "<node>" C:\dsh-rea\vendor\node_modules\rea-agents\scripts\rea.mjs doctor

# What the shim resolved, and where.
& "<node>" C:\dsh-rea\bin\rea-shim.mjs --shim-status
```

`rea doctor`'s top-level `healthy` stays `false` until the optional REA skill is
installed. Treat the Ghidra provider's `availability.status`, or
`rea mcp doctor`'s `healthy: true`, as the real signal.

## 6. Optional extras

```powershell
# Install the bundled REA skill into ~/.agents/skills (the only action it plans
# on a machine with no other supported agents detected):
& "<node>" C:\dsh-rea\vendor\node_modules\rea-agents\scripts\rea.mjs setup --all-detected --yes

# Preview first — this writes into OTHER agents' config directories.
& "<node>" ...\rea.mjs setup --all-detected --yes --dry-run
```

Not installed by default, and why:

* **Hopper** — commercial, macOS-oriented. `rea setup --install-hopper` or
  `HOPPER_LAUNCHER_PATH`.
* **IDA** — needs an existing registration JSON; `REA_IDA_MCP_CONFIG`.
* **`ilspycmd`** — .NET targets only.
* **Browser provider** — set `REA_BROWSER_EXECUTABLE`; the bundle records
  `paths.browser: null` and never downloads a browser.
* **`extract_firmware` / `inspect_firmware_regions`** — Linux-only (need Unblob /
  Binwalk and `prlimit`); unavailable on Windows by design, not by misconfiguration.

## 7. Uninstall

```powershell
dsh plugin --profile desktop remove dsh-plugin-rea
# then remove the profile override row
# then, if you want the disk space back:
Remove-Item C:\dsh-rea -Recurse -Force
```

## 8. Version pins used here

| Component | Version |
|---|---|
| DSH CLI | 0.2.0-rc.2 |
| `dsh-plugin-rea` | 2.0.1 |
| `rea-agents` | 6.4.0 |
| Ghidra | 12.1.4 (`Ghidra_12.1.4_build`) |
| Temurin JDK | 21.0.12.1+1 |
| apktool | 3.0.3 |
| jadx-headless-mcp | 0.7.1 |
| platform-tools | 37.0.1 |
| bundled Node | 24.21.0 |
| host | Windows, `win32-x64`, console code page 936 |
