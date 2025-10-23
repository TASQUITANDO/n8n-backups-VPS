# 🧭 Sistema de Monitorización y Observabilidad del VPS Nubia
### Stack: Grafana + Prometheus + Uptime Kuma + Caddy + n8n  
**Autor:** Alberto Tascón Fresnadillo (INCIBE)  
**Fecha:** Octubre 2025  

---

## 1. 🌐 Introducción y visión general

Este documento describe la arquitectura y configuración de un sistema completo de **monitorización y observabilidad** desplegado sobre una **VPS KVM2 de Hostinger** (6 €/mes) para el entorno `nubia.site`.

El objetivo del sistema es centralizar la **supervisión, métricas y seguridad** de los servicios desplegados (n8n, Caddy, Redis, PostgreSQL, Prometheus, Grafana y Uptime Kuma) bajo una infraestructura segura, automatizada y totalmente autoalojada.

---

## 2. 🧩 Arquitectura general del sistema

### Componentes principales
| Servicio | Rol principal | Puerto interno | Acceso externo |
|-----------|----------------|----------------|----------------|
| **Caddy** | Reverse proxy + TLS Let’s Encrypt | 80 / 443 | `*.nubia.site` |
| **n8n** | Automatización y workflows IA | 5678 | `https://n8n.nubia.site` |
| **Uptime Kuma** | Monitorización de servicios y pings Push | 3001 | `https://dash.nubia.site` |
| **Prometheus** | Recolección de métricas | 9090 | `https://prom.nubia.site` |
| **Grafana** | Visualización de métricas y dashboards | 3000 | `https://grafana.nubia.site` |
| **Node Exporter** | Métricas de sistema | 9100 | Interno |
| **cAdvisor** | Métricas de contenedores Docker | 8080 | Interno |
| **PostgreSQL** | Base de datos de n8n | 5432 | Interno |
| **Redis** | Cola de jobs n8n | 6379 | Interno |

---

## 3. ⚙️ Infraestructura Docker

La estructura principal se encuentra bajo `/opt/`, organizada por proyectos:

```bash
/opt
├── caddy              → Proxy inverso y certificados TLS
├── n8n                → Automatización de workflows
├── uptime-kuma        → Monitorización de servicios
├── metrics            → Prometheus, Grafana, cAdvisor y Node Exporter
├── monit              → Scripts de estado y cron jobs
└── containerd         → Dependencias del sistema
Redes Docker activas
docker network ls
# → n8n_default, web, metrics_metrics


La red web es la capa común de exposición HTTPS para todos los servicios detrás de Caddy.

4. 📊 Sistema de monitorización y métricas
4.1 Prometheus + cAdvisor + Node Exporter

El stack de métricas se levanta con docker-compose en /opt/metrics/:

services:
  prometheus:
    image: prom/prometheus:latest
    ports: ["127.0.0.1:9090:9090"]
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    networks: [web, metrics]

  node-exporter:
    image: prom/node-exporter
    ports: ["127.0.0.1:9100:9100"]
    networks: [metrics]

  cadvisor:
    image: gcr.io/cadvisor/cadvisor
    ports: ["127.0.0.1:8080:8080"]
    volumes:
      - /:/rootfs:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks: [metrics]

  grafana:
    image: grafana/grafana:latest
    ports: ["127.0.0.1:3000:3000"]
    volumes:
      - ./grafana:/var/lib/grafana
    networks: [web, metrics]

networks:
  web:
    external: true
  metrics:
    driver: bridge
volumes:
  prometheus_data:

4.2 Configuración de Prometheus (/opt/metrics/prometheus/prometheus.yml)
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: "node"
    static_configs:
      - targets: ["node-exporter:9100"]
  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]

4.3 Grafana

Acceso: https://grafana.nubia.site
Datasources:

Prometheus → http://prometheus:9090
Dashboards importados:

Docker Monitoring (cAdvisor + Node Exporter)

System Metrics Overview

Network & CPU Analysis

5. 💡 Supervisión y automatización (Uptime Kuma)
5.1 Automatización de comprobaciones

Ubicación: /opt/monit/

Scripts principales:

Script	Función	Frecuencia
sysinfo.sh	Estado de CPU, RAM, disco, carga	Cada 5 min
docker-health.sh	Estado de contenedores activos	Cada 5 min
kuma-push-all.sh	Envío PUSH a Uptime Kuma	Cada 5 min

Cada script envía datos vía API Push:

curl -fsS "https://dash.nubia.site/api/push/<TOKEN>?status=up&msg=OK"

5.2 Crontab con bloqueo flock
# ── Uptime Kuma PUSH (cada 5 min) ───────────────────────────
*/5 * * * * flock -n /tmp/sysinfo.lock        /opt/monit/sysinfo.sh         >/dev/null 2>&1
*/5 * * * * flock -n /tmp/docker-health.lock  /opt/monit/docker-health.sh   >/dev/null 2>&1
*/5 * * * * flock -n /tmp/kuma-master.lock    /opt/monit/kuma-push-all.sh   >/dev/null 2>&1

# ── Backup semanal de la DB de Kuma ─────────────────────────
0 3 * * 0 docker exec uptime-kuma sqlite3 /app/data/kuma.db ".backup '/app/data/kuma-$(date +\%F).db'"

6. 🔐 Seguridad y HTTPS
6.1 Firewall (UFW)
sudo ufw status
# 22/tcp (solo SSH), 80/tcp, 443/tcp permitidos

6.2 Fail2Ban y auto-updates
sudo apt install fail2ban unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades

6.3 Caddy + TLS Let’s Encrypt

Caddyfile simplificado:

n8n.nubia.site {
  encode zstd gzip
  basic_auth /* {
    tasqui $2a$14$gtNDTUF16kQ4Fq0zcp1g9eBC99zD6bCgzVnvInOlsUbiGATGSNlda
  }
  reverse_proxy n8n:5678
}

dash.nubia.site {
  encode zstd gzip
  @public_api path /api/push* /api/status-page* /status-page*
  handle @public_api {
    reverse_proxy uptime-kuma:3001
  }
  basic_auth /* {
    tasqui $2a$14$gtNDTUF16kQ4Fq0zcp1g9eBC99zD6bCgzVnvInOlsUbiGATGSNlda
  }
  reverse_proxy uptime-kuma:3001
}


Caddy genera automáticamente los certificados de:

n8n.nubia.site

dash.nubia.site

grafana.nubia.site

prom.nubia.site

7. 🧠 Buenas prácticas y roadmap futuro

🔄 Backup automático de Prometheus y Grafana.

🧩 Integración con n8n para recibir alertas de Kuma vía webhook.

📈 Alertmanager para thresholds críticos.

🧰 Fail2Ban extendido con notificaciones Telegram.

🧱 Logs centralizados (Loki + Promtail + Grafana Loki).

🕵️ Auditoría SSH con claves RSA y login MFA.

8. 📚 Anexo técnico
Estructura /opt
/opt
├── caddy/
│   ├── Caddyfile
│   └── docker-compose.yml
├── n8n/
│   ├── docker-compose.yml
│   ├── data/
│   ├── postgres/
│   └── redis/
├── uptime-kuma/
│   └── docker-compose.yml
├── metrics/
│   ├── prometheus/
│   ├── grafana/
│   └── docker-compose.yml
├── monit/
│   ├── sysinfo.sh
│   ├── docker-health.sh
│   └── kuma-push-all.sh

✅ Estado actual del sistema
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
# N8N, Kuma, Prometheus, Grafana → UP
# Node Exporter y cAdvisor → UP (healthy)
# TLS activo en todos los subdominios
# Monitores OK salvo Docker Salud (ajuste de token pendiente)


💬 Resumen:
El sistema del VPS Nubia logra una infraestructura segura, observada y autosuficiente,
combinando automatización (n8n), proxy seguro (Caddy), monitorización (Kuma) y métricas profesionales (Grafana + Prometheus).
Todos los procesos se autogestionan con cron jobs y están aislados en redes Docker internas.
Representa una base sólida para desplegar pipelines DevSecOps, agentes IA o integraciones de negocio baj
