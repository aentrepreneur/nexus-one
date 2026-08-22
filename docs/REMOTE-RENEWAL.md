# Reactivacion Remota de Licencias — NEXUS ONE

## Modelo de Licenciamiento

Cada harness (`/opt/ag-<proyecto>/`) contiene un token firmado HMAC que define:

| Campo | Descripcion |
|-------|-------------|
| `client` | Nombre del cliente asignado via `--license` |
| `generated_at` | Fecha de generacion del token |
| `duration_days` | Duracion en dias desde activacion |

La **activacion** ocurre en el primer `./boot.sh` del VPS cliente:

1. boot.sh genera un fingerprint del VPS (hostname + hostid + MAC + disco)
2. Crea `.vps-binding.json` con `activated_at = now` y `expires_at = activated_at + duration_days`
3. Cada 6h, `license-verify.timer` (systemd) ejecuta `license-verify.sh`
4. Si expiro o el fingerprint no coincide → agente detenido

## Procedimiento A: Renovacion via renew.sh (Recomendado)

Un solo archivo. No requiere regenerar todo el harness ni copiar multiples archivos.

### Paso 1: Generar nuevo token + renew.sh

```bash
# En tu servidor generador (NEXUS ONE)
prod-agent.sh --update /opt/ag-clienteX \
  --license "ClienteX (Renovado Dic 2026)" \
  --duration 365 \
  --heartbeat http://TU_IP:8443
```

Esto regenera:
- `.license.token` (nuevo cliente, nueva duracion)
- `renew.sh` (con el nuevo token incrustado)
- `license-verify.sh`, `checkin.sh`

### Paso 2: Enviar renew.sh al VPS del cliente

```bash
scp /opt/ag-clienteX/renew.sh usuario@vps-cliente:~
```

Un solo archivo. El token va incrustado dentro del script.

### Paso 3: Cliente ejecuta renew.sh

```bash
ssh usuario@vps-cliente
sudo bash renew.sh
```

**Lo que hace renew.sh internamente:**
1. Detecta el directorio del harness (auto-scan `/opt/ag-*/`)
2. Sobrescribe `.license.token` con el nuevo token firmado
3. Actualiza `.vps-binding.json`: cambia `client`, `duration_days`, `expires_at`, `activated_at`
4. Ejecuta `license-verify.sh` para confirmar que la renovacion es valida
5. No requiere internet ni heartbeat — funciona 100% offline

## Procedimiento B: Extension Manual (Solo Emergencia)

Usar solo si NO puedes regenerar el token pero confias en el cliente.

```bash
# En el VPS del cliente (como root o con sudo):
sudo nano /opt/ag-clienteX/.vps-binding.json
# Cambiar "expires_at" a nueva fecha, ej:
#   "expires_at": "2027-06-01T12:00:00Z"

# Luego reiniciar el timer para que recoja el cambio:
sudo systemctl restart license-verify.timer
```

**Advertencia**: Esto no cambia el token original. Solo extiende la fecha en el binding local. Usar solo en emergencias y documentar en el changelog del cliente.

## Procedimiento C: Regeneracion Completa del Harness

Cuando necesitas cambiar configuracion del harness ademas de la licencia:

```bash
# En tu servidor
prod-agent.sh --scan --force /opt/proyecto \
  --license "ClienteX (v2)" \
  --duration 365 \
  --heartbeat http://TU_IP:8443

# Enviar todo el harness
scp -r /opt/ag-proyecto usuario@vps-cliente:~/

# Cliente: ejecutar boot.sh desde cero
ssh usuario@vps-cliente './ag-proyecto/boot.sh'
```

## Procedimiento D: Verificacion Remota de Licencia (SSH)

Verifica el estado de la licencia en cualquier VPS cliente via SSH.
No requiere servidor heartbeat ni Python.

```bash
# Verificar por flag
bash /opt/nexus-one/heartbeat/heartbeat-check.sh -h usuario@vps-cliente

# Si no se pasa -h, pregunta interactivamente:
bash /opt/nexus-one/heartbeat/heartbeat-check.sh

# Password via env (o flag -p):
export SSH_PASS='<password>'
bash /opt/nexus-one/heartbeat/heartbeat-check.sh -h <vps-user>@<vps-ip>

# Ejemplo con lab:
SSH_PASS='<vps-password>' bash /opt/nexus-one/heartbeat/heartbeat-check.sh -h <vps-user>@<vps-ip>
```

**Que hace heartbeat-check.sh internamente:**
1. Conecta via SSH al VPS cliente
2. Detecta automaticamente el directorio del harness (`/opt/ag-*/`)
3. Muestra el `.vps-binding.json` actual
4. Ejecuta `sudo license-verify.sh` para validar fingerprint + expiracion
5. Reporta estado: LICENCIA VALIDA o LICENCIA INVALIDA

## Seguridad

| Mecanismo | Que Protege |
|-----------|-------------|
| Fingerprint VPS | Impide copiar harness a otro servidor |
| Fecha expiracion | Impide uso despues del vencimiento |
| integrity.sum | Detecta manipulacion de archivos del harness |
| root:root | Cliente no puede modificar archivos de licencia sin sudo |
| systemd timer | Verificacion automatica cada 6h, dificil de desactivar |

## Variables de Entorno (Generador)

| Variable | Propósito |
|----------|-----------|
| `LICENSE_HMAC_KEY` | Clave HMAC para firmar tokens (en `/opt/nexus-one/.env`) |
| `HEARTBEAT_URL` | URL del servidor heartbeat (ej: `http://1.2.3.4:8443`) |

Si `LICENSE_HMAC_KEY` no esta definida, se genera una temporal.
Para persistencia entre generaciones:

```bash
echo 'LICENSE_HMAC_KEY="<clave-secreta-64-caracteres>"' >> /opt/nexus-one/.env
chmod 600 /opt/nexus-one/.env
```

Puedes generar una clave con:

```bash
python3 -c "import secrets; print(secrets.token_hex(32))"
```

## Produccion — Checklist

- [ ] `LICENSE_HMAC_KEY` definida en `/opt/nexus-one/.env` (persistente)
- [ ] `heartbeat-check.sh` verifica VPS remoto correctamente
- [ ] Prueba de renovacion: demo 15d → produccion 365d
- [ ] Prueba de expiracion: esperar a que venza el token y verificar kill
- [ ] Prueba de fingerprint: copiar harness a otro VPS y verificar rechazo
- [ ] Backup de `.env` (contiene HMAC key)
