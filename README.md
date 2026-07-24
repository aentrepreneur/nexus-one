# NEXUS ONE: Harness Engineering Framework
# =================================================================
# Framework para generar agentes LLM autonomos a partir de proyectos
# existentes. Genera /opt/ag-<proyecto>/ autocontenido y portable
# para desplegar en cualquier VPS via SCP + ./boot.sh.
# =================================================================

## Two-Tier Overview

NEXUS ONE tiene **dos niveles** que debes entender por separado:

### NIVEL 1: NEXUS ONE (/opt/nexus-one/) -- El GENERADOR

Contiene el generador de harnesses (`prod-agent/`) y el subsistema de
remediación (`remediation/`).

```
nexus-one/
+-- prod-agent/          <-- El generador (lo ejecutas en tu maquina dev)
|   +-- prod-agent.sh    <-- Entry point CLI
|   +-- modules/         <-- scanner, generator, security, validator, backup
|   +-- templates/       <-- agent.sh, detect.sh (copia literal a cada harness)
+-- remediation/         <-- Sub-sistema de herramientas de remediación
+-- tests/               <-- Proyecto de prueba (custom-2026)
+-- docs/                <-- Documentación del framework
```

**Que haces aqui**: Ejecutas `prod-agent.sh --scan /opt/mi-proyecto` y genera
un harness autocontenido en `/opt/ag-mi-proyecto/`.

### NIVEL 2: ag-<proyecto> (/opt/ag-mi-proyecto/) -- El GENERADO (harness)

Agente LLM listo para desplegar en cualquier VPS.

```
/opt/ag-mi-proyecto/
+-- boot.sh               <-- Entry point remoto (SCP + ./boot.sh)
+-- detect.sh             <-- Fingerprinting de recursos VPS
+-- .agent/
|   +-- agent.sh          <-- Loop ReAct (conversacion + herramientas)
|   +-- agent.md          <-- Identidad YAML + instrucciones
|   +-- limits.conf       <-- Restricciones de seguridad
+-- src/                  <-- Código original del proyecto
+-- integrity.sum         <-- SHA256 de todos los archivos
```

**Que haces aqui**: Copias el harness a un VPS y ejecutas `./boot.sh`.

### Diagrama de Flujo Completo

```
MAQUINA DE DESARROLLO                    VPS DESTINO
=======================                  ===========
1. Tienes un proyecto:                   5. Copias el harness:
   /opt/mi-proyecto/                        scp -r /opt/ag-mi-proyecto user@vps:~/

2. Ejecutas el generador:                6. Ejecutas boot.sh:
   ./prod-agent.sh --scan /opt/mi-proyecto   ssh user@vps './ag-mi-proyecto/boot.sh'

3. El generador analiza:                 7. boot.sh hace todo automático:
   - Lenguaje, framework, entry point        - Verifica integridad (SHA256)
   - Dependencias, estructura                - Crea usuario aislado (sin shell)
   - Genera templates + copia src/           - Detecta RAM, disco, CPU, Docker

4. Resultado:                              - Instala runtime (Ollama o cloud API)
   /opt/ag-mi-proyecto/                    - Lanza conversation loop
   (autocontenido, portable)             8. Agente corriendo
```

### Regla Fundamental

| Nivel | Donde se ejecuta | Que produce | Comando |
|-------|------------------|-------------|---------|
| NEXUS ONE (generador) | Tu maquina dev | Harness en /opt/ag-<proyecto>/ | `prod-agent.sh --scan` |
| ag-<proyecto> (harness) | VPS destino | Agente LLM autonomo | `./boot.sh` |

NO ejecutas `prod-agent.sh` en el VPS. NO ejecutas `boot.sh` en dev.

## Zero to Agent

```bash
# PASO 1: En tu maquina de desarrollo
cd /opt/nexus-one/prod-agent
./prod-agent.sh --scan /opt/mi-proyecto
# El harness se genera en /opt/ag-mi-proyecto/

# PASO 2: En el VPS destino
scp -r /opt/ag-mi-proyecto user@vps:~/
ssh user@vps './ag-mi-proyecto/boot.sh'
# boot.sh: verifica integridad, crea usuario, detecta recursos,
# presenta menu LOCAL/CLOUD, instala runtime, lanza agente.
```

## Documentación del Proyecto

| Archivo | Rol |
|---------|-----|
| docs/QUICKSTART.md | Guia practica paso a paso (principiantes) |
| docs/CONCEPT.md | Concepto: que ganas con un agente orquestador |
| docs/AGENTS.md | Fuente de verdad arquitectonica (unificada) |
| prod-agent/context/ROADMAP.md | Roadmap del generador |
| remediation/context/ROADMAP.md | Roadmap de remediation |
| prod-agent/docs/README.md | Documentación específica del generador |
| remediation/docs/README.md | Documentación específica de remediation |
| docs/TECHNICAL.md | Documentación técnica detallada |
| docs/CHANGELOG.md | Historial de versiones |

## Comandos Principales

```bash
# Generación de harness
./prod-agent.sh --scan /opt/mi-proyecto        # Generar (+ backup)
./prod-agent.sh --scan --dry-run /opt/proyecto # Preview sin escribir archivos
./prod-agent.sh --scan --force /opt/proyecto   # Sobrescribir sin preguntar
./prod-agent.sh --scan --update /opt/proyecto  # Solo config, preserva src/
./prod-agent.sh --validate /opt/ag-proyecto    # Validar harness
./prod-agent.sh --list                         # Listar proyectos

# Remediation
./remediation.sh --remediate /opt/proyecto     # Ciclo completo
./remediation.sh --remediate --dry-run /ruta   # Preview de hallazgos
./remediation.sh --list                        # Listar remedies
./remediation.sh --remedy nombre               # Ver detalle remedy
```

## Runtimes Soportados

| Runtime | RAM Min | Internet | API Key | Grupo |
|---------|---------|----------|---------|-------|
| Ollama Qwen2.5-coder:7b | 8 GB | Solo descarga | No | LOCAL |
| Ollama Qwen2.5-coder:14b | 16 GB | Solo descarga | No | LOCAL |
| Claude API | 2 GB | Si | Si | CLOUD |
| OpenCode | 4 GB | Si | No | CLOUD |
| Gemini API | 2 GB | Si | Si | CLOUD |

## Modelo de Seguridad

3 capas de defensa en profundidad. Ver `docs/AGENTS.md` (sección 3.8-3.11)
para detalle completo con tabla de enforcement y matriz de riesgos.

| Capa | Mecanismo |
|------|-----------|
| 1. Boot | Rechazar root, sudo -v, sha256sum -c integrity.sum |
| 2. Usuario | ag-<proyecto> sin sudo, sin shell, sin login |
| 3. Archivos | Config root:root, workspace ag-<proyecto> |

## Requerimientos

**Maquina de Desarrollo**: Bash 4+, rsync, sudo.
**VPS**: Ubuntu 24.04, Bash 4+, sudo (NO root, NO NOPASSWD), 2GB RAM min, 5GB disco.

## Creditos

**Desarrollo:** Angel Esquivel (CyberSecurity)
**Framework:** NEXUS ONE Harness Engineering v1.0.0
**Year:** 2026

#End Development By Angel Esquivel (CyberSecurity) [NEXUS ONE 2026]
