# Fact Sheet — nexus-one

Valores críticos del proyecto. Fuente de verdad para agentes que necesitan
contexto rápido sin explorar el codebase.

**CONFIDENCIAL (REGLA ROJA)**: este archivo NO incluye credenciales VPS,
nombres de clientes, ni internos de remediación. Mantener anónimo.

## Identidad
- **Proyecto:** nexus-one
- **Tipo:** Harness Engineering Framework (generador de agentes LLM)
- **Stack:** Bash | Python 3.12+ | Shell scripting
- **Versión:** 0.1.0
- **Dueño:** Angelo Esquivel (CyberSecurity)
- **Repositorio:** aentrepreneur/nexus-one (portada pública)
- **Licencia:** Propietaria (EULA + CONTRACT)
- **Última actualización:** 2026-08-11

## Arquitectura (Two-Tier)
- **NIVEL 1 — El generador** (`/opt/nexus-one/`): `prod-agent/` + `remediation/`
- **NIVEL 2 — El generado** (`/opt/ag-<proyecto>/`): harness autocontenido para VPS
- **Entrypoints:** `prod-agent/prod-agent.sh`, `remediation/remediation.sh`
- **Heartbeat:** `heartbeat/heartbeat-check.sh` (licencia en VPS remoto)

## Lo que genera el framework (harness autocontenido)
- `boot.sh` — entry point remoto · `detect.sh` — fingerprinting de recursos
- `.agent/agent.sh` — loop ReAct · `.agent/agent.md` — identidad YAML
- `.agent/limits.conf` — restricciones de seguridad
- `src/` — código original · `integrity.sum` — SHA256 obligatorio

## Rutas críticas
- `/opt/nexus-one/PROMPT.md` — identidad, Two-Tier, REGLA ROJA, decisiones
- `/opt/nexus-one/docs/` — documentación (CONTRACT, EULA, README)
- `/opt/nexus-one/remediation/` — subsistema de remediación (NO exponer)
- `/opt/nexus-one/prod-agent/` — generador de agentes

## Restricciones de diseño
1. **REGLA ROJA**: nunca mencionar credenciales VPS, clientes, configs de seguridad de clientes, ni internos de remediación
2. Anonimizar siempre: "un VPS de producción", "un cliente corporativo", "herramientas de remediación"
3. Harnesses autocontenidos (sin dependencias del generador)
4. `integrity.sum` obligatorio en cada harness
5. Proyecto FUERA de auto-audit/issues del ecosistema (política REGLA ROJA, decisions.md 2026-08-11)

## Verificadores
- **Tests:** `bash prod-agent/tests/run_all.sh` (bats: test_generator, test_scanner, test_validator, test_backup)
- **Nota:** usa bats (no pytest) — oc-validate lo reporta "— sin tests" correctamente

## Dependencias clave
```
bash >= 5.0
ssh / sshpass (heartbeat remoto)
python3 >= 3.12 (scripts internos)
```

## Historial de cambios críticos
| Fecha | Cambio | Impacto |
|-------|--------|---------|
| 2026-07-19 | Creación PROMPT.md | Trazabilidad a TELOS |
| 2026-07-20 | batch8: skills → entregables facturables | Patrón monetizable |
| 2026-08-11 | FACT-SHEET creado (N4), política REGLA ROJA | Exclusión de auto-audit |

---
*Fuente de verdad: `/opt/nexus-one/PROMPT.md` + `docs/`. Generado desde `knowledge/FACT-SHEET-TEMPLATE.md`.*
