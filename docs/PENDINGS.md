# PENDINGS.md -- Mejoras de Proceso Aplicadas por Etapa
# =================================================================
# Registro de hallazgos de proceso identificados y mejorados
# durante el desarrollo del framework. Todos resueltos.
# =================================================================

## Mejoras de Proceso por Etapa

| # | Hallazgo | Etapa | Mejora Aplicada | Version |
|---|----------|-------|-----------------|---------|
| 1 | Path systemd no portable entre distros | D | install_service() parchea con sed en boot-time | v0.15.0 |
| 2 | Scripts con paths absolutos en tests | E | Usar SCRIPT_DIR (relativo) en vez de pwd | v0.15.0 |
| 3 | report_status.sh sin fallback sin python3 | G | Fallback a grep -c | v0.15.0 |
| 4 | check_auth_log.sh sin sugerencia de sudo | G | Mensaje actualizado: "Use sudo or adm group" | v0.15.0 |
| 5 | OpenCode --prompt flag obsoleto v1.15 | OC | Usar `opencode run` | v0.16.0 |
| 6 | Payload quoting con ''' fragil en shell | OLL | Usar PYTHON_* env vars en vez de embedding | v0.16.0 |
| 7 | stream:false vs stream:False en Python | OLL | Capitalizacion correcta: stream:False | v0.16.0 |

## Validación Remota (VPS <vps-hostname>, Ubuntu 24.04, 7GB RAM)

### Resultados por Fase

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

### Proveedores Validados

| Proveedor | Version | Modelo | Costo Prueba |
|-----------|---------|--------|--------------|
| OpenAI | gpt-4o-mini | Validado | ~$0.00001 |
| OpenCode | v1.15.11 | Validado | $0 |
| Ollama | v0.24.0 | llama3.2:1b | $0 |

### Estado General

**Version**: v1.0.0 | **Hallazgos**: 7/7 RESUELTOS | **Tests**: 40/40 PASS | **Pendientes**: 0

#End Development By Angel Esquivel (CyberSecurity) [NEXUS ONE 2026]
