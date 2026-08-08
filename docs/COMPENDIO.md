# Compendio: Framework NEXUS ONE – Prod-Agent + Remediation
# =================================================================
# Guia General del Proceso: Desde el Problema Inicial Hasta la
# Optimización de Proyectos via LLM.
#
# Documentación AUTORITATIVA (fuentes de verdad):
#   docs/AGENTS.md     -- arquitectura, flujo, reglas, seguridad
#   prod-agent/context/ROADMAP.md    -- fases, entregables, prioridades
#   docs/AGENTS.md    -- arquitectura NEXUS ONE
#   remediation/context/ROADMAP.md   -- road map NEXUS ONE
#
# Este documento valida y referencia a todos. No duplica información
# que ya existe en los context files; los expande con narrativa.
# =================================================================

## 1. El Problema

### 1.1 Pregunta Inicial

?Como convierto un proyecto existente en un agente LLM autonomo que
pueda desplegar en cualquier VPS con un solo comando?

### 1.2 Proyecto de Origen

**npm-2026** en /opt/npm-2026/ (69 archivos, 11,155 líneas, entry point bootstrap.sh).

### 1.3 Restricciones del Problema

| Restriccion | Implicacion |
|-------------|-------------|
| El proyecto original NO se modifica | El harness debe ser un directorio separado |
| El VPS de destino es Ubuntu 24.04 limpio | boot.sh debe instalar todo lo necesario |
| El VPS puede NO tener internet | Runtimes LLM locales (Ollama) como opcion primaria |
| El agente debe ser seguro aunque su prompt sea malicioso | Seguridad por restriccion OS, no por prompt engineering |
| No se pueden instalar dependencias pesadas en dev | prod-agent escrito en Bash puro |

### 1.4 Stack Tecnologico

| Componente | Eleccion | Justificacion |
|------------|----------|---------------|
| Generador | Bash puro | Cero dependencias, portabilidad |
| Modular | 4 modulos sourceados | Separacion de responsabilidades |
| Templates | Bash scripts | Copia literal o heredoc con sustitucion |
| Runtimes LLM | Ollama (local), Claude, OpenCode, Gemini (cloud) | Cobertura online/offline |
| Seguridad | Aislamiento de usuario + root:root + integrity.sum | Defensa en profundidad OS |
| Despliegue | boot.sh unico via SCP | Zero config en VPS |

### 1.5 Evolucion del Nombre

```
npm-agent  -->  prod-agent
(solo npm-2026)     (cualquier proyecto)
```

## 2. Las Referencias Absorbidas

| Referencia | Concepto Clave | Implementación en NEXUS ONE |
|------------|----------------|-----------------------------|
| **OpenCode** (opencode.ai) | Skills, MCP, sub-agentes | AGENTS.md como fuente de verdad, estructura de skills |
| **NEXUS ONE Framework** | Framework completo, niveles de madurez | Base metodologica: provision/validate/execute |
| **PI Agent** (Indydev Dan) | Peer-to-peer entre agentes | Fase 5 del roadmap (multi-agente) |
| **Stripe Minions** (engineering.stripe.com) | Blueprint engine para agentes | prod-agent como generador de harnesses |
| **Hermes Agent** (hermes-agent.github.io) | Memoria persistente, auto-skills | Fase 3: checkpoints JSON, skill discovery |
| **OpenAgent** (the-open-agent) | Stack abierto de agente | Templates parametrizables |
| **SimpleAgent** (linkxzhou) | Minimalismo funcional | boot.sh como entry point unico |

## 3. Las Decisiones Arquitectonicas (ADRs)

| # | Decision | Alternativa Descartada | Consecuencia |
|---|----------|------------------------|--------------|
| 1 | Generar /opt/ag-<proyecto>/ como proyecto separado | Subdirectorio .agent/ en proyecto original | Harness autonomo, Git-ready |
| 2 | boot.sh es el unico entry point remoto | Multiples scripts, Makefile, docker-compose | Un solo comando SCP + ./boot.sh |
| 3 | Recursos del VPS se detectan en boot-time | Escanear en tiempo de generación | Harness se adapta al VPS real |
| 4 | Seguridad por restriccion OS (no por prompt) | Confiar en que el LLM seguira instrucciones | Agente NO puede violar limites aunque intente |
| 5 | agent.md como YAML frontmatter + Markdown | Solo Markdown, solo YAML | Legible por humanos y maquinas |
| 6 | Model router jerarquico con fallback | Un solo LLM fijo | Resiliencia si un proveedor cae |
| 7 | Estado en memoria, checkpoint JSON opcional | Base de datos externa, Redis | Sin dependencias de infraestructura |
| 8 | prod-agent en Bash puro | Python, Go, Node | Zero dependencias, portabilidad total |
| 9 | Harness corre como usuario ag-<proyecto> sin sudo | Mismo usuario que SCP, contenedor Docker | Aislamiento real a nivel de SO |
| 10 | Re-escaneo con menu interactivo (4 opciones) + flags | Forzar siempre sobrescritura | Flexibilidad: update, backup, cancel |
| 11 | Menu LOCAL/CLOUD en boot.sh | Menu unificado sin distincion | Sin opciones cloud cuando no hay internet |
| 12 | sudo -v en boot.sh (NO requiere NOPASSWD) | Exigir NOPASSWD en sudoers | Password cacheado, menor riesgo |
| 13 | Templates: agent.sh y detect.sh se copian literales | Todos los archivos via heredoc | Consistencia entre generaciones |
| 14 | 4 modulos sourceados secuencialmente | Un solo archivo monolito | Separacion de responsabilidades |
| 15 | Bloquear `-c/-e` en interpretes en lugar de bloquear python/node completos | Prohibir python3, node, perl globalmente | Permite tests y scripts legitimos, bloquea ejecución inline |
| 16 | systemd como alternativa ligera a Docker | docker-compose multi-servicio | Menos dependencias, journald, sandbox de red |
| 17 | check_kernel advisory en boot.sh | Bloqueo estricto por kernel desactualizado | Mejora seguridad sin romper bootstrap |

## 4. La Arquitectura

Ver `docs/AGENTS.md` para la fuente de verdad arquitectonica completa.
Este compendio solo resume los puntos clave.

### 4.1 Mapa General

```
/opt/nexus-one/                        (maquina de desarrollo)
|
+-- prod-agent/                        (generador de harnesses)
|   +-- prod-agent.sh                  (entry point)
|   +-- modules/                       (scanner, generator, security, validator)
|   +-- templates/                     (agent.sh, detect.sh)
|   +-- context/                       (AGENTS.md, ROADMAP.md)
|
+-- remediation/                       (subsistema NEXUS ONE)
|   +-- remediation.sh                 (entry point --remediate)
|   +-- modules/                       (scanner, snapshot, verifier, remediator, reporter)
|   +-- tools/                         (8 herramientas de análisis y reparacion)
|   +-- templates/                     (remediate-agent.sh)
|   +-- context/                       (AGENTS.md, ROADMAP.md)
|   +-- remedy/                        (output: remedy-<proyecto>.json)
|   +-- snapshots/                     (backups pre-remediation)
|
+-- docs/                              (documentación global)
    +-- COMPENDIO.md, TECHNICAL.md, CHANGELOG.md

Un comando: ./prod-agent.sh --scan /opt/mi-proyecto
  genera: /opt/ag-mi-proyecto/        (listo para SCP + boot.sh)

Un comando: ./remediation.sh --remediate /opt/mi-proyecto
  genera: remedy/mi-proyecto.json     (hallazgos y cambios)
  genera: snapshots/*.tar.gz          (backup pre-cambio)
```

### 4.2 Capas del Framework vs Implementación

| Framework NEXUS ONE | Implementación prod-agent |
|---------------------|---------------------------|
| Provision | scanner.sh + generator.sh |
| Validate | security.sh + validator.sh |
| Execute | templates/agent.sh + templates/detect.sh |
| Monitor | agent.sh: audit.log (runtime) |
| Notify | Fase 4-5 del roadmap |

### 4.3 Niveles de Madurez

| Nivel | Descripción | Estado |
|-------|-------------|--------|
| 1. Scripting basico | boot.sh, detect.sh, limits.conf | COMPLETADO |
| 2. Agente reactivo | agent.sh + LLM + tools + context | COMPLETADO |
| 3. Multi-agente | prod-agent <-> ag-project (P2P) | ROADMAP Fase 5 |
| 4. Ecosistema | GitHub como registry de agentes | ROADMAP Fase 5 |
| 5. Meta-aprendizaje | Agentes que generan agentes | Vision futuro |

### 4.4 NEXUS ONE – Subsistema de Remediation

NEXUS ONE es el subsistema de optimización de proyectos dentro de NEXUS ONE.
Mientras prod-agent **genera** harnesses seguros, NEXUS ONE **remedia** el
proyecto original antes de generar el harness.

Ver `docs/AGENTS.md` para arquitectura completa.

## 5. El Generador: prod-agent

Ver `docs/AGENTS.md` secciones "Flujo de Ejecución" y "Comandos de Prueba"
para especificacion detallada de cada modulo (scanner, generator, security, validator).

### 5.1 CLI Completo

```bash
./prod-agent.sh --scan /ruta/al/proyecto    [generar harness]
./prod-agent.sh --scan --force /ruta        [sobrescribir sin preguntar]
./prod-agent.sh --scan --update /ruta       [solo config, preserva src/]
./prod-agent.sh --validate /ruta/al/harness [validar estructura]
./prod-agent.sh --list                      [listar proyectos en /opt/]
./prod-agent.sh --help                      [ayuda detallada]
./prod-agent.sh --version                   [version 1.0.0]
```

### 5.2 Flujo de 4 Modulos

```
Scanner --> [estructura, lenguaje, entry point, dependencias]
                   |
                   v
Generator --> [harness completo: .agent/ context/ src/ boot.sh detect.sh]
                   |
                   v
Security  --> [6 checks post-generación]
                   |
                   v
Validator --> [7 checks estructurales -> PASS/FAIL]
```

Cada modulo esta documentado en `docs/AGENTS.md` secciones "Flujo de Ejecución"
y "Mapa de Secuencia de 4 Modulos".

## 6. El Harness Generado

### 6.1 Estructura

Ver `docs/AGENTS.md` sección "Arbol de Archivos" para estructura completa
con permisos y ownership.

Resumen: `/opt/ag-<proyecto>/` es autocontenido con `.agent/` (config + agente), `boot.sh`
(entry point unico), `detect.sh` (fingerprinting), `context/` (metadata), `src/` (código).

### 6.2 Los 2 Templates

| Template | Uso | Proposito |
|----------|-----|-----------|
| templates/agent.sh | Copia literal a .agent/agent.sh | Conversation loop LLM-nativo + recovery mode + 9 tools + plugin system + daemon + router |
| templates/detect.sh | Copia literal a detect.sh | Fingerprinting RAM, disco, CPU, Docker, internet |

### 6.3 Archivos Generados (6 via heredoc)

| Archivo | Generador | Función en generator.sh |
|---------|-----------|------------------------|
| .agent/limits.conf | heredoc | generator_create_limits |
| .agent/agent.md | heredoc | generator_create_agent_md |
| boot.sh | heredoc | generator_create_boot_sh (9 funciones) |
| context/README.md | heredoc | generator_create_context_readme |
| context/SEGURIDAD.md | heredoc | generator_create_seguridad_md |
| integrity.sum | sha256sum | generator_create_integrity_sum |

### 6.4 Exclusiones de rsync

`.env`, `node_modules`, `.git`, `venv`, `__pycache__`, `build`, `dist`, `target`, `logs`, `*.log`

### 6.5 integrity.sum

11-12 archivos: `.agent/` (5: limits, agent.md, agent.sh, tools/hello.sh, expertise/project_rules.md), `boot.sh`, `detect.sh`, `.gitignore`, `context/` (3), `nexus-<proyecto>.service` opcional.
Excluye: `src/`, `.agent/audit.log`, `.agent/tasks/`, `.agent/results/`.

## 7. El Despliegue en VPS

### 7.1 Secuencia boot.sh

Ver `docs/AGENTS.md` sección "Flujo de Despliegue en VPS" para diagrama detallado.

Resumen de 7 pasos:

```
1. CHECK_SECURITY    -> EUID=0? exit 1, sudo -v, sha256sum -c
2. SETUP_AGENT_USER  -> useradd -m -s /usr/sbin/nologin ag-<proyecto>
                         chown root:root .agent/ boot.sh detect.sh integrity.sum
3. DETECT_RESOURCES  -> sudo -u ag-* ./detect.sh (tolerante a fallos || true)
                         RAM, disco, CPU, Docker, internet, arquitectura
4. EVALUATE_RESOURCES -> decide si hay opciones viables:
                           RAM < 2GB -> RUNTIME=none forzado (recovery)
                           RAM >= 8GB -> viable (Ollama local)
                           Internet + RAM >= 2GB -> viable (cloud)
                           Sin opciones -> RUNTIME=none forzado
5. SELECT_RUNTIME    -> Menu filtrado segun recursos disponibles
                          1) Ollama 7b  2) Ollama 14b  3) Claude
                          4) OpenCode   5) Gemini     r) Recovery
6. INSTALL_RUNTIME   -> curl install.sh | sudo bash, export API key, o skip si RUNTIME=none
7. ACTIVATE_AGENT    -> exec sudo -u ag-<proyecto> env RUNTIME=... OLLAMA_MODEL=... KEY=val .agent/agent.sh
```

### 7.2 Arquitectura de Usuarios en VPS

| Usuario | Rol | SSH | Sudo |
|---------|-----|-----|------|
| root | No se usa | Deshabilitado | N/A |
| (tu usuario) | Bootstrap | Si | Si (con password) |
| ag-<proyecto> | Agente runtime | No | No |

### 7.3 Manejo de API Keys

API keys NUNCA se escriben a disco dentro del harness.
Se capturan via read -sp en boot.sh y se pasan como variables de entorno:

- Claude: export ANTHROPIC_API_KEY
- Gemini: export GEMINI_API_KEY
- OpenCode: curl install.sh (no requiere key)
- Ollama: curl install.sh + ollama pull (no requiere key)

## 8. La Seguridad

Ver `docs/AGENTS.md` sección "Modelo de Seguridad" para la fuente de verdad completa
(3 capas, 10 mecanismos runtime, matriz de riesgos, gaps identificados).

### 8.1 Principio Fundamental

Seguridad por restriccion OS, NO por prompt engineering.
El agente NO puede violar sus limites aunque su prompt sea manipulado.
La sancion viene del kernel y filesystem.

### 8.2 Las 3 Capas (Resumen)

| Capa | Mecanismo |
|------|-----------|
| 1. Boot | Rechazar root, sudo -v, sha256sum |
| 2. Usuario | ag-<proyecto> sin sudo, sin shell |
| 3. Archivos | config root:root, workspace ag-* |

### 8.3 Gaps de Seguridad Identificados

Ver `docs/AGENTS.md` sección "Modelo de Seguridad" para matriz completa y gaps documentados.

## 9. Limites de Seguridad (limits.conf)

Ver `docs/AGENTS.md` sección "Archivo limits.conf" para las 8 secciones INI y valores default.

## 10. Backup de Proyectos

### 10.1 Sistema de Backups

Cada proyecto escaneado con `--scan` genera automaticamente un backup en `/opt/nexus-one/backups/`.
El backup se crea antes de copiar el código fuente al harness, sin intervencion del usuario.

### 10.2 Estructura por Proyecto

```
/opt/nexus-one/backups/
  +-- npm-2026.tar.gz          # Archivo comprimido del proyecto original
  +-- npm-2026.sha256          # Checksum SHA256 del tar.gz
  +-- npm-2026.meta.json       # Metadata: fecha, harness, tamaño, sha256
```

### 10.3 Backup Automático en --scan

```
generator_generate()
  ...
  backup_create()  -> /opt/nexus-one/backups/<proyecto>.tar.gz + .sha256 + .meta.json
  rsync src/       -> copia el proyecto al harness
  ...
```

El backup ocurre exactamente antes del rsync a `src/`. Un solo backup por proyecto
(se sobrescribe en el proximo `--scan` del mismo proyecto).

### 10.4 Comandos de Backup

```bash
# Listar backups disponibles
./prod-agent.sh --restore --list

# Restaurar un proyecto desde backup
./prod-agent.sh --restore npm-2026
```

### 10.5 Validación de Backups

- Cada backup tiene su checksum SHA256 (`.sha256`)
- `backup_restore()` verifica el checksum antes de extraer
- Si el checksum no coincide, la restauracion se rechaza (archivo corrupto)

## 11. Nuevas Capacidades del Agente (v0.10.0)

### 11.1 Debug Mode

Toggle en runtime via `/debug on|off` en el conversation loop.
Cuando esta activo, muestra en cada llamada LLM:

- Payload JSON completo enviado al proveedor
- Respuesta JSON completa del proveedor
- Conteo de tokens de entrada y salida
- Costo estimado (solo para Claude y Gemini)

### 11.2 Cost Tracking

Cada llamada a Claude o Gemini extrae tokens de la respuesta
y calcula el costo segun tarifas del modelo. OpenCode es gratuito
(sin tracking). Visualizacion via `/cost`.

```bash
nexus> /cost
=========================================================================
  Costo de Sesion
=========================================================================
  Tokens Input:  4500
  Tokens Output: 1200
  Costo Total:   $0.0435 USD
=========================================================================
```

### 11.3 Slash Commands

| Comando | Descripción |
|---------|-------------|
| `/debug on` | Activa modo debug (JSON payloads) |
| `/debug off` | Desactiva modo debug |
| `/cost` | Muestra tokens y costo |
| `/compact sliding N` | Ventana de N mensajes |
| `/compact summarize` | Resumen via LLM |
| `/compact none` | Desactiva compactacion |
| `/status` | Estado del sistema |
| `/help` | Lista de comandos |

### 11.4 Context Compaction

Dos estrategias para manejar contexto largo:

- **Sliding window**: Conserva los ultimos N mensajes (default 20).
  Se aplica automaticamente si la estrategia esta activa.

- **Summarize**: Envia el historial al LLM con un prompt de resumen
  y reemplaza el contexto con el resumen generado.

Estrategia default: `none` (sin compactacion).

### 11.5 Memory System

Dos nuevas herramientas accesibles para el LLM:

| Tool | Descripción |
|------|-------------|
| `[TOOL:remember] tags:x,y texto` | Guarda información importante en .agent/memory/ |
| `[TOOL:recall] query` | Busca en memoria por tags/palabras clave |

Caracteristicas:
- Almacenamiento en archivos JSON por recuerdo (mem_NNNNNN.json)
- Indice central en index.json para busqueda rápida
- Persiste entre sesiones (no se pierde al cerrar)
- El LLM decide que es importante recordar

## 12. Nuevas Capacidades del Harness (v0.13.0)

### 12.1 Plugin System de Tools

El template `agent.sh` ahora soporta tools plugin via convencion:

- Cada `.sh` en `.agent/tools/` exporta una variable `TOOLNAME_DESCRIPTION` y una función `toolname_run()`.
- `load_tools()` las sourcea en startup y registra en `TOOL_REGISTRY`.
- `build_system_prompt()` las lista bajo header `TOOLS PLUGIN:`.
- `execute_tool()` chequea plugin tools antes de las built-in.

Tool default: `hello.sh` (demo). Con `--with-remediation`, se copian 4 tools de remediation:
`scan_secrets.sh`, `audit_deps.sh`, `verify_process.sh`, `extract_secrets.sh`.

### 12.2 Daemon Mode

El agente puede operar como daemon via `agent.sh --daemon`:

- Lee tasks de `.agent/tasks/*.task` (formato JSON: `{"prompt": "...", "priority": N}`).
- Procesa cada task via LLM y escribe resultado en `.agent/results/<name>.result`.
- Mueve tasks completados a `.agent/tasks/done/`.
- Loop cada 30s con `check_session_timeout`.

### 12.3 Agent Expertise

Reglas de conducta persistentes en `.agent/expertise/*.md`:

- Se cargan alfabeticamente en `build_system_prompt()`.
- Cada archivo se presenta como `=== filename ===` + contenido.
- Solo lectura por el agente (nunca los modifica).

Archivo default: `project_rules.md` (seguridad, calidad, operaciones).

### 12.4 Model Router

Clasificador heuristico que selecciona variante de modelo segun complejidad:

- `router_classify()` analiza longitud del prompt + keywords (security, architecture, refactor, etc.).
- Niveles: `cheap` (<100 chars), `medium` (100-2000), `high` (>2000 o keyword match).
- Activado via `MODEL_ROUTER=true` en `limits.conf [ROUTER]`.
- Por provider: Claude elige entre sonnet/opus, Gemini entre flash/pro.
- Sin llamada LLM para clasificacion (puramente heuristico).

### 12.5 Remediation --watch Mode

El subsistema de remediation ahora soporta monitoreo continuo:

```bash
./remediation.sh --watch /opt/proyecto [intervalo_segundos]
```

- Ejecuta ciclo completo de remediation en loop perpetuo.
- Intervalo default: 300 segundos (5 minutos).
- Cada ciclo genera nuevo reporte y snapshot.

### 12.6 Comandos Actualizados

```bash
# prod-agent con remediation tools
./prod-agent.sh --scan --with-remediation /opt/proyecto

# prod-agent combinado
./prod-agent.sh --scan --force --systemd --with-remediation /opt/proyecto

# remediation watch mode
./remediation.sh --watch /opt/proyecto 60
```

## 13. Templates en Detalle (v0.13.0)

### 13.1 templates/agent.sh (1280 lines)

Features: conversation loop LLM-nativo con 4 providers (Ollama, Claude, Gemini, OpenCode),
9 tools built-in + plugin system via .agent/tools/, daemon mode via --daemon,
agent expertise via .agent/expertise/, model router heuristico.

### 13.2 templates/detect.sh (99 lines)

## 14. Re-escaneo y Mantenimiento

### 14.1 Menu de Re-escaneo

### 14.2 Ciclo de Vida del Harness

## 15. Prohibiciones y Sanciones

| Prohibicion | Sancion |
|-------------|---------|
| Duplicar información de AGENTS.md en docs Nivel 2 | Revertir commit, reescribir sección como referencia |
| Emojis o caracteres no ASCII | Reemplazar por equivalente ASCII |
| Sección desactualizada (contradice AGENTS.md) | Marcar como OBSOLETO hasta actualizar |
| Header/footer faltante en archivo nuevo | No aprobar PR hasta agregarlo |

## Apendice A: Archivos del Proyecto

| Archivo | Rol |
|---------|-----|
| /opt/nexus-one/prod-agent/prod-agent.sh | Entry point CLI |
| /opt/nexus-one/prod-agent/modules/scanner.sh | Análisis de proyectos |
| /opt/nexus-one/prod-agent/modules/generator.sh | Creacion de harnesses + reescaneo |
| /opt/nexus-one/prod-agent/modules/security.sh | Escaneo de seguridad |
| /opt/nexus-one/prod-agent/modules/validator.sh | Validación de estructura |
| /opt/nexus-one/prod-agent/templates/agent.sh | Plantilla de loop LLM + plugin + daemon + router + OpenAI |
| /opt/nexus-one/prod-agent/templates/detect.sh | Plantilla de deteccion recursos |
| /opt/nexus-one/docs/AGENTS.md | Fuente de verdad arquitectonica |
| /opt/nexus-one/prod-agent/context/ROADMAP.md | Roadmap de desarrollo |
| /opt/nexus-one/prod-agent/docs/README.md | README del subproyecto |
| /opt/nexus-one/README.md | README raíz (GitHub) |
| /opt/nexus-one/docs/CHANGELOG.md | Historial de versiones |
| /opt/nexus-one/docs/COMPENDIO.md | Este documento |
| /opt/nexus-one/docs/TECHNICAL.md | Documentación técnica |
| /opt/nexus-one/docs/PENDINGS.md | Fallas pendientes de resolver |
| /opt/nexus-one/tests/cleanup.sh | Script generico de desinstalacion remota |
| /opt/nexus-one/tests/custom-2026/bootstrap.sh | Proyecto test Security Audit Toolkit |
| /opt/nexus-one/tests/custom-2026/scripts/audit_users.sh | Auditoria de usuarios |
| /opt/nexus-one/tests/custom-2026/scripts/check_ports.sh | Escaneo de puertos abiertos |
| /opt/nexus-one/tests/custom-2026/scripts/check_auth_log.sh | Deteccion de fallos de autenticacion |
| /opt/nexus-one/tests/custom-2026/scripts/check_disk.sh | Verificación de particiones |
| /opt/nexus-one/tests/custom-2026/scripts/report_status.sh | Reporte JSON unificado |
| /opt/nexus-one/tests/custom-2026/config/thresholds.conf | Umbrales de auditoria |
| /opt/nexus-one/tests/custom-2026/config/services_to_monitor.conf | Puertos esperados |
| /opt/nexus-one/docs/COMPENDIO.md | Este documento |
| /opt/nexus-one/docs/TECHNICAL.md | Documentación técnica |
| /opt/nexus-one/remediation/remediation.sh | Entry point CLI de NEXUS ONE |
| /opt/nexus-one/remediation/modules/scanner.sh | Escaneo profundo de proyectos |
| /opt/nexus-one/remediation/modules/snapshot.sh | Backup de proyectos |
| /opt/nexus-one/remediation/modules/verifier.sh | Descubrimiento de procesos |
| /opt/nexus-one/remediation/modules/remediator.sh | Orquestador de cambios |
| /opt/nexus-one/remediation/modules/reporter.sh | Generador de reportes JSON |
| /opt/nexus-one/remediation/tools/scan_secrets.sh | Deteccion de secretos (25 patrones) |
| /opt/nexus-one/remediation/tools/extract_secrets.sh | Migracion de claves a .env |
| /opt/nexus-one/remediation/tools/audit_deps.sh | Auditoria de dependencias |
| /opt/nexus-one/remediation/tools/verify_process.sh | Ejecución de procesos |
| /opt/nexus-one/remediation/tools/gen_docs.sh | Generación de documentación via LLM |
| /opt/nexus-one/remediation/tools/refactor_plan.sh | Plan de refactor via LLM |
| /opt/nexus-one/remediation/tools/add_tests.sh | Generación de test skeletons |
| /opt/nexus-one/remediation/tools/add_lint.sh | Configuración de linters |
| /opt/nexus-one/remediation/templates/remediate-agent.sh | Template del agente de remediation |
| /opt/nexus-one/docs/AGENTS.md | Fuente de verdad NEXUS ONE |
| /opt/nexus-one/remediation/context/ROADMAP.md | Roadmap NEXUS ONE |
| /opt/nexus-one/remediation/docs/README.md | README del subsistema |

## Apendice B: Comandos Rapidos

### Generación (prod-agent)

```bash
cd /opt/nexus-one/prod-agent

./prod-agent.sh --scan /opt/mi-proyecto            # Generar harness
./prod-agent.sh --scan --force /opt/proyecto        # Sobrescribir sin preguntar
./prod-agent.sh --scan --update /opt/proyecto       # Solo config, preserva src/
./prod-agent.sh --scan --systemd /opt/proyecto      # Con servicio systemd
./prod-agent.sh --scan --with-remediation /opt/proyecto  # Con tools remediation
./prod-agent.sh --validate /opt/ag-proyecto         # Validar harness existente
./prod-agent.sh --list                               # Listar proyectos en /opt/
./prod-agent.sh --help                               # Ayuda
./prod-agent.sh --version                            # Version

# Desplegar en VPS
scp -r /opt/ag-mi-proyecto user@vps:~/
ssh user@vps './ag-mi-proyecto/boot.sh'
```

### Remediation (NEXUS ONE)

```bash
cd /opt/nexus-one/remediation

./remediation.sh --remediate /opt/proyecto          # Analizar y reparar proyecto
./remediation.sh --watch /opt/proyecto [intervalo]  # Modo vigilancia continua
./remediation.sh --list                               # Listar remedies generados
./remediation.sh --remedy nombre                     # Ver detalle de un remedy
./remediation.sh --help                               # Ayuda
./remediation.sh --version                            # Version

# Tools standalone (sin entry point)
bash tools/scan_secrets.sh /opt/proyecto
bash tools/audit_deps.sh /opt/proyecto
bash tools/gen_docs.sh /opt/proyecto ollama
bash tools/verify_process.sh /opt/proyecto
```

### Validación de Scripts

```bash
# Sintaxis de todos los scripts
bash -n /opt/nexus-one/prod-agent/prod-agent.sh
bash -n /opt/nexus-one/prod-agent/templates/agent.sh
bash -n /opt/nexus-one/prod-agent/modules/*.sh
bash -n /opt/nexus-one/remediation/remediation.sh
bash -n /opt/nexus-one/remediation/modules/*.sh

# Error checking (shellcheck si instalado)
shellcheck /opt/nexus-one/prod-agent/prod-agent.sh

# Probar generación con proyecto existente
/opt/nexus-one/prod-agent/prod-agent.sh --scan /opt/npm-2026

# Validar el harness generado
/opt/nexus-one/prod-agent/prod-agent.sh --validate /opt/ag-npm-2026
```

## Apendice B: Guia de Pruebas LLM en Remoto

### Referencia Completa
Ver `docs/REMOTE_TESTING.md` para procedimiento detallado.

### Resumen de Pasos
1. Verificar conectividad: `curl -I https://api.openai.com/v1/models`
2. Exportar API key: `export OPENAI_API_KEY="sk-proj-..."`
3. Test autenticacion: `curl -H "Authorization: Bearer $KEY" https://api.openai.com/v1/models`
4. Test chat: `curl -d '{"model":"gpt-4o-mini",...}' https://api.openai.com/v1/chat/completions`
5. Ejecutar agente: `echo "Hola" | timeout 30 bash .agent/agent.sh`
6. Verificar historial: `cat .agent/history.json`
7. Limpiar: `unset OPENAI_API_KEY`
8. Verificar seguridad: `grep -r "sk-proj-" harness/ | wc -l` (debe ser 0)

### Proveedores Validados
| Proveedor | Modelo | Estado | Costo Prueba |
|-----------|--------|--------|--------------|
| OpenAI | gpt-4o-mini | VALIDADO | ~$0.00001 |
| OpenAI | gpt-4o | Disponible | ~$0.00025 |
| OpenAI | o3-mini | Disponible | ~$0.00011 |
| OpenCode/Ollama | llama3.2:1b | Pendiente | $0 (local) |

### Resultados v0.16.0
- VPS: <vps-hostname> (<vps-ip>), Ubuntu 24.04, 7GB RAM, 8 cores
- Validación completa: 40 tests, 40 PASS, 0 FAIL, 0 pendientes
- OpenAI: conectividad, auth, chat completion, conversation loop (gpt-4o-mini)
- OpenCode: v1.15.11, conversation loop via `opencode run`, respuesta OK
- Ollama: v0.24.0, modelo llama3.2:1b, conversation loop local CPU-only, respuesta OK
- boot.sh: end-to-end setup (user, permissions, systemd)
- Systemd: start/stop/disable/logs (deploy en /opt/, sin CHDIR error)
- Uninstall.sh: 6 pasos, elimina todo correctamente
- Seguridad: restricted commands/paths bloqueados
- Recovery mode: case-insensitive, skip empty input, soporta /exit
- IPAddressAllow: fix aplicado (0.0.0.0/0 en vez de dominios)
- Costo total OpenAI: ~$0.00001 USD. OpenCode/Ollama: $0.

---

## Apendice C: Variables de Entorno

| Variable | Origen | Proposito |
|----------|--------|-----------|
| HARNESS_DIR | boot.sh | Directorio raíz del harness |
| AGENT_USER | boot.sh | Usuario ag-<proyecto> |
| RAM_TOTAL_GB | detect.sh | RAM disponible en VPS |
| HAS_INTERNET | detect.sh | Conectividad a internet |
| RUNTIME | boot.sh | Modo del agente (ollama, claude, gemini, opencode, none, auto) |
| OLLAMA_MODEL | select_runtime | Modelo Ollama (qwen2.5-coder:7b / 14b) |
| CLAUDE_API_KEY | Menu boot.sh | API key para Claude |
| GEMINI_API_KEY | Menu boot.sh | API key para Gemini |
| ANTHROPIC_API_KEY | boot.sh | Export para Claude SDK |

---

Documento actualizado v0.16.0.
Incluye validación completa 7 fases (A-G) + OpenCode + Ollama: 40 tests, 40 PASS, 0 FAIL.
Todos los issues conocidos resueltos (CHDIR, IPAddressAllow, Recovery mode, quoting payload).
Referencias a NEXUS ONE (secciones 4.1, 12, 15, Apendice A/B).
Valida contra: docs/AGENTS.md, prod-agent/context/ROADMAP.md,
docs/AGENTS.md, remediation/context/ROADMAP.md,
docs/REMOTE_TESTING.md.

#End Development By Angel Esquivel (CyberSecurity) [NEXUS ONE 2026]
