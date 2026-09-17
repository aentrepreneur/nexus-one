# TECHNICAL.md -- Documentación Técnica del Framework NEXUS ONE
# =================================================================
# Documentación técnica detallada del generador prod-agent y los
# harnesses generados. Expande y referencia a AGENTS.md (context/)
# como fuente de verdad arquitectonica.
#
# Validación contra: docs/AGENTS.md
# =================================================================

## 1. Arquitectura General

### 1.1 Mapa de Dependencias entre Modulos

```
prod-agent.sh (Entry Point CLI)
  |
  +-- source modules/scanner.sh
  |     No depende de otros modulos
  |     Exporta 9 variables globales
  |     Llama a generator_generate() en generator.sh
  |
  +-- source modules/generator.sh
  |     Consume variables exportadas por scanner.sh
  |     Copia templates/agent.sh -> .agent/agent.sh
  |     Copia templates/detect.sh -> detect.sh
  |     Llama a security_scan() en security.sh
  |
  +-- source modules/security.sh
  |     No depende de otros modulos
  |     Opera sobre el directorio del harness ya creado
  |
  +-- source modules/validator.sh
  |     No depende de otros modulos
  |     Opera sobre el directorio del harness ya creado
```

### 1.2 Arbol de Archivos Completo

```
/opt/nexus-one/
+-- README.md                          # Repo raíz (GitHub)
+-- docs/
|   +-- COMPENDIO.md                   # Compendio de sesion
|   +-- TECHNICAL.md                   # Este documento
+-- prod-agent/
    +-- prod-agent.sh                  # Entry point CLI (176 lines)
    +-- modules/
    |   +-- scanner.sh                 # Análisis de proyectos (267 lines)
    |   +-- generator.sh               # Creacion de harnesses (788 lines)
    |   +-- security.sh                # Escaneo post-generación (182 lines)
    |   +-- validator.sh               # Validación de integridad (177 lines)
    +-- templates/
    |   +-- agent.sh                   # Loop ReAct del agente (213 lines)
    |   +-- detect.sh                  # Fingerprinting VPS (99 lines)
    +-- context/
    |   +-- AGENTS.md                  # Fuente de verdad arquitectonica
    |   +-- ROADMAP.md                 # Fuente de verdad de fases
    +-- docs/
        +-- README.md                  # README del subproyecto

/opt/ag-<proyecto>/                     # Harness generado
    +-- .agent/
    |   +-- limits.conf                # 8 secciones INI
    |   +-- agent.md                   # YAML frontmatter + instrucciones
    |   +-- agent.sh                   # Copia de templates/agent.sh
    |   +-- audit.log                  # JSON lines (creado en runtime)
    +-- boot.sh                        # Generado inline (10 funciones)
    +-- detect.sh                      # Copia de templates/detect.sh
    +-- integrity.sum                  # SHA256 de 9-10 archivos
    +-- .gitignore
    +-- context/
    |   +-- README.md                  # Metadata del proyecto
    |   +-- SEGURIDAD.md               # Modelo de seguridad
    |   +-- estructura.txt             # Arbol del proyecto original
    +-- src/                           # Código fuente
```

## 2. Modulo Scanner (modules/scanner.sh)

### 2.1 Funciones

| Función | Línea | Descripción | Salida |
|---------|-------|-------------|--------|
| scanner_detect_language | 23 | Detecta lenguaje por archivos caracteristicos | PROJECT_LANG, PROJECT_PKG_MANAGER |
| scanner_detect_framework | 57 | Mapea lenguaje a framework conocido | PROJECT_FRAMEWORK |
| scanner_detect_entry_point | 100 | Heuristic search por entry point | PROJECT_ENTRY_POINT |
| scanner_count_files | 149 | Cuenta archivos y líneas de código | PROJECT_FILE_COUNT, PROJECT_LINE_COUNT |
| scanner_detect_features | 182 | Detecta Docker y .env.example | PROJECT_HAS_DOCKER, PROJECT_HAS_ENV_EXAMPLE |
| scanner_detect_dependencies | 190 | Lee requirements.txt o package.json | PROJECT_DEPS array |
| scanner_generate_tree | 213 | Genera arbol de directorios | estructura.txt en temp dir |
| scanner_scan | 223 | Orchestrador principal | Exporta 9 vars, llama a generator_generate() |

### 2.2 Lógica de Deteccion de Lenguaje (líneas 23-55)

```bash
scanner_detect_language() {
    # Prioridad: python > javascript > go > rust > bash
    # python: test -f main.py/setup.py/pyproject.toml/requirements.txt
    # go:     test -f go.mod
    # node:   test -f package.json
    # rust:   test -f Cargo.toml
    # bash:   ls *.sh >/dev/null 2>&1
    # default: unknown
}
```

### 2.3 Lógica de Deteccion de Entry Point (líneas 100-147)

```bash
scanner_detect_entry_point() {
    # python: busca main.py > app.py > manage.py > run.py
    # node:   jq -r '.main // "index.js"' package.json || node -e ...
    # go:     test -f main.go
    # rust:   test -f src/main.rs
    # bash:   ls *.sh | head -1
    # default: "<lang> entry point (no detectado)"
}
```

Fallbacks triples para JSON parsing: jq -> node -> grep+sed.

### 2.4 Salida del Scanner (9 variables exportadas)

```bash
PROJECT_NAME="npm-2026"
PROJECT_PATH="/opt/npm-2026"
PROJECT_LANG="bash"
PROJECT_FRAMEWORK="bash"
PROJECT_PKG_MANAGER=""
PROJECT_ENTRY_POINT="bootstrap.sh"
PROJECT_FILE_COUNT=69
PROJECT_LINE_COUNT=11155
PROJECT_HAS_DOCKER=true
PROJECT_HAS_ENV_EXAMPLE=false
PROJECT_DEPS=()  # array
```

### 2.5 Exclusiones en File Counting (líneas 151-159)

```bash
find "$project_path" -type f \
  -not -path "*/node_modules/*" \
  -not -path "*/venv/*" \
  -not -path "*/.git/*" \
  -not -path "*/__pycache__/*" \
  -not -path "*/build/*" \
  -not -path "*/dist/*" \
  -not -path "*/target/*"
```

## 3. Modulo Generator (modules/generator.sh)

### 3.1 Funciones

| Función | Línea | Descripción | Archivo Generado |
|---------|-------|-------------|------------------|
| generator_handle_existing | 13 | Menu 4 opciones si harness existe | N/A (lógica de decision) |
| generator_create_integrity_sum | 84 | sha256sum de 9-10 archivos | integrity.sum |
| generator_generate | 97 | Orchestrador principal | Todo el harness |
| generator_create_limits | 226 | Heredoc con 8 secciones INI | .agent/limits.conf |
| generator_create_agent_md | 278 | YAML frontmatter + instrucciones MD | .agent/agent.md |
| generator_create_boot_sh | 352 | Heredoc inline con 9 funciones | boot.sh |
| generator_create_context_readme | 635 | Metadata del proyecto escaneado | context/README.md |
| generator_create_seguridad_md | 684 | Modelo de seguridad detallado | context/SEGURIDAD.md |

### 3.2 Lógica de Re-escaneo (generator_handle_existing, líneas 13-82)

```
generator_handle_existing(harness_dir, project_name):
  1. Valida project_name contra regex ^[a-zA-Z0-9_.-]+$
  2. Si directorio NO existe: return 0
  3. Si FORCE_MODE=true: sudo rm -rf, return 0
  4. Si UPDATE_MODE=true:
     - rm -rf .agent/ context/ boot.sh detect.sh .gitignore integrity.sum
     - mkdir -p .agent/ context/
     - return 0 (conserva src/)
  5. Menu interactivo:
     1) Sobrescribir  -> rm -rf, return 0
     2) Actualizar    -> rm config, mkdir, return 0
     3) Backup+Nuevo  -> set NEW_BACKUP_DIR, return 0 (genera en .new/)
     4) Cancelar      -> exit 0
```

### 3.3 Flujo de Nuevo+Respaldo (Opcion 3, líneas 194-205)

```
1. NEW_BACKUP_DIR=/opt/ag-<proyecto>.new
2. rm -rf NEW_BACKUP_DIR (limpia residuos de ejecuciones previas)
3. Se sobreescribe harness_dir con .new
4. create_harness() se ejecuta con target=NEW_BACKUP_DIR
5. Si generación EXITOSA:
   a. rm -rf /opt/ag-<proyecto>.bak
   b. mv /opt/ag-<proyecto> -> /opt/ag-<proyecto>.bak
   c. mv /opt/ag-<proyecto>.new -> /opt/ag-<proyecto>
6. Si generación FALLA:
   a. ag-<proyecto>/ queda intacto
   b. .new/ se elimina
```

### 3.4 generator_create_integrity_sum (líneas 84-93)

```bash
generator_create_integrity_sum() {
    local harness_dir="$1"
    cd "$harness_dir"
    sha256sum .agent/limits.conf .agent/agent.md .agent/agent.sh \
              boot.sh detect.sh .gitignore context/README.md \
              context/SEGURIDAD.md context/estructura.txt > integrity.sum
    chmod 644 integrity.sum
}
```

Excluye deliberadamente: src/, .agent/audit.log, context/ (parcial).

### 3.5 Exclusiones de rsync (líneas 176-179)

```bash
rsync -a "$project_path/" "$harness_dir/src/" \
  --exclude=.env \
  --exclude=node_modules \
  --exclude=.git \
  --exclude=venv \
  --exclude=__pycache__ \
  --exclude=build \
  --exclude=dist \
  --exclude=target \
  --exclude=logs \
  --exclude='*.log'
```

### 3.6 Contenido Generado: limits.conf (8 secciones)

Ver `AGENTS.md` sección "Archivo limits.conf" para el contenido exacto.

### 3.7 Contenido Generado: agent.md (12 campos YAML)

```yaml
---
name: <proyecto>
version: 1.0.0
framework: NEXUS ONE
level: 2
language: <detectado>
framework_type: <detectado>
runtimes:
  primary: local
  fallback: claude
  emergency: gemini
budget:
  max_usd: 10
  mode: consumption
limits: .agent/limits.conf
skills:
  - scan, deploy, diagnose, monitor, update
security:
  model: user-isolation
  agent_user: <proyecto>
  config_owner: root
  enforced_by: limits.conf + kernel
cca_level: bronze
---
```

El contenido markdown posterior define instrucciones detalladas para el LLM.

### 3.8 Contenido Generado: boot.sh (Template Inline, 9 funciones)

El heredoc en generator_create_boot_sh() (líneas 356-686) genera un script con:

| Función | Línea en heredoc | Descripción |
|---------|------------------|-------------|
| print_banner | ~380 | ASCII art con nombre del proyecto |
| check_security | ~388 | EUID=0 check, sudo -v, sha256sum -c |
| setup_agent_user | ~421 | useradd, chown, chmod a archivos del harness |
| check_kernel | ~459 | Informa actualizaciones de kernel disponibles (apt list) |
| detect_resources | ~467 | Ejecuta detect.sh como ag-<proyecto> (tolerante a fallos) |
| evaluate_resources | ~483 | Evaluacion automática: RAM < 2GB o sin opciones -> RUNTIME=none |
| select_runtime | ~508 | Menu filtrado por recursos disponibles + opcion recovery |
| install_runtime | ~596 | curl install.sh, export KEYS, o skip si RUNTIME=none |
| load_context | ~639 | Lee context/README.md, estructura.txt, SEGURIDAD.md |
| activate_agent | ~653 | exec sudo -u ag-* env RUNTIME=... OLLAMA_MODEL=... .agent/agent.sh |

El boot.sh NO se copia de templates/ -- se genera inline en generator.sh.

## 4. Modulo Security (modules/security.sh)

### 4.1 Funciones

| Función | Línea | Descripción | Criterio PASS |
|---------|-------|-------------|---------------|
| security_scan | 14 | Orchestrador: 6 checks secuenciales | SECURITY_ISSUES==0 |
| security_check_limits | 45 | 7 campos requeridos en limits.conf | grep -q para cada campo |
| security_check_permissions | 68 | +x en scripts, sin world-writable fuera de src/ | find -perm -o+w -not -path "*/src/*" |
| security_check_secrets | 91 | detect-secrets o grep (8 patrones) | Sin matches |
| security_check_gitignore | 131 | 5 reglas: .env, *.log, venv/, __pycache__/, node_modules/ | grep -q para cada una |
| security_check_env | 151 | .env fuera de src/ | find -name ".env" -not -path "*/src/*" |
| security_check_integrity | 164 | sha256sum -c integrity.sum | Exit code 0 |

### 4.2 Patrones de Deteccion de Secrets (grep fallback, líneas 107-121)

Cuando detect-secrets no esta instalado, busca estos patrones en 8 tipos de archivo:

```bash
# Patrones:
API_KEY|api_key|api.secret|password|PASSWORD|secret|SECRET|token|TOKEN

# Tipos de archivo escaneados:
*.py, *.sh, *.js, *.ts, *.yaml, *.yml, *.json, *.cfg, *.conf, *.ini

# Directorios escaneados:
.agent/, boot.sh, detect.sh
```

### 4.3 Variables Globales

| Variable | Línea | Default | Tipo |
|----------|-------|---------|------|
| SECURITY_ISSUES | 11 | 0 | Contador de fallos |
| SECURITY_WARNINGS | 12 | () | Array de strings de warning |

## 5. Modulo Validator (modules/validator.sh)

### 5.1 Funciones

| Función | Línea | Descripción |
|---------|-------|-------------|
| validator_validate | 14 | Orchestrador: resuelve path, valida nombre, 6 checks |
| validator_check_structure | 77 | 3 directorios: .agent/ context/ src/ |
| validator_check_boot_sh | 95 | boot.sh existe, +x, shebang valido |
| validator_check_limits | 114 | 5 campos requeridos en limits.conf |
| validator_check_agent_md | 131 | agent.md existe, inicia con --- |
| validator_check_context | 145 | context/README.md existe, directorio no vacio |
| validator_check_src | 158 | src/ no vacio |
| validator_check_gitignore | 168 | .gitignore existe (recomendado) |

### 5.2 Criterios de Validación

| Check | PASS | FAIL |
|-------|------|------|
| Nombre | Empieza con ag- | No empieza con ag- |
| Directorios | .agent/ + context/ + src/ existen | Falta alguno |
| boot.sh | Existe + -x + shebang | No existe / no +x / no shebang |
| limits.conf | Existe + 5 campos | Falta campo |
| agent.md | Existe + comienza con --- | No existe / no YAML |
| context/ | README.md existe | Vacio |
| src/ | Tiene archivos | Vacio |
| .gitignore | Existe | No existe (warning) |

### 5.3 Codigos de Salida

| Código | Significado |
|--------|-------------|
| 0 | VALIDATOR_ERRORS es 0 (PASS) |
| 1 | VALIDATOR_ERRORS > 0 (FAIL) |

## 6. Templates

### 6.1 templates/agent.sh (1128 lines, conversation loop LLM-nativo)

Conversation loop con recovery mode. Arquitectura:

- **Conversation**: chat natural con LLM, 9 tools via `[TOOL:nombre] args`
- **Recovery mode**: sin LLM, 4 comandos (diagnose, retry, status, exit)
- **Seguridad**: 10 mecanismos runtime preservados del REPL original

```bash
# Core security (preservados del REPL original)
check_integrity()   # Verifica que limits.conf NO sea owned por agente
load_limits()       # Lee limits.conf con fallbacks a defaults
audit_log()         # JSON a audit.log
validate_command()  # Bloquea RESTRICT_COMMANDS
validate_path()     # Bloquea RESTRICTED_PATHS (prefix match)
check_session_timeout()  # Auto-exit tras MAX_SESSION_HOURS
exec_cmd()          # timeout + printf '%q' + audit

# LLM integration
detect_runtime()    # Auto-detects: ollama, claude, gemini, opencode, none
llm_call_ollama()   # POST /api/chat con qwen2.5-coder
llm_call_claude()   # POST /v1/messages con clave API
llm_call_gemini()   # POST /v1beta/models con clave API
llm_call_opencode() # pipe a opencode CLI

# Tools (11, llamadas via [TOOL:nombre] args)
execute_tool()      # Parser: "exec", "read_file", "search_code", "list_dir",
                    #          "write_file", "scan_secrets", "run_tests",
                    #          "status", "remember", "recall", "done"

# Conversation loop
build_system_prompt() # Desde agent.md + estructura.txt + limits
conversation_loop()   # while true: user -> LLM -> tools -> LLM -> response
                      # Soporta slash commands: /debug, /cost, /compact, /status, /help
recovery_mode()       # Modo offline sin LLM

# Debug mode
DEBUG_MODE=false      # Toggle via /debug on|off, muestra JSON payload/response

# Cost tracking
SESSION_COST=0.0      # USD acumulado por llamadas a proveedores de pago
TOKEN_INPUT=0         # Tokens de entrada acumulados
TOKEN_OUTPUT=0        # Tokens de salida acumulados

# Context compaction
COMPACT_STRATEGY="none"  # "none", "sliding", "summarize"
compact_sliding()     # Mantiene ultimos N mensajes (default 20)
compact_summarize()   # Usa LLM para resumir el historial

# Memory system
MEMORY_DIR="$HARNESS_DIR/.agent/memory/"
tool_remember()       # [TOOL:remember] tags:x,y texto -> guarda en JSON
tool_recall()         # [TOOL:recall] query -> busca en index.json

# History
load_history()      # Desde .agent/history.json
save_history()      # JSON a disco (max 50 mensajes)
```

**validate_command()** (línea 72): Ademas del filtro RESTRICT_COMMANDS, detecta y bloquea flags -c/-e en python3, python, node, perl. Esto previene interpreter bypass attacks tipo `python3 -c "evil_code"` sin bloquear scripts validos como `python3 script.py` o `npm test`.

**Default values (cuando limits.conf no existe o campo faltante):**

```bash
BUDGET_USD=10
TIMEOUT_CMD=300
MAX_RETRIES=3
MAX_SESSION_HOURS=4
RESTRICTED_PATHS="/etc,/root/.ssh"
RESTRICT_COMMANDS="rm,dd"
PIDS_LIMIT=100
MAX_SESSION_SECONDS=$((MAX_SESSION_HOURS * 3600))  # 14400
```

### 6.2 templates/detect.sh (99 lines, fingerprinting VPS)

**Variables detectadas y metodos:**

| Variable | Comando | Default si falla |
|----------|---------|------------------|
| RAM_TOTAL_GB | grep MemTotal /proc/meminfo / 1024 / 1024 | 0 |
| DISK_AVAIL_GB | df / --output=avail / 1024 / 1024 | 0 |
| CPU_CORES | nproc | 1 |
| HAS_DOCKER | command -v docker | false |
| HAS_INTERNET | curl -s --connect-timeout 5 https://google.com (o wget) | false |
| ARCH | uname -m | (vacio) |
| WHOAMI | whoami | (vacio) |

**Flujo de ejecución:**

```
1. verify_ownership()  -- limits.conf no owned por current user
2. sha256sum -c integrity.sum (si existe)
3. RAM_TOTAL_KB = MemTotal / 1024
4. RAM_TOTAL_GB = KB / 1024 / 1024
5. DISK_AVAIL_KB = df / --output=avail
6. CPU_CORES = nproc
7. HAS_DOCKER = command -v docker
8. curl/wget a google.com -> HAS_INTERNET
9. ARCH = uname -m
10. WHOAMI = whoami
11. Minimo: RAM >= 2GB, DISK >= 5GB -> exit 1 si no cumple
12. export de 7 variables
```

## 7. Mapa de Archivos Generados y Propietarios

### 7.1 En Maquina de Desarrollo (generador)

| Archivo | Tamano | Lines | Permisos |
|---------|--------|-------|----------|
| prod-agent.sh | 5388 B | 176 | 755 |
| modules/scanner.sh | 9967 B | 267 | 755 |
| modules/generator.sh | 27046 B | 788 | 755 |
| modules/security.sh | 6459 B | 182 | 755 |
| modules/validator.sh | 5717 B | 177 | 755 |
| modules/backup.sh | 4380 B | 134 | 755 |
| templates/agent.sh | 40381 B | 1128 | 755 |
| templates/detect.sh | 3220 B | 99 | 755 |

### 7.2 En VPS (despues de boot.sh)

| Archivo | Owner | Permisos | Efecto sobre ag-<proyecto> |
|---------|-------|----------|----------------------------|
| .agent/limits.conf | root:root | 644 | Solo lectura |
| .agent/agent.md | root:root | 644 | Solo lectura |
| .agent/agent.sh | root:root | 755 | Ejecuta, no modifica |
| boot.sh | root:root | 755 | No puede ejecutar (owner mismatch) |
| detect.sh | root:root | 755 | Ejecutable via sudo -u |
| integrity.sum | root:root | 644 | Solo lectura |
| .gitignore | root:root | 644 | Solo lectura |
| src/ | ag-* | 755 | Propio |
| context/ | ag-* | 755 | Propio |
| .agent/audit.log | ag-* | Creado en runtime | Escritura |
| .agent/memory/ | ag-* | Creado en generación | Escritura |

## 8. Modelo de Seguridad en Profundidad

### 8.1 Las 22 Capas de Defensa

| # | Capa | Mecanismo | Enforced By | File:Line |
|---|------|-----------|-------------|-----------|
| 1 | Shell | set -euo pipefail | Bash runtime | prod-agent.sh:10 |
| 2 | No-root | EUID check -> exit 1 | boot.sh generado | generator.sh:~390 |
| 3 | Sudo required | sudo -v cachea password | boot.sh generado | generator.sh:~400 |
| 4 | Integrity | sha256sum -c integrity.sum | boot.sh + agent.sh + detect.sh | multiples |
| 5 | User isolation | useradd -s /usr/sbin/nologin | boot.sh generado | generator.sh:~426 |
| 6 | No sudo | Sin grupo sudo en useradd | boot.sh generado | generator.sh:~426 |
| 7 | File ownership | chown root:root archivos config | boot.sh generado | generator.sh:~437 |
| 8 | Permissions | chmod 644/755 segun archivo | boot.sh generado | generator.sh:~440 |
| 9 | Ownership check | -O limits.conf -> abort | agent.sh:21, detect.sh:17 | templates/ |
| 10 | Command restrict | validate_command() bloquea lista | agent.sh:64 | templates/ |
| 11 | Path restrict | validate_path() prefix match | agent.sh:82 | templates/ |
| 12 | Session timeout | MAX_SESSION_HOURS -> exit | agent.sh:100 | templates/ |
| 13 | Command timeout | timeout $TIMEOUT_CMD | agent.sh:118 | templates/ |
| 14 | Shell injection | printf '%q' escaping | agent.sh:117 | templates/ |
| 15 | Audit trail | JSON lines a audit.log | agent.sh:55 | templates/ |
| 16 | Post-gen security | 6 checks en security.sh | security.sh | modules/ |
| 17 | Pre-deploy validate | 7 checks en validator.sh | validator.sh | modules/ |
| 18 | Project name val | Regex ^[a-zA-Z0-9_.-]+$ | generator.sh:22 | modules/ |
| 19 | rsync exclusions | 8 exclusiones de .env, .git, etc | generator.sh:176 | modules/ |
| 20 | Interpreter bypass | Bloquea -c/-e en python3/node/perl | validate_command() | templates/agent.sh:87 |
| 21 | PIDS_LIMIT enforce | ulimit -u desde limits.conf | load_limits() | templates/agent.sh:61 |
| 22 | Systemd sandbox | IPAddressDeny + Resource control | systemd service | generator.sh:systemd |

### 8.2 Matriz de Riesgos

| Riesgo | Mitigacion | Capa # | Efectividad |
|--------|------------|--------|-------------|
| Agente ejecuta rm -rf / | RESTRICT_COMMANDS bloquea rm | 10 | Alta |
| Agente accede a /etc/shadow | RESTRICTED_PATHS excluye /etc | 11 | Alta |
| Agente modifica sus limites | .agent/ es root:root | 7,8,9 | Alta |
| Agente escapa al usuario sudoer | ag-* no tiene sudo | 5,6 | Alta |
| Alguien modifica boot.sh en disco | integrity.sum detecta cambios | 4 | Alta |
| Atacante fuerza shell injection | printf '%q' escapa | 14 | Alta |
| Usuario corre boot.sh como root | EUID check + exit | 2 | Alta |
| .env se copia a src/ | rsync --exclude=.env | 19 | Alta |
| Agente hace fork bomb | ulimit -u desde limits.conf | 21 | Alta |
| Agente consume API sin control | Budget check antes de cada llm_call | templates/agent.sh | Media |
| Atacante usa python3 -c / node -e | validate_command() bloquea interprete inline | 20 | Alta |

### 8.3 Gaps de Seguridad Identificados

1. **validate_path no bloquea redirecciones**: `echo > /etc/passwd` no se detecta porque el comando es `echo`, no una escritura directa.
2. **No hay kernel-level sandboxing completo**: Hay sandbox systemd de red, pero no seccomp, AppArmor, SELinux o namespaces dedicados.
3. **No hay umask hardening**: agent.sh usa umask default del sistema.

## 9. Manejo de API Keys

| Runtime | Metodo de Captura | Almacenamiento |
|---------|-------------------|----------------|
| Claude | read -sp "API Key: " | export ANTHROPIC_API_KEY (memoria) |
| Gemini | read -sp "API Key: " | export GEMINI_API_KEY (memoria) |
| OpenCode | curl install.sh | Config local (no requiere key) |
| Ollama | curl install.sh + ollama pull | No requiere key |

Las API keys NUNCA se escriben a disco dentro del harness. Solo existen como variables de entorno en el proceso del agente.

## 10. Comandos Externos Utilizados (Inventario Completo)

| Comando | Usado en | Proposito |
|---------|----------|-----------|
| sha256sum | generator, agent, detect, security | Integridad |
| sudo | generator, boot.sh (generado) | Privileged ops |
| rsync | generator | Copia src/ con exclusiones |
| grep | scanner, agent, detect, security, validator | Busqueda en archivos |
| find | scanner, generator, security, validator | File counting, permisos |
| awk | scanner, agent, detect | Field extraction |
| cut | agent | Field extraction de limits.conf |
| realpath | scanner, validator, agent | Path resolution |
| bash -c | agent | Command execution (escaped) |
| timeout | agent | Command timeout |
| printf %q | agent | Shell injection prevention |
| curl | detect, boot.sh (generado), agent | Internet check, install, LLM API |
| wget | detect | Internet check (fallback) |
| useradd | boot.sh (generado) | Creacion de usuario ag-* |
| chown | boot.sh (generado), generator | File ownership |
| chmod | boot.sh (generado), generator | File permissions |
| date | agent | Timestamps |
| whoami | generator, agent, detect | Current user |
| free | agent | RAM display |
| df | agent, detect | Disk display/check |
| python3 | agent (templates/) | JSON parsing (historial LLM, payloads API) |
| detect-secrets | security | Secret scanning (opt) |
| tree | scanner | Directory tree (opt, fallback a find) |
| jq | scanner | JSON parsing (opt, fallback a node/grep+sed) |

## 11. Codigos de Error

| Código | Origen | Significado |
|--------|--------|-------------|
| Exit 0 | Cualquiera | Exito / usuario cancelo / sesion terminada |
| Exit 1 | prod-agent.sh | Modulo faltante en modules/ |
| Exit 1 | generator.sh | Nombre de proyecto invalido |
| Exit 1 | generator.sh | sudo mkdir -p fallo |
| Exit 1 | boot.sh (generado) | Ejecutado como root (EUID=0) |
| Exit 1 | boot.sh (generado) | Integridad fallida (sha256sum) |
| Exit 1 | boot.sh (generado) | sudo no disponible |
| Exit 1 | detect.sh | RAM < 2GB o disco < 5GB |
| Exit 1 | agent.sh | limits.conf es owned por agente (no root) |
| Exit 1 | validator.sh | Nombre no empieza con ag- |
| Exit 1 | validator.sh | VALIDATOR_ERRORS > 0 |
| Exit 1 | backup.sh | backup_restore: respaldo no encontrado |
| Exit 1 | backup.sh | backup_restore: SHA256 mismatch (corrupto) |
| Exit 1 | backup.sh | backup_restore: restauracion fallida |

## 12. Variables de Entorno del Sistema

| Variable | Set En | Consumido En | Proposito |
|----------|--------|-------------|-----------|
| PROJECT_NAME | scanner.sh | generator.sh | Identidad del proyecto |
| PROJECT_PATH | scanner.sh | generator.sh | Ruta del proyecto original |
| PROJECT_LANG | scanner.sh | generator.sh | Lenguaje para agent.md |
| PROJECT_FRAMEWORK | scanner.sh | generator.sh | Framework para agent.md |
| PROJECT_ENTRY_POINT | scanner.sh | generator.sh | Entry point |
| PROJECT_FILE_COUNT | scanner.sh | generator.sh | File stats |
| PROJECT_LINE_COUNT | scanner.sh | generator.sh | Line stats |
| PROJECT_HAS_DOCKER | scanner.sh | generator.sh | Docker flag |
| FORCE_MODE | prod-agent.sh | generator.sh | Override overwrite menu |
| UPDATE_MODE | prod-agent.sh | generator.sh | Update-only mode |
| BACKUP_DIR | backup.sh | backup_create, list, restore | Ruta de backups (/opt/nexus-one/backups/) |
| RAM_TOTAL_GB | detect.sh | boot.sh (generado) | Menu runtime |
| HAS_INTERNET | detect.sh | boot.sh (generado) | Menu runtime |
| RUNTIME | boot.sh (generado) | agent.sh (via env) | Modo agente: ollama/claude/gemini/opencode/none/auto |
| OLLAMA_MODEL | boot.sh (generado) | agent.sh (via env) | Modelo Ollama si aplica (qwen2.5-coder:7b/14b) |
| DEBUG_MODE | agent.sh (global) | conversation_loop | Toggle debug on/off via /debug |
| SESSION_COST | agent.sh (global) | Slash command /cost | USD acumulado por llamadas LLM |
| TOKEN_INPUT | agent.sh (global) | Slash command /cost | Tokens de entrada acumulados |
| TOKEN_OUTPUT | agent.sh (global) | Slash command /cost | Tokens de salida acumulados |
| COMPACT_STRATEGY | agent.sh (global) | conversation_loop | Estrategia de compactacion (none/sliding/summarize) |
| ANTHROPIC_API_KEY | boot.sh (generado) | agent.sh (via env) | Claude API |
| GEMINI_API_KEY | boot.sh (generado) | agent.sh (via env) | Gemini API |

## 13. Backup Module (modules/backup.sh)

### 13.1 Arquitectura

```
BACKUP_DIR="/opt/nexus-one/backups/"

<proyecto>.tar.gz      <- tar con 8 exclusiones
<proyecto>.sha256      <- sha256sum del tar.gz
<proyecto>.meta.json   <- project, backup_at, harness, harness_path, sha256, size_bytes
```

### 13.2 Flujo de Backup Automático

```
generator_generate()
  |-> backup_create()  -> tar.gz + sha256 + meta.json
  |-> rsync src/       -> /opt/ag-<proyecto>/src/
```

Se invoca en generator.sh antes del rsync. No requiere confirmacion del usuario.
Un backup por proyecto (se sobrescribe en el proximo --scan).

### 13.3 Restauracion

```
backup_restore("npm-2026")
  1. Verificar existencia de npm-2026.tar.gz
  2. sha256sum -c npm-2026.sha256
  3. Si /opt/npm-2026/ existe, preguntar confirmacion
  4. rm -rf target_dir (si confirmo)
  5. tar -xzf backup -C /opt/
```

### 13.4 Exclusiones del Backup

Mismas que rsync en generator.sh: .git, node_modules, venv, __pycache__, build, dist, target, logs, *.log.

## Referencias

- **AGENTS.md** (`docs/AGENTS.md`): Fuente de verdad arquitectonica
- **ROADMAP.md** (`prod-agent/context/ROADMAP.md`): Estado del proyecto y fases
- **COMPENDIO.md** (`docs/COMPENDIO.md`): Compendio de sesion

#End Development By Angel Esquivel (CyberSecurity) [NEXUS ONE 2026]
- Licensing: D3 — Source Available (BSL + Commercial)
