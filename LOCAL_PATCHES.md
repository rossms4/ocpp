# Local patches — Ross's Jueclat / LFC charger fork

This fork of [`lbbrhzn/ocpp`](https://github.com/lbbrhzn/ocpp) carries a small set
of local modifications needed to talk to a **Jueclat / Lotus Flash Charge (LFC)
ESP32-based** wall charger. Everything else tracks upstream.

Baseline: **v0.10.17** (the version that was installed when the fork was made).
The three patches are the only difference between this `main` and upstream
`v0.10.17`.

## The patches

| # | Commit | File | What it does |
|---|--------|------|--------------|
| 1 | Add NUL-frame handling for ESP32-based chargers | `custom_components/ocpp/chargepoint.py` | Adds `route_message()` (strip a trailing/embedded `0x00` on receive) and `_send()` (append `0x00` on send). The charger's firmware NUL-terminates every OCPP frame and expects the same inbound; without this its frames don't parse and our replies are rejected. |
| 2 | Raise OCPP response_timeout 10s → 60s | `custom_components/ocpp/chargepoint.py` | `super().__init__(id, connection, 10)` → `60`. The charger answers CALLs slowly (worse while its web UI / log export is busy); at 10s the reply lands after the result future has already timed out and is discarded, so control commands look like they fail. |
| 3 | Lower TLS SECLEVEL to accept the charger's weak handshake | `custom_components/ocpp/api.py` | Appends `set_ciphers("ALL:@SECLEVEL=0")` to the server SSL context so OpenSSL 3 accepts the charger's low-strength TLS. **Currently vestigial** in Ross's setup because `socat` terminates TLS in front of HA (the integration runs with SSL off) — kept so the set is complete if the direct-TLS path is ever used. |

## Keeping up with upstream

Upstream is configured as the `upstream` remote. To pull a new upstream release
(e.g. `v0.10.18`) into this fork:

```bash
git fetch upstream --tags
git checkout main
git merge v0.10.18        # merge the RELEASE TAG, not upstream/main
# resolve any conflicts (only ever in the 2 files above), then:
git push origin main
```

Merging the **release tag** (not `upstream/main`) keeps you on published
releases instead of pulling in un-released work. The merge base is `v0.10.17`,
so only upstream's real delta is merged against the three patches.

If a merge conflicts (only possible if upstream heavily refactors
`chargepoint.py` / `api.py`), the three patches are small and self-contained —
re-apply them by hand from the table above, or `git rebase --onto <newtag>
v0.10.17 main` to replay them.

### After every update — verify the patches survived

```bash
grep -n "route_message\|async def _send"        custom_components/ocpp/chargepoint.py   # patch 1 present
grep -n "connection, 60)"                        custom_components/ocpp/chargepoint.py   # patch 2 present
grep -n "SECLEVEL=0"                             custom_components/ocpp/api.py           # patch 3 present
```

Then cut a new tag/release on the fork so HACS offers it (see SETUP.md).
