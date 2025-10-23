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

