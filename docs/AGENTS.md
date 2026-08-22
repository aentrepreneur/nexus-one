# AGENTS.md -- Fuente de Verdad Arquitectonica Unificada
# =================================================================
# Especificacion completa del framework NEXUS ONE. Contiene la
# arquitectura del generador de harnesses (prod-agent) y del
# subsistema de remediación (remediation). Reglas de desarrollo,
# flujos de ejecución, modelo de seguridad y dependencias.
# TODO documento derivado debe referenciar este archivo como
# autoridad en arquitectura, reglas, flujo y seguridad.
# =================================================================

## 1. Arquitectura General

### 1.1 Arbol del Proyecto

```
/opt/nexus-one/
+-- README.md                              # Repo raíz (GitHub)
+-- docs/                                  # Documentación global
|   +-- AGENTS.md                          # Este archivo (fuente de verdad)
|   +-- COMPENDIO.md                       # Compendio narrativo de sesion
|   +-- TECHNICAL.md                       # Documentación técnica detallada
|   +-- CHANGELOG.md                       # Historial de versiones
|   +-- PENDINGS.md                        # Issues y estado de resolucion
|   +-- REMOTE_TESTING.md                  # Procedimiento de pruebas LLM remoto
+-- prod-agent/                            # Generador de harnesses
|   +-- prod-agent.sh                      # Entry point CLI
|   +-- modules/                           # scanner, generator, security, validator, backup
|   +-- templates/                         # agent.sh, detect.sh (copia literal)
|   +-- context/
|   |   +-- ROADMAP.md                     # Roadmap de desarrollo del generador
|   +-- docs/
|       +-- README.md                      # README del subproyecto
+-- remediation/                           # Subsistema de remediación
|   +-- remediation.sh                     # Entry point CLI
|   +-- modules/                           # scanner, snapshot, verifier, remediator, reporter
|   +-- tools/                             # 8 herramientas (4 no-LLM, 4 LLM)
|   +-- templates/                         # remediate-agent.sh
|   +-- context/
|   |   +-- ROADMAP.md                     # Roadmap de remediation
|   +-- docs/
|   |   +-- README.md                      # README del subsistema
|   +-- remedy/                            # Output: remedy-<proyecto>.json
|   +-- snapshots/                         # Backups pre-remediation
+-- tests/                                 # Proyectos de prueba
+-- backups/                               # Backups automaticos de --scan
```

### 1.2 Dos Subsistemas

NEXUS ONE se compone de dos subsistemas independientes pero integrables:

| Subsistema | Proposito | Entry Point | Output |
|------------|-----------|-------------|--------|
| prod-agent | Genera harnesses de agente LLM para proyectos existentes | `prod-agent.sh --scan /ruta` | `/opt/ag-<proyecto>/` |
| remediation | Analiza, repara y optimiza proyectos de código | `remediation.sh --remediate /ruta` | `remedy/<proyecto>.json` |

Flujo de integración tipico:
```
# Opcion A: Remediar primero, luego generar harness
./remediation.sh --remediate /opt/proyecto
./prod-agent.sh --scan /opt/proyecto

# Opcion B: Inyectar tools de remediation en el harness
./prod-agent.sh --scan --with-remediation /opt/proyecto
# Luego dentro del harness, el agente ejecuta tools de remediation
```

## 2. Reglas de Desarrollo Compartidas

Reglas que aplican a AMBOS subsistemas. Las reglas especificas de cada
uno se listan en su sección correspondiente.

| # | Regla | Descripción |
|---|-------|-------------|
| 1 | Bash puro 4+ | Zero dependencias externas. Sin Python, Node, Go u otros runtimes en maquina de generación. |
| 2 | Modular por source | Cada modulo en modules/ es autocontenido y sourceable desde el entry point. |
| 3 | set -euo pipefail | Obligatorio en todo entry point. Heredado por modulos sourceados. |
| 4 | Headers/footers NEXUS ONE | Todo archivo .sh, .py, .md, .yml, .cfg debe iniciar y terminar con bloque de desarrollo. |
| 5 | ASCII puro | Prohibidos emojis, emoticones y caracteres no latinos en código y documentación. |
| 6 | PascalCase en output | Todo texto visible al usuario en PascalCase, sin preposiciones. Tags [MODULO] se conservan. |
| 7 | API keys nunca a disco | Captura via read -sp en runtime. Solo existen como variables de entorno. |
| 8 | Seguridad por restriccion OS | El modelo de seguridad NO depende del prompt del LLM, sino de permisos de archivo, usuarios sin sudo y verificación de integridad. |
| 9 | Integrity post-generation | Todo harness o snapshot debe generar checksum SHA256 inmediatamente despues de crearse. |
| 10 | Validación de nombre | project_name pasa regex `^[a-zA-Z0-9_.-]+$` para prevenir path traversal. |
| 11 | Backup automático | Cada operación destructiva genera backup antes de modificar. |
| 12 | Sin shell=True | Prohibido `shell=True`, `os.system()`, `subprocess.Popen(shell=True)`. Usar `asyncio.create_subprocess_exec()`. |

## 3. Subsistema prod-agent (Generador de Harnesses)

Genera `/opt/ag-<proyecto>/` autocontenido y portable listo para
desplegar en cualquier VPS via SCP. Un comando escanea el proyecto,
analiza su estructura, genera el harness y lo valida.

### 3.1 Arbol del Generador

```
prod-agent/
+-- prod-agent.sh                    # Entry point CLI
+-- modules/
|   +-- scanner.sh                   # Análisis de proyectos (267 lines)
|   +-- generator.sh                 # Creacion de harnesses (788 lines)
|   +-- security.sh                  # Escaneo post-generación (182 lines)
|   +-- validator.sh                 # Validación de integridad (177 lines)
|   +-- backup.sh                    # Backup, list y restore (134 lines)
+-- templates/
|   +-- agent.sh                     # Loop ReAct + 10 mecanismos seguridad (1128 lines)
|   +-- detect.sh                    # Fingerprinting VPS (99 lines)
+-- context/
|   +-- ROADMAP.md                   # Roadmap de desarrollo
+-- docs/
    +-- README.md                    # README del subproyecto
```

### 3.2 Flujo de Ejecución (4 Modulos)

```
prod-agent --scan /opt/mi-proyecto
  |
  |-- [--dry-run]: Si flag presente, scanner corre pero generator
  |   solo muestra preview del arbol del harness. No se escribe nada.
  |
  v
[prod-agent.sh] CLI parser
  |   Valida args, determina modo (scan/validate/list/help)
  |   source modules/
  v
[scanner.sh]
  |   detect_language()     -> PROJECT_LANG
  |   detect_framework()    -> PROJECT_FRAMEWORK
  |   detect_entry_point()  -> PROJECT_ENTRY_POINT
  |   detect_features()     -> PROJECT_HAS_DOCKER, HAS_ENV_EXAMPLE
  |   detect_dependencies() -> PROJECT_DEPS array
  |   count_files()         -> PROJECT_FILE_COUNT, PROJECT_LINE_COUNT
  |   generate_tree()       -> estructura.txt
  |   export 9 vars
  v
[generator.sh]
  |   [DRY-RUN] Return inmediato con preview del arbol. No se
  |   escriben archivos, no se llama backup ni rsync.
  |
  |   Si /opt/ag-<proyecto>/ existe:
  |     --dry-run: solo muestra que ya existe, no pregunta nada
  |     --force: borra y regenera
  |     --update: regenera config, preserva src/
  |     default: menu interactivo (4 opciones)
  |
  |   create_limits()       -> .agent/limits.conf (8 secciones INI)
  |   create_agent_md()     -> .agent/agent.md (YAML + MD)
  |   cp templates/agent.sh -> .agent/agent.sh
  |   generator_create_default_tools() -> .agent/tools/hello.sh
  |   generator_create_default_expertise() -> .agent/expertise/project_rules.md
  |   generator_create_default_tasks() -> .agent/tasks/README.task
  |   Si WITH_REMEDIATION:
  |     generator_create_remediation_tools() -> .agent/tools/ (4 tools)
  |   create_boot_sh()      -> boot.sh (heredoc inline, 10 funciones)
  |   create_uninstall_sh() -> uninstall.sh (heredoc inline, 6 pasos)
  |   cp templates/detect.sh -> detect.sh
  |   create_context()      -> context/README.md
  |   create_seguridad()    -> context/SEGURIDAD.md
  |   cp estructura.txt     -> context/estructura.txt
  |   backup_create()       -> /opt/nexus-one/backups/ (tar.gz + sha256 + meta)
  |   rsync proyecto/       -> src/ (8 exclusiones)
  |   create_integrity()    -> integrity.sum (9-10 archivos)
  |
  v
[security.sh]
  |   6 checks: limits, permissions, secrets, gitignore, env, integrity
  v
[validator.sh]
  |   7 checks: structure, boot.sh, limits, agent.md, context, src, gitignore
  v
[Reporte final] PASS/FAIL con detalle por componente
```

### 3.3 Variables Exportadas del Scanner

Estas 9 variables se exportan en scanner.sh y se consumen en generator.sh:

| Variable | Tipo | Origen |
|----------|------|--------|
| PROJECT_NAME | string | basename del path |
| PROJECT_PATH | string | realpath del proyecto |
| PROJECT_LANG | string | detect_language() |
| PROJECT_FRAMEWORK | string | detect_framework() |
| PROJECT_PKG_MANAGER | string | detect_language() |
| PROJECT_ENTRY_POINT | string | detect_entry_point() |
| PROJECT_FILE_COUNT | int | count_files() |
| PROJECT_LINE_COUNT | int | count_files() |
| PROJECT_HAS_DOCKER | bool | detect_features() |
| PROJECT_HAS_ENV_EXAMPLE | bool | detect_features() |
| PROJECT_DEPS | array | detect_dependencies() |

### 3.4 Metodo de Deteccion de Lenguaje

Prioridad: Python > Go > Node > Bash > Ruby > Rust.

Orden de chequeo por archivo caracteristico:

| Lenguaje | Archivo Indicador |
|----------|-------------------|
| python | main.py / setup.py / pyproject.toml / requirements.txt |
| go | go.mod |
| javascript | package.json |
| rust | Cargo.toml |
| bash | *.sh files exist |

Entry point por lenguaje:

| Lenguaje | Busqueda | Default |
|----------|----------|---------|
| python | main.py, app.py, manage.py, run.py | main.py |
| javascript | package.json "main", index.js, app.js, server.js | index.js |
| go | main.go | main.go |
| rust | src/main.rs | src/main.rs |
| bash | *.sh | bootstrap.sh |

### 3.5 Exclusiones de rsync

```
--exclude=.env --exclude=node_modules --exclude=.git
--exclude=venv --exclude=__pycache__ --exclude=build
--exclude=dist --exclude=target --exclude=logs --exclude='*.log'
```

### 3.6 Flujo de Despliegue en VPS

```
Usuario VPS (sudoer)
  | scp -r /opt/ag-<proyecto> user@vps:~/
  | ssh user@vps
  v
./ag-<proyecto>/boot.sh
  |
  |-- check_security()       EUID=0? exit 1, sudo -v, sha256sum -c
  |-- setup_agent_user()     useradd -s /usr/sbin/nologin ag-<proyecto>,
  |                          chown root:root archivos config
  |                          chown condicional de models/ si existe
  |-- detect_resources()     sudo -u ag-* ./detect.sh (tolerante a fallos)
  |                          RAM, disco, CPU, Docker, internet, arquitectura
  |-- evaluate_resources()   RAM < 2GB -> RUNTIME=none forzado
  |                          RAM >= 8GB -> viable Ollama
  |                          Internet -> viable cloud APIs
  |-- select_runtime()       Menu filtrado por recursos:
  |                          1) Ollama 7b  2) Ollama 14b
  |                          3) Claude    4) OpenCode
  |                          5) Gemini    r) Recovery
  |                          --runtime <name> override forzado
  |-- install_runtime()      [Self-contained]
  |   +-- ollama runtime:     descarga ollama binary a models/bin/ollama
  |   |                        (zst preferido, tgz fallback)
  |   |                        crea models/ollama/ para modelos
  |   +-- opencode runtime:    descarga binary de GitHub a models/bin/opencode
  |   +-- claude/gemini:      exporta API keys con ${VAR:-} (set -u safe)
  |   +-- python3/node/bash:  no-op (solo crea models/bin/)
  |-- start_ollama_local()   Si runtime ollama:
  |   |                        inicia models/bin/ollama serve via nohup
  |   |                        como ag-<proyecto> con OLLAMA_MODELS local
  |   |                        espera hasta 30s a que responda
  |   +-- ollama pull $MODEL  descarga modelo a models/ollama/
  |-- activate_agent()       exec sudo -u ag-* env
  |                            RUNTIME=... PATH=$HARNESS_DIR/models/bin:...
  |                            OLLAMA_MODELS=$HARNESS_DIR/models/ollama/
  |                            .agent/agent.sh
  |
  |-- (systemd alternativo)  install_service() instala nexus-<proyecto>.service
  |                          con sandbox de red (IPAddressDeny) y journald
```

### 3.7 Variables Exportadas del detect.sh (Runtime)

| Variable | Tipo | Origen |
|----------|------|--------|
| RAM_TOTAL_GB | int | grep MemTotal /proc/meminfo |
| DISK_AVAIL_GB | int | df / --output=avail |
| CPU_CORES | int | nproc |
| HAS_DOCKER | bool | command -v docker |
| HAS_INTERNET | bool | curl/wget a google.com |
| ARCH | string | uname -m |
| WHOAMI | string | whoami |
| RUNTIME | string | boot.sh evaluate_resources() |
| OLLAMA_MODEL | string | boot.sh select_runtime() |

### 3.8 Modelo de Seguridad

Tres capas de defensa en profundidad:

| Capa | Mecanismo | Implementación |
|------|-----------|----------------|
| 1. Boot | No-root, sudo -v, integrity.sum | boot.sh check_security() |
| 2. Usuario | ag-<proyecto> sin sudo, sin shell | boot.sh setup_agent_user() |
| 3. Archivos | root:root en config, ag-* en src/ | boot.sh chown/chmod |

Jerarquía de usuarios en VPS:

| Usuario | Rol | SSH | Sudo |
|---------|-----|-----|------|
| root | No se usa | Deshabilitado | N/A |
| (tu usuario) | Bootstrap | Si | Si (con password) |
| ag-<proyecto> | Agente runtime | No | No |

Propiedad de archivos en harness:

| Archivo | Owner | Permisos | Efecto |
|---------|-------|----------|--------|
| .agent/limits.conf | root:root | 644 | Agente solo lee |
| .agent/agent.md | root:root | 644 | Agente solo lee |
| .agent/agent.sh | root:root | 755 | Agente ejecuta, no modifica |
| boot.sh | root:root | 755 | Agente no puede ejecutar (owner mismatch) |
| uninstall.sh | root:root | 755 | Solo root ejecuta |
| detect.sh | root:root | 755 | Ejecutable via sudo -u |
| integrity.sum | root:root | 644 | Solo lectura |
| src/ context/ | ag-<proyecto> | 755 | Agente es dueno |
| .agent/audit.log | ag-<proyecto> | Creado en runtime | Agente escribe |

### 3.9 Archivo limits.conf (8 Secciones)

```
[BUDGET]       BUDGET_USD=10, BILLING_MODE=consumption
[TIMEOUT]      TIMEOUT_CMD=300, SESSION=14400, IDLE=600
[RETRIES]      MAX_RETRIES=3, BACKOFF=exponential, BASE_DELAY=5
[PATHS]        RESTRICTED=/etc,/root/.ssh,/var/lib/docker,/boot,/sys
[COMMANDS]     RESTRICT=rm,dd,format,mkfs,reboot,shutdown,poweroff,halt
               DANGER=chmod,chown,mount,umount,iptables,systemctl
[RATE_LIMIT]   MAX_REQUESTS_PER_MINUTE=30, PER_HOUR=500
[SESSION]      MAX_HOURS=4, CONCURRENT=1, TIMEOUT_ACTION=kill
[SYSTEM]       PIDS_LIMIT=100, INTEGRITY_CHECK=true
```

### 3.10 integrity.sum (12-13 Archivos)

```
.agent/limits.conf .agent/agent.md .agent/agent.sh
.agent/tools/hello.sh .agent/expertise/project_rules.md
uninstall.sh boot.sh detect.sh .gitignore
context/README.md context/SEGURIDAD.md context/estructura.txt
nexus-<proyecto>.service   # Opcional, solo con --systemd
```

Excluye: src/, .agent/audit.log, .agent/tasks/, .agent/results/.

### 3.11 Limites de Seguridad en agent.sh (Runtime Enforcement)

El template `templates/agent.sh` implementa 10 mecanismos en runtime:

| # | Mecanismo | Default | Que Previene |
|---|-----------|---------|--------------|
| 1 | Ownership check | limits.conf owned by root? | Bypass de boot.sh |
| 2 | Integrity check | sha256sum -c | Modificación de harness |
| 3 | Command restriction | rm,dd,format,mkfs,.. | Comandos destructivos |
| 4 | Path restriction | /etc,/root/.ssh,/boot,/sys | Acceso a sistema |
| 5 | Session timeout | 4h | Sesiones eternas |
| 6 | Command timeout | 300s | Comandos colgados |
| 7 | Shell injection prev | printf '%q' | Injection de comandos |
| 8 | Audit logging | JSON a audit.log | Sin trazabilidad |
| 9 | Interpreter bypass | Bloquea -c/-e en python3/node/perl | Ejecución inline maliciosa |
| 10 | PIDS_LIMIT | ulimit -u 100 | Fork bombs |

### 3.12 Reglas Especificas del Generador

| # | Regla | Descripción |
|---|-------|-------------|
| 13 | Variables exportadas | Scanner exporta 9 vars globales para consumo del generator. |
| 14 | Templates: cp literal | agent.sh y detect.sh se copian literalmente. boot.sh, limits.conf, agent.md se generan via heredoc. |
| 15 | Syntax systemd | Servicio debe incluir IPAddressDeny, CPUQuota, MemoryMax, StandardOutput=journal, Restart=on-failure. |
| 16 | Interpreter bypass | validate_command() bloquea -c/-e en python3/node/perl. No bloquear los binarios. |
| 17 | Plugin tools | Todo .sh en .agent/tools/ se carga via load_tools(). Exporta TOOLNAME_DESCRIPTION + toolname_run(). |
| 18 | Daemon mode | --daemon activa daemon_loop() que procesa .agent/tasks/*.task en loop de 30s. |
| 19 | Agent expertise | .agent/expertise/*.md se carga como contexto en build_system_prompt(). Solo lectura. |
| 20 | Model router | router_classify() heuristica por longitud + keywords. Sin llamada LLM para clasificacion. |
| 21 | --with-remediation | Copia 4 tools no-LLM desde remediation/tools/ a .agent/tools/ del harness. |
| 22 | Uninstall autogenerado | Cada harness incluye uninstall.sh con 6 pasos de limpieza. Incluido en integrity.sum. |
| 23 | OpenAI provider | agent.sh soporta OpenAI via llm_call_openai(). Modelos: gpt-4o-mini (cheap), gpt-4o (medium), o3-mini (high). |
| 24 | Prueba LLM remoto | Todo proveedor debe validarse en VPS antes de producción. 40 tests, 40 PASS, 0 FAIL. Ver docs/REMOTE_TESTING.md. |

### 3.13 Comandos de Prueba

```bash
# Generación de harness
./prod-agent.sh --scan /opt/mi-proyecto
./prod-agent.sh --scan --dry-run /opt/mi-proyecto
./prod-agent.sh --scan --force /opt/mi-proyecto
./prod-agent.sh --scan --update /opt/mi-proyecto
./prod-agent.sh --scan --systemd /opt/mi-proyecto
./prod-agent.sh --scan --with-remediation /opt/mi-proyecto
./prod-agent.sh --scan --force --systemd --with-remediation /opt/mi-proyecto

# Validación
./prod-agent.sh --validate /opt/ag-mi-proyecto
./prod-agent.sh --list
./prod-agent.sh --help
./prod-agent.sh --version
```

## 4. Subsistema remediation

Analiza, repara y optimiza proyectos de código existentes mediante
un ciclo automatizado de descubrimiento, propuesta, aplicación y
verificación de cambios, asistido por LLM cuando esta disponible.

### 4.1 Arbol de Archivos

```
remediation/
+-- remediation.sh                    # Entry point CLI
+-- modules/
|   +-- scanner.sh                    # Escaneo profundo de hallazgos
|   +-- snapshot.sh                   # Backup del proyecto original
|   +-- verifier.sh                   # Descubrimiento y validación de procesos
|   +-- remediator.sh                 # Orquestador de cambios interactivos
|   +-- reporter.sh                   # Generador de reportes JSON
+-- tools/
|   +-- scan_secrets.sh               # Deteccion de secretos (no LLM)
|   +-- extract_secrets.sh            # Migracion de claves a .env (no LLM)
|   +-- audit_deps.sh                 # Auditoria de dependencias (no LLM)
|   +-- verify_process.sh             # Ejecutor de procesos (no LLM)
|   +-- gen_docs.sh                   # Generador de documentación (LLM)
|   +-- refactor_plan.sh              # Planificador de refactor (LLM)
|   +-- add_tests.sh                  # Generador de test skeletons (LLM)
|   +-- add_lint.sh                   # Configurador de linters (LLM)
+-- templates/
|   +-- remediate-agent.sh            # Plantilla del agente de remediation
+-- context/
|   +-- ROADMAP.md                    # Roadmap de desarrollo
+-- docs/
|   +-- README.md                     # Documentación detallada
+-- remedy/                           # Output: remedy-<proyecto>.json
+-- patterns/                         # Patrones extraidos (futuro)
+-- snapshots/                        # Backups: snapshot-<proyecto>-<fecha>.tar.gz
```

### 4.2 Flujo de Ejecución (6 Fases)

```
remediation.sh --remediate /opt/proyecto
  |
  |-- [--dry-run]: Si flag presente, fase 1, 5 y 6 se omiten
  |   y se muestra preview de hallazgos. Fases 2-4 son iguales.
  |
  |-- [FASE 1] SNAPSHOT
  |     snapshot_create() -> snapshots/snapshot-proyecto-*.tar.gz
  |     [DRY-RUN] Omitido
  |
  |-- [FASE 2] LLM DETECTION
  |     detect_llm() -> ollama | claude | gemini | opencode | none
  |     Si none: pregunta usuario si continuar sin LLM
  |     [DRY-RUN] Auto-continua sin preguntar
  |
  |-- [FASE 3] DISCOVER + BASELINE
  |     verifier_discover() -> procesos del proyecto
  |     verifier_baseline() -> exit codes + stdout
  |
  |-- [FASE 4] ANALYZE
  |     scanner_run() -> 5 sub-análisis
  |       + tools sin LLM (scan_secrets, audit_deps)
  |       + tools con LLM si disponible (gen_docs, refactor_plan, add_tests, add_lint)
  |
  |-- [FASE 5] REMEDIATE (ciclo interactivo)
  |     remediator_run()
  |       1. Propone cambio 2. Pregunta usuario [y/n]
  |       3. Aplica cambio 4. Re-ejecuta baseline
  |       5. Si hay regresion: ofrece rollback
  |     [DRY-RUN] Preview de hallazgos, no se aplican cambios
  |
  |-- [FASE 6] REPORT
        reporter_generate() -> remedy/remedy-proyecto.json
```

### 4.3 Tools y Capa LLM

| Tool | Requiere LLM? | Proposito |
|------|---------------|-----------|
| scan_secrets.sh | NO | 25 patrones de regex para detectar API keys, tokens, passwords |
| extract_secrets.sh | NO | Migra claves del código a .env + .env.example |
| audit_deps.sh | NO | pip/npm audit, deteccion de dependencias fijadas |
| verify_process.sh | NO | Descubre y ejecuta procesos del proyecto con timeout |
| gen_docs.sh | SI | Genera README.md usando Ollama/Claude/Gemini/OpenCode |
| refactor_plan.sh | SI | Planifica refactor estructural usando LLM |
| add_tests.sh | SI | Genera test skeletons para funciones publicas |
| add_lint.sh | SI* | Configura linter/formatter segun lenguaje |

### 4.4 Modelo de Seguridad

Reglas de datos en remedy JSON:
1. NUNCA se guardan valores de API keys, tokens o passwords.
2. Solo se almacenan referencias: archivo, línea, tipo, severidad.
3. Las llaves reales solo existen en snapshot.tar.gz y .env del proyecto.
4. El .env del proyecto se crea con permisos 600 (solo dueno).

Reglas de ejecución:
1. Siempre se crea snapshot ANTES de cualquier modificación.
2. Cada cambio propuesto es interactivo: usuario decide [y/n].
3. Post-cambio se re-ejecuta baseline para detectar regresiones.
4. Rollback automático disponible si el usuario lo confirma.
5. Tools con LLM solo se ejecutan si hay LLM detectado o usuario autoriza.

### 4.5 Formato remedy-<proyecto>.json

```json
{
  "project": "nombre-proyecto",
  "project_path": "/opt/proyecto",
  "scanned_at": "2026-05-20T14:30:00Z",
  "llm_used": "ollama",
  "baseline": {
    "processes": [{"name": "pytest", "exit_code": 0}],
    "stats": {"files": 34, "lines": 2891}
  },
  "findings": [
    {
      "id": "SEC-001",
      "type": "hardcoded_secret",
      "severity": "high",
      "file": "src/config.py:15",
      "applied": true
    }
  ],
  "changes": {
    "applied": [],
    "rejected": [],
    "total_applied": 1,
    "total_rejected": 0
  },
  "signature": {
    "generator": "nexus-duo-0.1.0",
    "timestamp": "2026-05-20T14:30:00Z"
  }
}
```

### 4.6 Reglas Especificas de Remediation

| # | Regla | Descripción |
|---|-------|-------------|
| 25 | Tools autocontenidas | Cada tool en tools/ es ejecutable standalone. |
| 26 | Sin LLM = funcional | Todas las tools sin LLM deben funcionar sin modelo externo. |
| 27 | LLM = opcional | Tools con LLM deben tener fallback o reportar "no disponible". |
| 28 | Seguridad por omision | remedy JSON nunca guarda valores de secrets. |
| 29 | Snapshot first | Ningun cambio se aplica sin backup previo. |
| 30 | Interactivo obligatorio | Toda modificación requiere confirmacion del usuario. |
| 31 | Consistencia seguridad | Documentación derivada debe reflejar enforcement real de prod-agent. |

### 4.7 Comandos de Prueba

```bash
# Remediation completa
./remediation.sh --remediate /opt/mi-proyecto

# Preview sin aplicar cambios
./remediation.sh --remediate --dry-run /opt/mi-proyecto

# Modo vigilancia continua
./remediation.sh --watch /opt/mi-proyecto [intervalo_segundos]

# Listar / ver remedies
./remediation.sh --list
./remediation.sh --remedy mi-proyecto

# Tools standalone
bash tools/scan_secrets.sh /opt/mi-proyecto
bash tools/audit_deps.sh /opt/mi-proyecto
bash tools/gen_docs.sh /opt/mi-proyecto ollama
bash tools/verify_process.sh /opt/mi-proyecto
```

### 4.8 Integración con prod-agent

El subsistema de remediation se integra con prod-agent de dos formas:

1. **Pre-generación**: Remediar el proyecto antes de generar el harness.
2. **Inyeccion de tools**: `--with-remediation` copia 4 tools no-LLM
   (`scan_secrets.sh`, `audit_deps.sh`, `verify_process.sh`, `extract_secrets.sh`)
   a `.agent/tools/` del harness para que el agente las ejecute en runtime.

## 5. Variables de Entorno (Unificadas)

### 5.1 Variables de Generación

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
| SYSTEMD_MODE | prod-agent.sh | generator.sh | Con servicio systemd |
| WITH_REMEDIATION | prod-agent.sh | generator.sh | Inyecta tools remediation |

### 5.2 Variables de Runtime (VPS)

| Variable | Set En | Consumido En | Proposito |
|----------|--------|-------------|-----------|
| HARNESS_DIR | boot.sh | agent.sh | Directorio raíz del harness |
| AGENT_USER | boot.sh | agent.sh | Usuario ag-<proyecto> |
| RAM_TOTAL_GB | detect.sh | boot.sh | RAM disponible en VPS |
| HAS_INTERNET | detect.sh | boot.sh | Conectividad a internet |
| RUNTIME | boot.sh | agent.sh | Modo del agente |
| OLLAMA_MODEL | select_runtime | agent.sh | Modelo Ollama |
| OLLAMA_MODELS | install_runtime | agent.sh, ollama serve | Ruta local models/ollama/ |
| ANTHROPIC_API_KEY | boot.sh | agent.sh | Claude API |
| GEMINI_API_KEY | boot.sh | agent.sh | Gemini API |
| DEBUG_MODE | agent.sh | conversation_loop | Debug on/off via /debug |

## 6. Referencias Cruzadas

### 6.1 Documentación Relacionada

| Documento | Proposito | Path |
|-----------|-----------|------|
| README.md | Repo raíz (GitHub) | /opt/nexus-one/README.md |
| QUICKSTART.md | Guia practica paso a paso | /opt/nexus-one/docs/QUICKSTART.md |
| CONCEPT.md | Concepto: que ganas con un agente orquestador | /opt/nexus-one/docs/CONCEPT.md |
| COMPENDIO.md | Compendio narrativo de sesion | /opt/nexus-one/docs/COMPENDIO.md |
| TECHNICAL.md | Documentación técnica detallada | /opt/nexus-one/docs/TECHNICAL.md |
| CHANGELOG.md | Historial de versiones | /opt/nexus-one/docs/CHANGELOG.md |
| PENDINGS.md | Issues y estado de resolucion | /opt/nexus-one/docs/PENDINGS.md |
| REMOTE_TESTING.md | Pruebas LLM en remoto | /opt/nexus-one/docs/REMOTE_TESTING.md |
| ROADMAP.md (prod-agent) | Roadmap del generador | prod-agent/context/ROADMAP.md |
| ROADMAP.md (remediation) | Roadmap de remediation | remediation/context/ROADMAP.md |
| README.md (prod-agent) | README del generador | prod-agent/docs/README.md |
| README.md (remediation) | README de remediation | remediation/docs/README.md |

### 6.2 Archivos del Proyecto (Scripts)

| Archivo | Rol |
|---------|-----|
| prod-agent/prod-agent.sh | Entry point CLI del generador |
| prod-agent/modules/scanner.sh | Análisis de proyectos |
| prod-agent/modules/generator.sh | Creacion de harnesses |
| prod-agent/modules/security.sh | Escaneo de seguridad post-generación |
| prod-agent/modules/validator.sh | Validación de estructura |
| prod-agent/modules/backup.sh | Backup, list y restore |
| prod-agent/templates/agent.sh | Plantilla de loop LLM + 10 mecanismos seguridad |
| prod-agent/templates/detect.sh | Plantilla de deteccion recursos VPS |
| remediation/remediation.sh | Entry point CLI de remediation |
| remediation/modules/scanner.sh | Escaneo profundo de proyectos |
| remediation/modules/snapshot.sh | Backup de proyectos |
| remediation/modules/verifier.sh | Descubrimiento de procesos |
| remediation/modules/remediator.sh | Orquestador de cambios |
| remediation/modules/reporter.sh | Generador de reportes JSON |
| remediation/tools/scan_secrets.sh | Deteccion de secretos (25 patrones) |
| remediation/tools/extract_secrets.sh | Migracion de claves a .env |
| remediation/tools/audit_deps.sh | Auditoria de dependencias |
| remediation/tools/verify_process.sh | Ejecución de procesos |
| remediation/tools/gen_docs.sh | Generación de documentación via LLM |
| remediation/tools/refactor_plan.sh | Plan de refactor via LLM |
| remediation/tools/add_tests.sh | Generación de test skeletons |
| remediation/tools/add_lint.sh | Configuración de linters |
| remediation/templates/remediate-agent.sh | Template del agente de remediation |

#End Development By Angel Esquivel (CyberSecurity) [NEXUS ONE 2026]
