# Changelog


## 2026-07-04
- Self-contained harness architecture: ollama binary → `models/bin/ollama` (standalone .tar.zst/tgz download, no `curl | sudo bash`), models → `models/ollama/` via `OLLAMA_MODELS`; opencode → descarga directa de GitHub (`github.com/anomalyco/opencode/releases/latest/download/opencode-linux-<arch>.tar.gz`) a `models/bin/opencode`
- `start_ollama_local()`: lanza `models/bin/ollama serve` via `nohup` como `ag-<proyecto>`, sin systemd, espera respuesta hasta 30s
- `activate_agent()`: `PATH` incluye `$HARNESS_DIR/models/bin`, `OLLAMA_MODELS="$HARNESS_DIR/models/ollama/"` pasado al agente
- `setup_agent_user()`: `chown` condicional de `models/` si existe
- MAIN sequence: tras `install_runtime`, si es runtime ollama → `start_ollama_local` + `ollama pull $OLLAMA_MODEL` como agente
- API key vars (`CLAUDE_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`): usan `${VAR:-}` default expansion para evitar `unbound variable` con `set -u`
- `install_runtime()`: todas las operaciones de escritura usan `sudo` (mkdir, chmod) para compatibilidad con ownership mixto root/agent
- OpenCode self-contained: eliminada busqueda en sistema + symlink; reemplazado por descarga directa desde GitHub Releases a `models/bin/opencode` (mismo mecanismo que ollama). Tests actualizados.
- `docs/QUICKSTART.md`: expandida seccion 4 (deploy SCP/API/boot.sh/--runtime/verify) + nueva seccion 5 (usar agente, /help, /cost, /exit, tools, logs) + renumeracion.
- `docs/REMOTE_TESTING.md`: cross-ref QUICKSTART; OpenCode/Ollama actualizados a self-contained; Validacion Self-Contained checklist (7 checks).
- `AGENTS.md` MemPalace Standard: reducido a 2 tools (add_drawer + kg_add), eliminado diary_write.
- Validación remota 07-04 (.190): 5/5 runtimes (python3, opencode, ollama-tiny, claude, gemini) — todos PASS

## 2026-07-02
- Token HMAC licensing engine (Modelo B): `generator_create_license()` genera `.license.token` con HMAC-SHA256, sistema de verificacion con `license-verify.sh/.service/.timer`, y `checkin.sh` para heartbeat
- `generator_create_renew_script()`: renovacion offline con token incrustado — un solo archivo `renew.sh` que actualiza binding directamente sin internet ni heartbeat
- `prod-agent.sh`: nuevos flags `--license <nombre>`, `--duration <dias>` (default 180), `--heartbeat <url>` para configurar licenciamiento por harness
- `heartbeat/heartbeat-check.sh`: verificacion remota de licencia via SSH con flag `-h <host>` o prompt interactivo — reemplaza `server.py` (deprecado). No requiere Python ni servidor heartbeat. Conecta al VPS, detecta el harness y ejecuta `license-verify.sh` via sudo.
- Boot.sh headless deployment: flag `--non-interactive` para despliegue automatizado; auto-seleccion de runtime en no-TTY (ollama-14b > ollama-7b > opencode > none); flag `--runtime <name>` para override explicito
- Generator robustness: token activation usa variable de timestamp correcta; Python binding heredoc con aislamiento de variables runtime; JSON payload engine reescrito para comunicacion API confiable; sudo detection automatica con NOPASSWD fallback; kernel detection ahora se ejecuta en runtime en vez de generacion
- `docs/REMOTE-RENEWAL.md`: procedimiento de renovacion via `renew.sh` (un solo archivo, sin copiar multiples archivos al VPS)
- `docs/VALIDATION-PIPELINE.md`: documento de validacion con 7 tests y resultados
- Validacion completa en VPS remoto: 7/7 PASS — generacion, boot.sh, timer, verificacion remota SSH, renovacion offline, deteccion de expiracion, fingerprint mismatch

## 2026-07-01
- Licensing dimension assigned: D3 — Source Available (BSL + Commercial)
- D3.generator: subtype generator, each artifact inherits D3.child
- generator.sh: inyecta LICENSE.D3.child + AUTHORS + checkin.sh por harness
- prod-agent.sh: flag --license <client> para etiquetar harnesses
- Added scripts/license-fingerprint.sh + license-revoke.sh
- Added .gitignore excluye .fingerprint.json, .license.json, /opt/ag-*/
- Initial project scaffold for nexus-one

# CHANGELOG.md -- Historial de Versiones del Proyecto
# =================================================================
# Registro cronologico de cambios, mejoras y correcciones por version.
# =================================================================

## [1.0.0] - 2026-06-01
### Stage 1: Fundacion (v0.1.0 - v0.3.0)
- Estructura modular del proyecto: prod-agent.sh entry point, modules/ sourceables
- scanner.sh: deteccion de lenguaje, framework, entry point, dependencias, features
- generator.sh: creacion completa de /opt/ag-<proyecto>/ con heredocs inline
- templates/agent.sh y detect.sh: plantillas del harness y fingerprinting VPS
- Headers/footers de desarrollo NEXUS ONE en todos los archivos

### Stage 2: Seguridad y Validación (v0.4.0 - v0.6.0)
- security.sh: 6 checks post-generación (limits, permissions, secrets, gitignore, env, integrity)
- validator.sh: 7 checks estructurales del harness
- templates/agent.sh: loop ReAct con 10 mecanismos de seguridad runtime
- boot.sh: evaluate_resources() + menu runtime dinamico
- detect_resources(): tolerante a fallos (|| true)
- docs: TECHNICAL.md, COMPENDIO.md, CHANGELOG.md, reglas de documentación

### Stage 3: Expansion con Remediation (v0.7.0 - v0.10.0)
- Subsistema remediation: remediation.sh, 5 modulos, 8 tools, template agente
- REPL reescrito a conversation loop LLM-nativo (213 -> 831 lines)
- 4 proveedores LLM: Ollama, Claude, Gemini, OpenCode
- 9 tools via [TOOL:nombre], recovery mode, historial JSON
- Cost tracking: tokens input/output + costo USD
- Slash commands: /debug, /cost, /compact, /status, /help
- Context compaction: sliding window y summarize via LLM
- Memory system: [TOOL:remember] y [TOOL:recall] persistente

### Stage 4: Maduracion Producción (v0.11.0 - v0.13.0)
- PIDS_LIMIT enforcement via ulimit -u
- BUDGET_USD enforcement con cierre automático
- BATS test suite: test_backup, test_scanner, test_validator
- Plugin tools: load_tools() via .agent/tools/*.sh
- Agent expertise: .agent/expertise/*.md en system prompt
- Daemon mode: daemon_loop() con tasks JSON + results
- Model router: router_classify() heuristico (cheap/medium/high)
- --with-remediation: copia 4 tools no-LLM al harness
- Interpreter bypass: validate_command() bloquea -c/-e en python3/node/perl
- systemd sandbox: IPAddressDeny + CPUQuota + MemoryMax + journald
- check_kernel() en boot.sh: advise kernel updates

### Stage 5: Multi-Proveedor e Integración (v0.14.0 - v0.16.0)
- OpenAI provider: llm_call_openai() con gpt-4o-mini/4o/o3-mini
- Uninstall autogenerado: cada harness con uninstall.sh (6 pasos)
- tests/custom-2026/: Security Audit Toolkit de prueba
- templates/agent.sh: opencode --prompt obsoleto -> opencode run
- Payload quoting: ''' -> PYTHON_* env vars (Ollama, Claude, OpenAI, Gemini)
- stream:false -> stream:False (Python capitalizacion)
- AGENTS.md: fuente de verdad unificada (fusion prod-agent + remediation)
- README.md: Two-Tier Overview con diagrama de flujo
- REMOTE_TESTING.md: guia de pruebas LLM remoto
- 40 tests remotos, 40 PASS, 0 FAIL

### Stage 6: Documentación y UX (v0.17.0 - v0.18.0)
- --dry-run en prod-agent.sh y remediation.sh: preview sin efectos secundarios
- READMEs reestructurados sin duplicacion: root (general), prod-agent/docs (específico), remediation/docs (específico)
- docs/QUICKSTART.md: guia practica paso a paso para principiantes
- docs/CONCEPT.md: concepto de agente orquestador con ejemplos DSToken (163 lines, 6 secciones)
- ROADMAPs actualizados con items completados
- Referencias cruzadas entre todos los documentos

## [0.18.0] - 2026-06-01
### Added
- docs/CONCEPT.md: concepto de agente orquestador con ejemplos DSToken (163 lines, 6 secciones)
- README.md: referencia a CONCEPT.md en tabla de documentos
- AGENTS.md: referencia a CONCEPT.md en tabla de documentos cruzados

## [0.17.0] - 2026-05-30
### Added
- `--dry-run` flag en prod-agent.sh: muestra preview del harness sin escribir archivos
- `--dry-run` flag en remediation.sh: muestra hallazgos sin snapshot ni cambios
- READMEs: documentación de --dry-run en todos los niveles
- AGENTS.md: comandos dry-run en secciones 3.13 y 4.7
- docs/QUICKSTART.md: guia practica paso a paso para principiantes
- ROADMAPs: --dry-run marcado como COMPLETADO en ambos subsistemas
- Pruebas: prod-agent dry-run con /opt/npm-2026, remediation dry-run con /opt/dstoken2026

### Changed
- READMEs reestructurados: root (general) + prod-agent/docs/ (específico) + remediation/docs/ (específico)
- Sin duplicacion entre READMEs: cada uno con proposito unico
- ROADMAPs: todos los items de Fase 1 marcados COMPLETADO

## [0.16.0] - 2026-05-27
### Added
- README.md: Two-Tier Overview con diagrama de flujo (generador vs harness)
- README.md: "Zero to Agent" sección para nuevos usuarios
- docs/AGENTS.md: fuente de verdad unificada (fusion prod-agent + remediation)
- Prueba OpenCode y Ollama remota validada en VPS
- REMOTE_TESTING.md: secciones OpenCode y Ollama con instalación

### Improved
- templates/agent.sh: `opencode --prompt -` obsoleto -> usar `opencode run`
- templates/agent.sh: payload quoting con ''' -> PYTHON_* env vars (Ollama, Claude, OpenAI, Gemini)
- templates/agent.sh: `stream:false` -> `stream:False` (Python capitalizacion)

### Verified
- OpenCode v1.15.11, Ollama v0.24.0 (llama3.2:1b): conversation loop OK
- 40 tests, 40 PASS, 0 FAIL, 0 pendientes

## [0.15.0] - 2026-05-26
### Added
- `--propagate-uninstall` flag en prod-agent.sh
- install_service() parchea path systemd con sed en boot-time
- docs/REMOTE_TESTING.md: guia de pruebas LLM remoto

### Changed
- tests/custom-2026: todos los scripts usan rutas relativas (SCRIPT_DIR)

### Improved
- Systemd path hardcodeado: parcheado con sed en install_service()
- IPAddressAllow: 0.0.0.0/0 en vez de dominios (systemd no resuelve DNS)
- Recovery mode: case-insensitive, skip empty, soporta /exit
- Plugin tools: bash en vez de source para evitar main() execution

### Verified
- OpenAI: conectividad, auth, chat, conversation loop, seguridad API key, ~$0.00001
- Validación 7 fases (A-G): 36 tests, 36 PASS, 0 FAIL, 0 pendientes

## [0.14.0] - 2026-05-25
### Added
- OpenAI provider: llm_call_openai() con gpt-4o-mini/4o/o3-mini
- Uninstall autogenerado: cada harness incluye uninstall.sh (6 pasos)
- tests/custom-2026/: Security Audit Toolkit de prueba

### Changed
- templates/agent.sh: nuevo case openai en llm_call()
- generator.sh: llama a generator_create_uninstall_sh()

## [0.13.0] - 2026-05-25
### Added
- Plugin tools: load_tools() via .agent/tools/*.sh
- Agent expertise: .agent/expertise/*.md en system prompt
- Daemon mode: daemon_loop() con tasks JSON + results
- Model router: router_classify() heuristico (cheap/medium/high)
- `--with-remediation`: copia 4 tools no-LLM al harness
- `--watch` mode en remediation.sh

### Security
- Interpreter bypass: validate_command() bloquea -c/-e en python3/node/perl
- systemd sandbox: IPAddressDeny + CPUQuota + MemoryMax + journald
- check_kernel() en boot.sh: advise actualizaciones de kernel

## [0.11.0] - 2026-05-21
### Added
- PIDS_LIMIT enforcement via ulimit -u
- BUDGET_USD enforcement con cierre automático
- BATS test suite: test_backup, test_scanner, test_validator
- Regla PascalCase en remediation/AGENTS.md

## [0.10.0] - 2026-05-21
- Cost tracking: tokens input/output + costo USD
- Slash commands: /debug, /cost, /compact, /status, /help
- Context compaction: sliding window y summarize via LLM
- Memory system: [TOOL:remember] y [TOOL:recall] persistente

## [0.9.0] - 2026-05-21
### Added
- backup_create/list/restore en backup.sh
- CLI --restore <proyecto> y --restore --list
- Backup automático en generator_generate() antes de rsync

## [0.8.0] - 2026-05-20
### Changed
- REPL reescrito a conversation loop LLM-nativo (213 -> 831 lines)
  - 4 providers: Ollama, Claude, Gemini, OpenCode
  - 9 tools via [TOOL:nombre], recovery mode, historial JSON
- boot.sh: evaluate_resources() antes de menu runtime
- detect_resources(): tolerante a fallos (|| true)

## [0.7.0] - 2026-05-20
### Added
- Subsistema remediation: remediation.sh, 5 modulos, 8 tools
- remediation/context/AGENTS.md y ROADMAP.md

## [0.6.0] - 2026-05-20
### Added
- docs/TECHNICAL.md, COMPENDIO.md, CHANGELOG.md
- Reglas de documentación y referencias a AGENTS.md

### Improved
- Documentos derivados ahora validan contra AGENTS.md

## [0.5.0] - 2026-05-18
### Added
- Menu de re-escaneo: 4 opciones (Sobrescribir, Update, Backup, Cancelar)
- Flags --force y --update
- integrity.sum: SHA256 de 9 archivos del harness

### Changed
- sudo NOPASSWD -> sudo -v (cacheo de password)
- Harness: /opt/nexus-one/ag-* -> /opt/ag-*

## [0.4.0] - 2026-05-15
### Added
- security.sh: 6 checks (limits, permissions, secrets, gitignore, env, integrity)
- validator.sh: 7 checks estructurales
- templates/agent.sh: loop ReAct con 8 mecanismos seguridad
- templates/detect.sh: fingerprinting VPS

## [0.3.0] - 2026-05-12
### Added
- generator.sh: creacion completa de harness (limits.conf, agent.md, boot.sh, context)
- Estructura /opt/ag-<proyecto>/ con exclusiones rsync

## [0.2.0] - 2026-05-10
### Added
- scanner.sh: deteccion de lenguaje, framework, entry point, deps, features
- 9 variables exportadas

## [0.1.0] - 2026-05-08
### Added
- Estructura inicial del proyecto /opt/nexus-one/
- prod-agent.sh: entry point CLI con --scan, --validate, --list
- Arquitectura modular con modules/ sourceables
- Headers/footers de desarrollo NEXUS ONE

#End Development By Angel Esquivel (CyberSecurity) [NEXUS ONE 2026]
