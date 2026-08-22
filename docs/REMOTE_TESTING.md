# REMOTE_TESTING.md -- Guia de Pruebas LLM en Servidores Remotos
# =================================================================
# Procedimiento paso a paso para validar proveedores LLM (OpenAI,
# OpenCode, Ollama) en VPS remoto. Incluye verificación de
# conectividad, autenticacion, conversation loop y seguridad.
# =================================================================

## 1. Pre-Requisitos

> **Nota**: Este documento asume que ya desplegaste un harness con `boot.sh`.
> Si no, primero segui la guia paso a paso en `docs/QUICKSTART.md` (seccion 4).

### Acceso al VPS
- SSH habilitado con usuario sudo
- Conexión estable a internet
- RAM minima: 2GB (para OpenAI), 4GB+ (para Ollama local)
- Disco disponible: 1GB+ (para descargar modelos Ollama)

### Herramientas Necesarias
- `curl` (instalado por defecto en Ubuntu 24.04)
- `sshpass` (para automatización SSH desde local)
- `jq` o `python3` (para parsear JSON en remoto)

### Variables de Entorno
| Variable | Uso | Ejemplo |
|----------|-----|---------|
| OPENAI_API_KEY | Autenticacion OpenAI | `sk-proj-...` |
| RUNTIME | Seleccion de proveedor | `openai`, `opencode`, `ollama`, `none` |

---

## 2. Seguridad de API Keys

### Reglas Obligatorias
1. **NUNCA** escribir API keys a disco dentro del harness
2. **SIEMPRE** usar variables de entorno (`export KEY=...`)
3. **NUNCA** loguear keys en stdout o archivos de log
4. **SIEMPRE** limpiar variables al finalizar (`unset KEY`)
5. **NUNCA** pasar keys como argumentos de línea de comandos

### Metodo Correcto
```bash
# Correcto: variable de entorno
export OPENAI_API_KEY="sk-proj-..."
curl -H "Authorization: Bearer $OPENAI_API_KEY" https://api.openai.com/...
unset OPENAI_API_KEY
```

### Metodo Incorrecto (PROHIBIDO)
```bash
# Incorrecto: key en archivo
echo "sk-proj-..." > .env

# Incorrecto: key en argumento
python3 script.py --api-key sk-proj-...

# Incorrecto: key en log
echo "Usando key: sk-proj-..." | tee -a log.txt
```

### Verificación Post-Prueba
```bash
# Confirmar que no hay keys en archivos del harness
grep -r "sk-proj-" /ruta/harness/ 2>/dev/null | wc -l
# Resultado esperado: 0
```

---

## 3. Test Rápido OpenAI (curl)

### Paso 3.1: Verificar Conectividad
```bash
curl -s -o /dev/null -w "%{http_code}" https://api.openai.com/v1/models
```
- **Esperado**: `401` (Unauthorized sin autenticacion, confirma red)
- **Si falla**: Verificar DNS, firewall, proxy

### Paso 3.2: Verificar Autenticacion
```bash
export OPENAI_API_KEY="sk-proj-..."
curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  https://api.openai.com/v1/models
```
- **Esperado**: `200` (lista de modelos disponible)
- **401**: API key invalida o expirada
- **403**: Cuenta suspendida o sin credito
- **429**: Rate limit alcanzado

### Paso 3.3: Test Chat Completion
```bash
curl -s \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Responde solo: OK"}],
    "max_tokens": 10
  }' \
  https://api.openai.com/v1/chat/completions
```
- **Esperado**: JSON con `choices[0].message.content = "OK"`
- **Costo**: ~$0.00001 (12 prompt tokens, 1 completion token)

### Paso 3.4: Verificar Modelos Disponibles
```bash
curl -s \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  https://api.openai.com/v1/models | python3 -c "
import json, sys
data = json.load(sys.stdin)
for m in data['data'][:10]:
    print(m['id'])
"
```

---

## 4. Test Agent.sh con OpenAI

### Paso 4.1: Configurar Entorno
```bash
cd /ruta/harness/ag-<proyecto>
export OPENAI_API_KEY="sk-proj-..."
export RUNTIME=openai
```

### Paso 4.2: Ejecutar Agente
```bash
# Modo interactivo (manual)
bash .agent/agent.sh

# Modo automatizado (pipe)
echo "Hola, confirma funcionamiento" | timeout 30 bash .agent/agent.sh
```
- **Esperado**: Banner con "Runtime: openai" + respuesta del LLM
- **Timeout**: 30 segundos para evitar bloqueo

### Paso 4.3: Verificar Historial
```bash
cat .agent/history.json | python3 -m json.tool
```
- **Esperado**: Array con mensajes user/assistant
- **Formato**: `[{"role": "user", "content": "..."}, {"role": "assistant", "content": "..."}]`

### Paso 4.4: Verificar Cost Tracking
```bash
ls -la .agent/cost* 2>/dev/null || echo "Sin cost tracking"
```
- **Nota**: Cost tracking puede no estar implementado para todos los proveedores

### Paso 4.5: Limpieza
```bash
unset OPENAI_API_KEY
unset RUNTIME
```

---

## 5. Test OpenCode

### Paso 5.1: Verificar OpenCode (boot.sh lo descarga)
```bash
# boot.sh --runtime opencode descarga el binary de GitHub:
ls -la models/bin/opencode        # Binary (~50MB, descargado de GitHub)
opencode --version                # Debe responder 1.x
```
- **Requisitos**: 4GB RAM, internet, sin API key necesaria
- **Instalación**: `install_runtime()` descarga opencode desde
  `github.com/anomalyco/opencode/releases/latest/download/opencode-linux-${arch}.tar.gz`
  directamente a `models/bin/opencode`. Sin writes a `/usr/` ni `/home/`.
- **Nota**: No requiere `curl | bash`. Todo dentro del harness (`models/bin/`).
  El PATH del agente incluye `$HARNESS_DIR/models/bin` automaticamente.

### Paso 5.2: Ejecutar Agente con OpenCode
```bash
cd /opt/ag-<proyecto>
export RUNTIME=opencode
echo "Hola, confirma funcionamiento" | timeout 60 bash .agent/agent.sh
```
- **Esperado**: Banner con "Runtime: opencode" + respuesta del LLM
- **Timeout**: 60 segundos para primera respuesta

---

## 6. Test Ollama

### Paso 6.1: Instalar Ollama (self-contained)
```bash
# boot.sh --runtime ollama descarga el binary a models/bin/ollama
# y los modelos a models/ollama/ (OLLAMA_MODELS local)
ls -la models/bin/ollama           # Binary (~38MB)
ls -la models/ollama/              # Modelos descargados aqui (no en ~/.ollama/)
models/bin/ollama list             # Lista modelos descargados
```
- **Requisitos**: 1GB descarga, 2GB RAM para modelo 1B
- **Modelos recomendados**:
  - `llama3.2:1b` - Ligero, 1GB RAM, rápido
  - `llama3.2:3b` - Balanceado, 3GB RAM
  - `llama3.1:8b` - Completo, 8GB RAM
- **Instalación**: `install_runtime()` descarga binary desde
  `https://ollama.com/download/ollama-linux-${arch}.tar.zst` (zst preferido,
  tgz fallback). Modelos se descargan a `models/ollama/` via
  `OLLAMA_MODELS=$HARNESS_DIR/models/ollama/`. Sin writes a `/usr/` ni `~/.ollama/`.
- **Nota**: Requiere `zstd` para .tar.zst. Si falta, fallback a .tgz (sin zstd).

### Paso 6.2: Ejecutar Agente con Ollama
```bash
cd /opt/ag-<proyecto>
export RUNTIME=ollama
export OLLAMA_MODEL=llama3.2:1b
echo "Hola, responde SOLO OK" | timeout 120 bash .agent/agent.sh
```
- **Esperado**: Banner con "Runtime: ollama" + respuesta local
- **Nota**: Primera respuesta puede tardar (modelo en memoria, CPU-only)
- **Timeout**: 120 segundos para modelo local sin GPU

---

## 7. Troubleshooting

| Error | Causa | Solucion |
|-------|-------|----------|
| HTTP 401 en /v1/models | Sin auth o key invalida | Verificar `echo $OPENAI_API_KEY` |
| HTTP 403 | Cuenta suspendida | Revisar dashboard.openai.com |
| HTTP 429 | Rate limit | Esperar 60s, reducir frecuencia |
| HTTP 500 | Error interno OpenAI | Reintentar en 5 minutos |
| Connection refused | Sin internet o firewall | `ping api.openai.com` |
| DNS resolution failed | DNS mal configurado | `nslookup api.openai.com` |
| agent.sh no responde | RUNTIME no detectado | Verificar `detect_runtime` en agent.sh |
| "No LLM available" | Ningun provider disponible | Instalar Ollama o configurar API key |
| Timeout en curl | Red lenta o bloqueada | Aumentar timeout, verificar proxy |
| grep encuentra API key | Key escrita a disco | Eliminar archivo, cambiar metodo |

---

## 8. Costos Estimados

| Proveedor | Modelo | Input/1M tokens | Output/1M tokens | Costo Prueba |
|-----------|--------|-----------------|------------------|--------------|
| OpenAI | gpt-4o-mini | $0.15 | $0.60 | ~$0.00001 |
| OpenAI | gpt-4o | $2.50 | $10.00 | ~$0.00025 |
| OpenAI | o3-mini | $1.10 | $4.40 | ~$0.00011 |
| Ollama | llama3.2:1b | $0 | $0 | $0 (local) |
| Ollama | llama3.2:3b | $0 | $0 | $0 (local) |

### Presupuesto Recomendado
- Prueba unica: $0.001 USD (10 llamadas a gpt-4o-mini)
- Validación completa: $0.01 USD (100 llamadas)
- Uso continuo: $1-5 USD/mes (segun volumen)

---

## 9. Checklist Post-Prueba

- [ ] Conectividad a API verificada (HTTP 401 sin auth)
- [ ] Autenticacion exitosa (HTTP 200 con key)
- [ ] Chat completion funciona (respuesta JSON valida)
- [ ] Agent.sh responde con runtime correcto
- [ ] Historial persistido en .agent/history.json
- [ ] API key NO escrita a disco (grep = 0 resultados)
- [ ] Variables de entorno limpiadas (unset)
- [ ] Costo dentro de presupuesto esperado
- [ ] Documentación actualizada si hay cambios

### Validación Self-Contained

Estos checks verifican que el runtime es 100% autocontenido:

- [ ] `models/bin/ollama` existe y es ejecutable (si runtime ollama)
- [ ] `models/bin/opencode` existe (symlink o binary) (si runtime opencode)
- [ ] `models/ollama/` contiene modelos descargados (no en `~/.ollama/`)
- [ ] No hay binaries en `/usr/local/bin/` escritos por boot.sh
- [ ] `OLLAMA_MODELS` apunta a `$HARNESS_DIR/models/ollama/` (no default)
- [ ] API keys solo en variables de entorno, no en archivos del harness
- [ ] `PATH` del agente incluye `$HARNESS_DIR/models/bin`

---

## 10. Validación Completa (Fases A-G)

### Fase A: boot.sh End-to-End
| Paso | Verificación | Estado |
|------|--------------|--------|
| Crear usuario ag-<proyecto> | `id ag-custom-2026` muestra uid/gid | PASS |
| Setear permisos | root:root config, ag-<proyecto> src/ | PASS |
| Instalar servicio systemd | `systemctl is-enabled` = enabled | PASS |
| detect.sh post-setup | 17 checks OK (RAM, Disk, CPU, Docker, Internet) | PASS |

### Fase B: Conversation Loop Multi-Turn
| Paso | Verificación | Estado |
|------|--------------|--------|
| Mensaje 1: "Hola, confirma funcionamiento" | Respuesta en espanol | PASS |
| Comando /status | Muestra RAM, Disco, Uptime, PID, Sesion | PASS |
| Mensaje 2: "Cual es 2+2?" | Respuesta "4" con contexto | PASS |
| Comando /help | Muestra todos los comandos disponibles | PASS |
| Comando /exit | Cierra sesion graceful | PASS |
| Historial JSON | 4 mensajes persistidos (2 user, 2 assistant) | PASS |

### Fase C: Tools Plugin
| Tool | Verificación | Estado |
|------|--------------|--------|
| hello.sh | Definicion HELLO_DESCRIPTION, hello_run() | PASS |
| scan_secrets.sh | 25 patrones regex, ejecución directa | PASS |
| audit_deps.sh | Detecta lenguaje, audit deps | PASS |
| verify_process.sh | Verifica procesos activos | PASS |
| extract_secrets.sh | Extrae candidatos a .env | PASS |
| **Nota**: Tools con `main "$@"` ejecutan al ser sourced. Fix aplicado: execute_tool() usa `bash "$plugin_file"` en vez de `source`. | | |

### Fase D: Systemd Lifecycle
| Paso | Verificación | Estado |
|------|--------------|--------|
| systemctl start | Service inicia (activating) | PASS |
| systemctl status | Muestra ExecStart, PID, logs | PASS |
| journalctl -u | Logs en journald | PASS |
| systemctl stop | Service inactive | PASS |
| systemctl disable | Service disabled | PASS |
| **Issue**: CHDIR error en /home/angelo/ (750). En /opt/ (755) funciona. | | |
| **Issue**: IPAddressAllow con dominios invalidos. Fix: usar 0.0.0.0/0. | | |

### Fase E: Uninstall.sh
| Paso | Verificación | Estado |
|------|--------------|--------|
| Pre: User exists | `id ag-custom-2026` = YES | PASS |
| Pre: Service exists | `systemctl list-unit-files` = 1 | PASS |
| Pre: Harness exists | `ls -d` = YES | PASS |
| Ejecutar uninstall.sh | 6 pasos completados | PASS |
| Post: User deleted | `id ag-custom-2026` = NO | PASS |
| Post: Service removed | `systemctl list-unit-files` = 0 | PASS |
| Post: Harness removed | `ls -d` = NO | PASS |
| Post: Journald cleaned | Logs eliminados | PASS |

### Fase F: Seguridad
| Test | Verificación | Estado |
|------|--------------|--------|
| Restricted command: `rm -rf /etc` | Bloqueado: "No puedo ejecutar ese comando..." | PASS |
| Restricted path: `/root/.ssh/authorized_keys` | Bloqueado: "No tengo permitido acceder..." | PASS |
| Ownership check | limits.conf root:root 644 | PASS |
| Integrity check | integrity.sum validado | PASS |

### Fase G: Error Recovery
| Escenario | Verificación | Estado |
|-----------|--------------|--------|
| Sin API key (RUNTIME=openai) | "ERROR: OPENAI_API_KEY no configurada" | PASS |
| Recovery mode (RUNTIME=none) | Muestra menu recovery con opciones | PASS |
| Recovery mode (RUNTIME=none) | Muestra menu recovery con opciones | PASS |

---

## 11. Resultados de Validación Completa v0.16.0

### Fecha: 2026-05-27
### VPS: <vps-hostname> (<vps-ip>)
### Config: Ubuntu 24.04, 7GB RAM, 8 cores, Docker, Internet

### Resumen por Fase
| Fase | Tests | PASS | FAIL | PENDIENTE |
|------|-------|------|------|-----------|
| A: boot.sh End-to-End | 4 | 4 | 0 | 0 |
| B: Conversation Loop | 6 | 6 | 0 | 0 |
| C: Tools Plugin | 5 | 5 | 0 | 0 |
| D: Systemd Lifecycle | 6 | 6 | 0 | 0 |
| E: Uninstall.sh | 8 | 8 | 0 | 0 |
| F: Seguridad | 4 | 4 | 0 | 0 |
| G: Error Recovery | 3 | 3 | 0 | 0 |
| OpenCode Runtime | 2 | 2 | 0 | 0 |
| Ollama Runtime | 2 | 2 | 0 | 0 |
| **TOTAL** | **40** | **40** | **0** | **0** |

### Issues Resueltos
| Issue | Fase | Solucion Aplicada | Estado |
|-------|------|-------------------|--------|
| CHDIR error en /home/angelo/ (750) | D | Deploy en /opt/ (755) | RESUELTO |
| IPAddressAllow con dominios invalidos | D | Fix: usar 0.0.0.0/0 | RESUELTO |
| Recovery mode input parsing | G | Case-insensitive, skip empty, /exit | RESUELTO |

### Validación Final en /opt/
| Test | Resultado | Detalle |
|------|-----------|---------|
| Systemd start desde /opt/ | PASS | Sin CHDIR error, sin IPAddressAllow warnings |
| Systemd status | PASS | ExecStart=/opt/ag-custom-2026/.agent/agent.sh |
| Journald logs | PASS | Sin errores de permisos |
| Recovery mode (empty input) | PASS | Skip graceful, sin error |
| Recovery mode (DIAGNOSE) | PASS | Case-insensitive, muestra diagnostico |
| Recovery mode (Status) | PASS | Muestra RAM, Disco |
| Recovery mode (/exit) | PASS | Cierra sesion correctamente |
| OpenAI conversation desde /opt/ | PASS | Runtime: openai, historial persistido |
| OpenCode conversation desde /opt/ | PASS | Runtime: opencode, respuesta via opencode run |
| Ollama conversation desde /opt/ | PASS | Runtime: ollama, modelo llama3.2:1b, respuesta local |
| Seguridad API key | PASS | 0 archivos con key expuesta |

### Fixes Aplicados en v0.16.0
| Fix | Archivo | Descripción |
|-----|---------|-------------|
| OpenCode --prompt -> run | agent.sh | `opencode --prompt -` no existe en v1.15; usar `opencode run` |
| Payload quoting env vars | agent.sh | `'''$var'''` fragile con comillas simples; usar PYTHON_* env vars |
| stream:false -> False | agent.sh | Python necesita `False` (no `false`) para json.dumps |

### Costo Total: ~$0.00001 USD (1 llamada a gpt-4o-mini)
OpenCode y Ollama: $0 USD (locales/gratuitos)

---

#End Development By Angel Esquivel (CyberSecurity) [NEXUS ONE 2026]
