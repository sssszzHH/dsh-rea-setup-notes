# REA on DeepSeek Harness — setup notes

Wiring [`czcdr/dsh-plugin-rea`](https://github.com/czcdr/dsh-plugin-rea) (the REA
reverse-engineering MCP server) into DeepSeek Harness on Windows, including two
non-obvious failures and the evidence that identified each.

Observed 2026-10-15 · Windows `win32-x64` · DSH CLI 0.2.0-rc.2 · Node 24.21.0.
Every claim below is backed by a command result; nothing is inferred.

## Files

| File | Contents |
|---|---|
| [findings.md](findings.md) | The two root causes, with measured evidence |
| [install-notes.md](install-notes.md) | Install, provision, relocate, verify, uninstall |
| [rea-cordis-row.yml](rea-cordis-row.yml) | The profile patch that makes it work |

## Working end state

```
$ node C:\dsh-rea\bin\rea-shim.mjs mcp doctor
healthy: true
server:            rea 6.4.0, protocol 2025-11-25
inventory.tools:   expected 170, observed 170, missing []
inventory.prompts: expected 6,   observed 6
request_flow:      tool binary_session, ok true
```

| Component | Version | Where |
|---|---|---|
| `dsh-plugin-rea` | 2.0.1 | DSH `desktop` profile |
| REA CLI (`rea-agents`) | 6.4.0 | `C:\dsh-rea\vendor\node_modules\rea-agents` |
| Ghidra | 12.1.4 | `C:\dsh-rea\vendor\ghidra` |
| Temurin JDK | 21.0.12.1+1 | `C:\dsh-rea\vendor\jdk` |
| apktool / jadx-headless-mcp / adb | 3.0.3 / 0.7.1 / 37.0.1 | `C:\dsh-rea\vendor\...` |

## Two load-bearing deviations from a plain install

1. **The bundle lives at `C:\dsh-rea`**, not the default
   `<DSH_HOME>/plugins/rea-bundle`, because the default path contained non-ASCII
   characters — which silently disabled all native-binary analysis.
2. **Installed from the npm registry, not `github:`**, because `github.com` is
   unreachable from this machine. It is the same commit, verified by hash.

Both are explained in [findings.md](findings.md).

## Quick version

```powershell
# 1. Install the plugin (registry route; see findings.md for why)
dsh plugin --profile desktop add dsh-plugin-rea@2.0.1

# 2. Provision the engine into an ASCII-only root.
#    --root is mandatory, and use a Node OUTSIDE the bundle.
& "<external node>" C:\dsh-rea\scripts\setup.mjs --root C:\dsh-rea

# 3. Add the override row from rea-cordis-row.yml to
#    %USERPROFILE%\.dsh\profiles\desktop\cordis.patch.yml

# 4. Restart the harness, then verify via mcp__rea__binary_session:
#    analysis_provider_candidates[0].availability.status == "available"
```

Do **not** preinstall `npm install -g rea-agents`. Node/npm are not on `PATH` on
this machine, so a global `rea` would never be found, while the plugin vendors its
own Node (with npm) and REA inside the bundle. A global copy is unreachable
duplication.

## Optional components not installed

| Item | Why |
|---|---|
| Hopper | commercial, macOS-oriented — `HOPPER_LAUNCHER_PATH` |
| IDA | needs a registration JSON — `REA_IDA_MCP_CONFIG` |
| REA skill (`skill:identity`) | writes into `~/.agents/skills/` for *other* agents; left as a user decision |
| browser provider | set `REA_BROWSER_EXECUTABLE`; the bundle never downloads a browser |
| `extract_firmware`, `inspect_firmware_regions` | Linux-only by design (Unblob/Binwalk + `prlimit`) |

> `rea doctor`'s top-level `healthy` stays `false` until the optional REA skill is
> installed, even when the engine fully works. Treat the Ghidra provider's
> `availability.status`, or `rea mcp doctor`, as the real readiness signal.
