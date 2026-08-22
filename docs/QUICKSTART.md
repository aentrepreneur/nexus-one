# QUICKSTART.md -- Guia Practica Paso a Paso
# =================================================================
# Solo comandos. Sin teoria. Para usuarios nuevos que ya tienen un
# proyecto en /opt/ y quieren generar un harness o remediar código.
# =================================================================

## Requisitos

- Bash 4+, rsync, sudo instalados
- Proyecto existente en `/opt/mi-proyecto/`

## 1. Ver Preview Sin Riesgos

```bash
cd /opt/nexus-one/prod-agent
./prod-agent.sh --scan --dry-run /opt/mi-proyecto
# Muestra: lenguaje detectado, archivos, arbol del harness
# NO escribe nada, NO modifica nada
```

## 2. Generar un Harness

```bash
cd /opt/nexus-one/prod-agent
./prod-agent.sh --scan /opt/mi-proyecto
# Resultado: /opt/ag-mi-proyecto/ creado
```

Si el harness ya existe, aparece menu con 4 opciones:
`[1] Sobrescribir [2] Actualizar [3] Nuevo+Respaldo [4] Cancelar`

## 3. Validar el Harness

```bash
./prod-agent.sh --validate /opt/ag-mi-proyecto
# Resultado esperado: 7 checks PASS
```

## 4. Desplegar en un VPS

```bash
# === 4.1: SCP el harness al VPS ===
scp -r /opt/ag-mi-proyecto usuario@vps:~

# === 4.2: Configurar API keys (NUNCA a disco) ===
# Exportar solo en la sesion, jamas escribir al harness
export ANTHROPIC_API_KEY="sk-ant-..."
export OPENAI_API_KEY="sk-proj-..."
export GEMINI_API_KEY="..."

# === 4.3: Ejecutar boot.sh ===
ssh usuario@vps
cd ~/ag-mi-proyecto
./boot.sh
# boot.sh: verifica integridad, crea usuario ag-*, detecta recursos,
#          menu interactivo de runtime, instala binarios, lanza agente

# === 4.4: Forzar runtime especifico ===
# Saltar el menu interactivo con --runtime
./boot.sh --runtime opencode    # OpenCode local (sin API key)
./boot.sh --runtime claude      # Claude API
./boot.sh --runtime gemini      # Gemini API
./boot.sh --runtime ollama      # Ollama local (descarga modelo)
./boot.sh --runtime openai      # OpenAI API
./boot.sh --runtime none        # Recovery / sin LLM

# === 4.5: Verificar que el agente esta vivo ===
# Si boot.sh termina, el agente se ejecuta como usuario ag-<proyecto>
# Para pruebas automatizadas de cada proveedor:
# Ver REMOTE_TESTING.md
```

## 5. Usar el Agente

Una vez desplegado, el agente presenta el prompt:

```
nexus>
```

### 5.1: Dar Tareas en Lenguaje Natural

```
nexus> escanea este proyecto en busca de secretos
nexus> genera documentacion para el modulo principal
nexus> cual es el estado del servidor?
nexus> ejecuta los tests y reporta resultados
```

### 5.2: Comandos Especiales

| Comando | Que hace |
|---------|----------|
| `/help` | Lista comandos y herramientas disponibles |
| `/debug` | Activa/desactiva modo debug (muestra llamadas LLM) |
| `/cost` | Muestra costo acumulado de la sesion |
| `/exit` | Cierra la sesion del agente graceful |

### 5.3: Herramientas del Agente

El agente carga automaticamente las tools en `.agent/tools/`:

| Tool | Proposito |
|------|-----------|
| `hello.sh` | Tool de ejemplo: saludo + echo |
| `scan_secrets.sh` | 25 patrones regex de deteccion de secretos |
| `extract_secrets.sh` | Migra claves del codigo a .env |
| `audit_deps.sh` | Audita dependencias (pip/npm audit) |
| `verify_process.sh` | Descubre y verifica procesos del proyecto |

### 5.4: Ejemplos de Uso

```
nexus> /help
[AGENT] Comandos: /help, /debug, /cost, /exit
[AGENT] Tools: hello, scan_secrets, audit_deps, verify_process, extract_secrets

nexus> ejecuta scan_secrets en src/
[AGENT] Escaneando src/ en busca de secretos...
[AGENT] 0 hallazgos en 12 archivos

nexus> /cost
[AGENT] Costo sesion: $0.00023 USD (3 llamadas)

nexus> /exit
[AGENT] Cerrando sesion. Hasta luego.
```

### 5.5: Salir y Revisar Logs

```bash
# Desde dentro del agente
/exit

# Desde fuera (SSH)
cat .agent/audit.log       # Auditoria JSON de toda la sesion
cat .agent/history.json    # Historial de mensajes
ls .agent/tasks/           # Tareas pendientes/completadas
```

## 6. Remediar un Proyecto

```bash
cd /opt/nexus-one/remediation

# Preview de hallazgos (sin cambios)
./remediation.sh --remediate --dry-run /opt/mi-proyecto

# Ciclo completo con cambios interactivos
./remediation.sh --remediate /opt/mi-proyecto
# 6 fases: snapshot -> discover -> baseline -> analyze -> remediate -> report

# Ver remedios generados
./remediation.sh --list
./remediation.sh --remedy mi-proyecto
```

## 7. Comandos de Referencia Rápida

| Comando | Que hace |
|---------|----------|
| `--scan --dry-run /ruta` | Preview del harness sin escribir nada |
| `--scan /ruta` | Genera harness completo |
| `--scan --force /ruta` | Sobrescribe sin preguntar |
| `--scan --update /ruta` | Regenera config, conserva src/ |
| `--validate /opt/ag-*` | Valida harness (7 checks) |
| `--list` | Lista proyectos escaneables |
| `--restore --list` | Lista backups disponibles |
| `--remediate /ruta` | Ciclo completo de remediation |
| `--remediate --dry-run /ruta` | Preview de hallazgos |
| `--list` (remediation) | Lista remedies generados |

## 8. Problemas Comunes

| Problema | Solucion |
|----------|----------|
| `rsync no esta instalado` | Responder `y` para instalarlo via apt |
| `Permission denied` al generar | Usar `sudo` o asegurar que /opt/ es escribible |
| Harness ya existe y quiero regenerar | Usar `--force` o elegir opcion 1 en el menu |
| boot.sh falla con "ejecutar como root" | Conectarse al VPS como usuario normal con sudo |
| boot.sh falla en integrity.sum | Harness modificado manualmente. Re-generar con --scan |

#End Development By Angel Esquivel (CyberSecurity) [NEXUS ONE 2026]
