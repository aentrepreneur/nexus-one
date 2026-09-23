# Pipeline de Validacion — NEXUS ONE Licensing

## Entorno de Validacion

| Item | Valor |
|------|-------|
| Generador | WSL2 Ubuntu 24.04, `/opt/nexus-one/` |
| VPS destino | `<vps-ip>` (<vps-hostname>), Ubuntu 24.04 |
| Verificacion remota | `heartbeat-check.sh <usuario@host>` via SSH |
| Proyecto test | `amazon-light` (bash, 28 archivos) |
| Duracion inicial | 15 dias |
| Duracion renovacion | 30 dias |

## Test 1: Generacion de Harness con Licencia

```bash
prod-agent.sh --scan --force \
  --license "Lab-Validation-VPS" \
  --duration 15 \
  --heartbeat http://127.0.0.1:8443 \
  /opt/amazon-light
```

**Verificar archivos generados:**
- `LICENSE.D3.child` — Atribución D3.generator
- `AUTHORS` — Autoría
- `.license.token` — HMAC-SHA256 firmado (base64)
- `license-verify.sh` — Verificación fingerprint + expiración + heartbeat
- `license-verify.service` — Systemd oneshot unit
- `license-verify.timer` — Systemd timer (cada 6h, offset aleatorio 0-300s)
- `checkin.sh` — Métricas completas (uptime, RAM%, disk%, kernel)
- `renew.sh` — Token incrustado para renovación offline
- `boot.sh` — Script de arranque con check_security(), verify_license_token(), install_license_timer()
- `integrity.sum` — SHA256 de todos los archivos de configuracion

**Resultado:** PASS

## Test 2: Despliegue en VPS (boot.sh)

```bash
# Copiar harness
scp -r /opt/ag-amazon-light root@<vps-ip>:/opt/

# Nota: heartbeat-server fue reemplazado por heartbeat-check.sh (SSH)
# Ver seccion "Verificacion Remota de Licencia" en REMOTE-RENEWAL.md

# Instalar harness
cd /opt/ag-amazon-light && ./boot.sh
```

**Verificaciones boot.sh:**
- `[SECURITY] Sudo verificado` — sudo -n true || sudo -v
- `[SECURITY] Integridad verificada: OK` — sha256sum -c integrity.sum
- `[LICENSE] Cliente: ...` — Token decodificado correctamente
- `[LICENSE] Primer arranque. Activando licencia...` — Binding creado
- `[LICENSE] Licencia activada. Cliente: ... Expira: ...`
- `[BOOT] Usuario ag-<proyecto> creado` — Sin shell, sin sudo
- `[BOOT] Permisos configurados` — .agent/ root:root, src/ ag-usuario

**Archivos post-boot:**
- `.vps-binding.json` — Fingerprint + activated_at + expires_at
- Propietario root:root en: boot.sh, license-verify.*, checkin.sh, .license.token, .vps-binding.json

**Resultado:** PASS

## Test 3: Timer systemd

```bash
# Instalar timer
sudo cp license-verify.service /etc/systemd/system/
sudo cp license-verify.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now license-verify.timer

# Verificar
systemctl status license-verify.timer
journalctl -u license-verify.service --no-pager
```

**Resultado:** PASS — Timer activo, dispara cada 6h (00:00, 06:00, 12:00, 18:00)

## Test 4: Verificacion Remota via SSH

```bash
# Verificar por flag -h
SSH_PASS='<password>' bash /opt/nexus-one/heartbeat/heartbeat-check.sh -h <vps-user>@<vps-ip>
```

**Que verifica:**
1. Conexion SSH al VPS
2. Deteccion del directorio del harness (`/opt/ag-*/`)
3. Lectura de `.vps-binding.json` (binding actual)
4. Ejecucion de `license-verify.sh` (valida fingerprint + expiracion)

**Output esperado:**
```
[HEARTBEAT-CHECK] Conectando a <vps-user>@<vps-ip> ...
[HEARTBEAT-CHECK] Ejecutando license-verify.sh en remoto ...

[HEARTBEAT-CHECK] Harness detectado: /opt/ag-amazon-light

--- Binding Actual ---
{
  "vps_fingerprint": "...",
  "client": "Lab-Validation-VPS",
  "activated_at": "...",
  "expires_at": "...",
  "duration_days": 15
}

--- Verificacion de Licencia ---
[LICENSE-VERIFY] OK: Cliente=Lab-Validation-VPS Expira=... Restan=...d

[HEARTBEAT-CHECK] Estado: LICENCIA VALIDA
```

**Resultado:** PASS — Verificacion remota funciona sin servidor heartbeat

## Test 5: Renovacion Offline (renew.sh)

```bash
# En generador: regenerar token + renew.sh
prod-agent.sh --update /opt/ag-amazon-light \
  --license "Lab-Validation-VPS (Renovado)" \
  --duration 30

# SCP solo renew.sh al VPS
scp renew.sh root@<vps-ip>:/opt/ag-amazon-light/

# En VPS: ejecutar
sudo bash /opt/ag-amazon-light/renew.sh
```

**Lo que hace renew.sh:**
1. Detecta directorio del harness
2. Sobrescribe `.license.token` con el nuevo (root:root, 644)
3. Actualiza `.vps-binding.json`: client, duration_days, expires_at, activated_at
4. Ejecuta `license-verify.sh` para confirmar validez

**Verificacion post-renovacion:**
```bash
cat .vps-binding.json
# client: "Lab-Validation-VPS (Renovado)"
# duration_days: 30
# expires_at: 2026-08-01T12:04:31Z

HEARTBEAT_URL=http://127.0.0.1:8443 sudo -E bash license-verify.sh
# [LICENSE-VERIFY] OK: Cliente=... Restan=30d
```

**Resultado:** PASS — Binding actualizado sin regenerar todo el harness

## Test 6: Deteccion de Expiracion

```bash
# Simular expiracion
sudo python3 -c "
import json
with open('.vps-binding.json') as f:
    d = json.load(f)
d['expires_at'] = '2020-01-01T00:00:00Z'
json.dump(d, open('.vps-binding.json', 'w'), indent=2)
"

# Verificar
HEARTBEAT_URL=http://127.0.0.1:8443 sudo -E bash license-verify.sh
# Output esperado:
# [LICENSE-VERIFY] ERROR: Licencia expirada en 2020-01-01T00:00:00Z
# [LICENSE-VERIFY] AGENTE DETENIDO por licencia expirada.
# Exit code: 1
```

**Comportamiento de kill:**
- `sudo pkill -f "$AGENT_SCRIPT"` — Mata proceso del agente
- `sudo chmod 000 "$AGENT_SCRIPT"` — Bloquea ejecucion del agente
- Log en `license-verify.log`

**Resultado:** PASS

## Test 7: Deteccion de Fingerprint Mismatch (Anti-Copia)

```bash
# Simular copia a otro VPS
sudo python3 -c "
import json
with open('.vps-binding.json') as f:
    d = json.load(f)
d['vps_fingerprint'] = 'FAKE_FINGERPRINT_FOR_TEST'
json.dump(d, open('.vps-binding.json', 'w'), indent=2)
"

# Verificar
HEARTBEAT_URL=http://127.0.0.1:8443 sudo -E bash license-verify.sh
# Output esperado:
# [LICENSE-VERIFY] ERROR: Fingerprint no coincide. Posible copia a otro VPS.
# [LICENSE-VERIFY] Esperado: FAKE_FINGERPRINT_FOR_TEST
# [LICENSE-VERIFY] Actual:   1f2e4f78...
# [LICENSE-VERIFY] AGENTE DETENIDO por violacion de fingerprint.
# Exit code: 1
```

**Resultado:** PASS

## Resultados Consolidados

| Test | Resultado |
|------|-----------|
| 1. Generacion harness + licencia | PASS |
| 2. Despliegue VPS (boot.sh) | PASS |
| 3. Timer systemd (cada 6h) | PASS |
| 4. Verificacion remota (SSH) | PASS |
| 5. Renovacion offline (renew.sh) | PASS |
| 6. Deteccion de expiracion | PASS |
| 7. Fingerprint mismatch (anti-copia) | PASS |

## Incidencias Resueltas Durante Validacion

| Error | Causa | Fix |
|-------|-------|-----|
| `LICENSE_GEN_AT: unbound variable` | Variable no exportada en generator.sh | Cambiar a `$LICENSE_NOW` |
| `0: command not found` en check_kernel() | `$(apt list ...)` ejecutado en generacion, no escapado | Escapar con `\$` |
| `exit 22` en license-verify.service | `\\\"` en curl -d producia JSON invalido | Cambiar `\\\\\"` a `\"` en heredocs |
| `sudo -v` falla sin TTY | `sudo -v` requiere terminal interactiva | Primero intentar `sudo -n true` |
| renew.sh falla al ejecutar boot.sh como root | renew.sh corre como root, boot.sh rechaza root | renew.sh actualiza binding directamente |

# VALIDATION-PIPELINE.md — Full Pipeline Validation Reference
# =================================================================
# Procedimiento completo de validacion del sistema de licenciamiento
# Token HMAC + systemd timer + verificacion remota SSH.
# =================================================================
