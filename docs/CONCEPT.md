# CONCEPT.md -- Agentes LLM Autonomos: Que Ganas con NEXUS ONE
# =================================================================
# Explica el valor de orquestar proyectos existentes con agentes
# LLM. Para principiantes y tecnicos que entienden que un proyecto
# funciona solo, pero no se gestiona solo.
# =================================================================

## 1. El Problema: Tu Proyecto Funciona, Pero No Se Gestiona Solo

Tienes DSToken corriendo en un VPS. 6 containers Docker (mysql,
privacyidea, redis, openldap, apache, freeradius), 8 scripts de
mantenimiento, health checks cada 15s, backups semanales, TLS certs
que expiran a los 365 dias, y 3 versiones de privacyIDEA con limitaciones
conocidos (DataError en 3.2.2, ERR904 en FreeRADIUS sin LDAP).

El proyecto arranca con `bootstrap.sh` y funciona. Pero:

- Quien revisa los logs cuando falla RADIUS a las 3am?
- Quien ejecuta `make_backup.sh` cada semana y verifica que el tgz
  no este corrupto?
- Quien detecta que MySQL se cayo por OOM y lo reinicia?
- Quien te avisa que el TLS cert de Apache expira en 15 dias?
- Quien responde "que version de privacyIDEA estamos corriendo?"
  cuando lo necesitas saber?

Tu proyecto funciona. Pero no se gestiona solo. Ahí entra un agente.

## 2. Que Aporta un Agente Orquestador

Un agente LLM no reemplaza tu proyecto. Lo envuelve en una capa de
gestion que ejecuta tus scripts, lee tus logs, y toma decisiones
dentro de limites que tu defines.

| Capacidad | Mecanismo | Ejemplo con DSToken |
|-----------|-----------|---------------------|
| Vigilancia | Loop ReAct ejecuta checks cada N segundos | Verifica que 6/6 containers esten healthy via docker compose ps |
| Auto-reparacion | Si check falla, ejecuta script de recovery | Container caido -> `docker compose up -d` + re-verifica |
| Automatización programada | Daemon mode procesa .agent/tasks/*.task | Backup semanal: `make_backup.sh` -> verifica tgz -> audit.log |
| Diagnostico bajo demanda | Lee logs del proyecto, parsea, responde | "Por que fallo RADIUS?" -> grep freeradius log -> respuesta |
| Notificaciones | Audit log JSON de toda accion | Cada restauro, cada error, cada decision queda registrada |
| Ejecución de herramientas | Corre scripts del proyecto via command dispatch | `analyze_custom.sh`, `validate_dstoken.sh`, `restore_custom.sh` |

El valor no es magia. Es que cada script que escribiste para DSToken
ahora tiene un orquestador que decide cuando ejecutarlo, verifica el
resultado, y actua en consecuencia.

## 3. Ejemplo Concreto: DSToken con Agente

### Dia 1 - Despliegue

```bash
# En tu maquina de desarrollo
./prod-agent.sh --scan /opt/dstoken2026
# Crea /opt/ag-dstoken2026/ con: agent.sh, boot.sh, detect.sh,
# limits.conf, context/, src/ (copia del proyecto)

scp -r /opt/ag-dstoken2026 user@vps:~
ssh user@vps './ag-dstoken2026/boot.sh'
# boot.sh: verifica integridad (sha256sum), crea usuario aislado,
#   detecta recursos (8GB RAM -> perfil medium),
#   menu runtime -> seleccionas Ollama,
#   instala runtime, lanza agente.
```

### Dia 7 - Backup Automático

```
03:00 AM  Daemon mode detecta .agent/tasks/backup.task
03:00 AM  Ejecuta scripts/make_backup.sh
03:01 AM  Verifica que el .tgz generado no este corrupto
03:01 AM  Registra en audit.log:
           {"ts":"...","action":"backup","file":"dstoken-backup-...tgz",
            "size_mb":47,"status":"ok"}
```

### Dia 14 - Container Caido

```
02:47 AM  Health check detecta mysql: unhealthy
02:47 AM  Ejecuta: docker compose up -d mysql
02:47 AM  Re-verifica health cada 5s
02:48 AM  mysql: healthy (2 ciclos consecutivos)
02:48 AM  Audit log:
           {"ts":"...","action":"recovery","service":"mysql",
            "downtime_sec":47,"status":"restored"}
```

### Dia 30 - Certificado TLS Proximo a Vencer

```
08:00 AM  Detecta: openssl x509 -checkend 1296000 en cert Apache
08:00 AM  Quedan 15 dias -> alerta en conversation loop
08:01 AM  Usuario: "renuevalo"
08:01 AM  Ejecuta: openssl req -new -x509 -days 365 -key ... -out ...
08:01 AM  Recarga Apache: docker compose exec apache httpd -k graceful
08:01 AM  Re-verifica: curl -k https://localhost/ -> 200 OK
```

### Bajo Demanda

```
Usuario:  "Que version de privacyIDEA estamos corriendo?"
Agente:   curl -s http://localhost:8000/ | grep -oP 'privacyIDEA \K[0-9.]+'
          -> "3.12.3, ultima version estable disponible."
```

## 4. Cuando Tiene Sentido vs Cuando No

| Escenario | Recomendación | Por que |
|-----------|---------------|---------|
| DSToken en producción, 6 containers, usuarios activos | SI | Monitoreo 24/7 + backup automático + auto-recovery en caidas |
| Script de migracion que se ejecuta una vez | NO | No hay estado que vigilar ni tareas recurrentes |
| Proyecto local de desarrollo donde trabajas activamente | NO | Tu ya estas ahi, el agente no suma valor |
| Servicio critico con SLA (VPN corporativa, 2FA) | SI | El agente detecta y repara en segundos, no en horas |
| Stack con mantenimiento documentado (backup, restore, validate) | SI | El agente ejecuta EXACTAMENTE esos scripts, no los reemplaza |
| Quieres aprender agentes LLM sin riesgo | SI | Sandbox con limites de seguridad: $10 budget, 4h session, sin sudo |
| Proyecto personal sin usuarios ni SLA | NO | El overhead del agente no se justifica |

## 5. Lo Que el Agente NO Hace

Tan importante como lo que hace es lo que NO puede hacer:

- **No modifica docker-compose.yml** ni archivos de configuración sin
  autorizacion explicita tuya en el conversation loop
- **No toca /etc/**, /root/.ssh/, /var/lib/docker/ ni /boot/
  (RESTRICTED_PATHS en limits.conf)
- **No ejecuta rm -rf**, dd, mkfs, reboot, shutdown, poweroff, halt
  (RESTRICT_COMMANDS en limits.conf)
- **No gasta mas de $10** sin cortar automaticamente (BUDGET_USD=10 en
  limits.conf, tracking en audit.log)
- **No corre para siempre**: session timeout de 4h, command timeout
  de 300s, idle timeout de 600s
- **No modifica sus propios limites**: .agent/limits.conf es root:root
  644, el agente corre como usuario sin sudo
- **No bypassa integrity.sum**: si modificas agent.sh, boot.sh o
  detect.sh, el checksum SHA256 falla y el agente no arranca
- **No interpreta comandos peligrosos**: -c/-e en python3, node, perl
  estan bloqueados por validate_command()

## 6. La Idea en Una Línea

El agente NEXUS ONE es como tener un DevOps engineer junior viviendo
dentro del VPS de DSToken. No es magico: solo ejecuta los scripts que
tu ya escribiste en `scripts/`, lee los logs que ya se generan en cada
container, y toma decisiones dentro de las reglas que definiste en
`limits.conf`. Pero a diferencia de un humano:

- No duerme: verifica 28800 health checks al dia si el loop es cada
  15s (24h * 60min * 60s / 15s = 5760 ciclos * 5 servicios monitoreados)
- No se equivoca de comando: ejecuta rutas fijas, no interpreta
  parcialmente
- No olvida: cada accion queda en audit.log con timestamp exacto
- No se salta las reglas: los limites estan en root:root, no puede
  modificarlos aunque el LLM lo intente

Si DSToken necesita backups semanales, health checks cada 15s, y
recuperacion automática ante caidas nocturnas, un agente LLM ejecuta
ese ciclo 24/7. Eso es el valor: **no reemplaza el conocimiento del
proyecto, lo escala operativamente**.

#End Development By Angel Esquivel (CyberSecurity) [NEXUS ONE 2026]
