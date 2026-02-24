# Home Assistant Working Configurations

**Last Updated:** 2026-02-24

## Integration Status

| Integration | Status | Notes |
|-------------|--------|-------|
| SmartThings (Samsung AC) | WORKING | 2 ACs: el_brisas_cuarto, el_brisas_living |
| Xiaomi Miot | WORKING | Cloud mode, server: de |
| Govee | WORKING | Rate limited - wait 24h, polling: 60s |
| Sonoff | CHECK | May have network issues |

---

## SmartThings Configuration
- **Entry ID:** (re-added by user via OAuth)
- **Location:** Mi hogar
- **Devices:** 2x Samsung AC

---

## Xiaomi Miot Configuration
- **Entry ID:** 01KJ1VQ1K72B0409KPE9741KCE
- **Connection Mode:** cloud (NOT local)
- **Server:** de (Germany)
- **Username:** jlabreu24@gmail.com

### Xiaomi Devices (from token_extractor)
| Device | Model | IP | Token | Status |
|--------|-------|-----|-------|--------|
| Succionadora (Vacuum) | xiaomi.vacuum.ov31gl | 192.168.1.19 | 6f44356473775276514e563658785138 | WORKING |
| La Calentita (Air Fryer) | xiaomi.fryer.maf07d | 192.168.1.29 | 116d768d9e4161f85f7a1e4d1ff054b4 | WORKING |
| Calefon (Smart Plug) | cuco.plug.v2eur | 192.168.1.4 | 96de3d67d157baed69088b5c47d15e50 | WORKING |
| El ojo de dios (Camera) | chuangmi.camera.046c04 | 192.168.1.8 | 76703456624353743734575353337750 | WORKING |
| La Remolina (Juicer) | chunmi.juicer.uka1 | 192.168.1.23 | da10e8052827195a4e46fdc4d6b231e3 | CHECK |

---

## Govee Configuration
- **API Key:** 4a86f028-28f8-40be-abc9-a9b4958271be
- **Polling Delay:** 60 seconds
- **Lights:** Cocina, Lampara living, Cuarto (H6008 models)

---

## Known Issues

1. **Network timing during startup:** HA sometimes fails to connect to cloud APIs during startup. Usually works after restart.
2. **Govee rate limiting:** 10,000 requests/day limit. Wait between commands.

---

## If Something Breaks

### SmartThings not working:
1. Go to Settings → Devices & Services
2. Delete SmartThings
3. Re-add SmartThings via OAuth

### Xiaomi devices unavailable:
1. Check if device IP changed (run token_extractor.py)
2. Restart HA
3. If still broken, ensure conn_mode is "cloud" and server is "de"

### Govee not responding:
1. Wait 2-3 seconds between commands
2. If completely broken, wait 24h for rate limit reset
