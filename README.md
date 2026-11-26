# Dojo: monitoreo del EC2 de FleetOps Mantenimiento

Prometheus y Grafana corren en tu máquina local y consultan el EC2 (modelo pull). En el EC2 corre
un solo compose (`ec2/docker-compose.yml`) con los exportadores y el stack de logs (Loki + Alloy).

| Señal | Origen en el EC2 | Puerto | Fuente en Grafana |
|---|---|---|---|
| Sistema operativo | Node Exporter | 19100 | Prometheus (`ec2-node`) |
| Contenedores Docker | cAdvisor | 18080 | Prometheus (`ec2-docker`) |
| API, JVM, base de datos, MS Vehículos y health | Actuator del microservicio | 18001 | Prometheus (`maintenance-api`) |
| Logs de los contenedores | Loki (Alloy los recolecta) | 13100 | Loki |

Links directos:

- Node Exporter: `http://16.58.217.207:19100/metrics`
- cAdvisor: `http://16.58.217.207:18080/metrics`
- Actuator (métricas): `http://16.58.217.207:18001/actuator/prometheus`
- Actuator (health con detalle): `http://16.58.217.207:18001/actuator/health`

## Modo local (todo en tu máquina, un solo comando)

Levanta el microservicio (desde `../grupo_03_mantenimiento`) con PostgreSQL local, un mock de
MS Vehículos, el stack de `ec2/` (Node Exporter, cAdvisor, Loki + Alloy), Prometheus y Grafana con
el dashboard ya cargado. No usa el `.env` del servicio, la base real ni el EC2.

Usa los mismos puertos que el modo producción (19090, 13000, 19100, 18080, 13100): apagá primero
el Prometheus/Grafana de producción (`docker compose down` en esta carpeta).

```bash
docker compose -f docker-compose.local.yml up -d --build
```

Abrí `http://localhost:13000`: entra sin login y abre directo el dashboard **FleetOps · Mantenimiento**.

| URL | Qué es |
|---|---|
| `http://localhost:13000` | Grafana (dashboard, Explore con Loki) |
| `http://localhost:19090/targets` | Prometheus: `maintenance-api` debe estar UP |
| `http://localhost:18001/actuator/health` | Health del servicio con detalle |
| `http://localhost:8083/swagger-ui/index.html` | API del servicio |

En modo local, los paneles "EC2" (CPU, RAM, disco, contenedores) muestran tu máquina y tus
contenedores, leídos por los exportadores locales.

Para provocar fallas y ver cómo reacciona el dashboard:

```bash
docker compose -f docker-compose.local.yml stop vehicles-mock   # "vehicles" pasa a DOWN (~45 s)
docker compose -f docker-compose.local.yml start vehicles-mock
docker compose -f docker-compose.local.yml stop postgres        # "db" pasa a DOWN y aparecen errores en los logs
docker compose -f docker-compose.local.yml start postgres
```

Los paneles de API usan `rate()`: necesitan tráfico sostenido (al menos un par de scrapes de 15 s)
para mostrar datos. Si editás el JSON del dashboard, reiniciá Grafana para que lo tome:
`docker compose -f docker-compose.local.yml restart grafana`.

Para apagar todo (`-v` borra también la base local y los logs guardados):

```bash
docker compose -f docker-compose.local.yml down -v
```

## 1. En el EC2: exportadores + stack de logs

Copiá la carpeta `ec2/` al servidor:

```bash
scp -i clave.pem -r ec2 ec2-user@16.58.217.207:~/observability
```

Si Node Exporter y cAdvisor ya corrían desde otro compose, bajalos primero: este compose usa los
mismos `container_name` y Docker no deja crear dos contenedores con el mismo nombre.

```bash
ssh -i clave.pem ec2-user@16.58.217.207 'docker rm -f node-exporter cadvisor'
```

Levantá todo:

```bash
ssh -i clave.pem ec2-user@16.58.217.207 'cd ~/observability && docker compose up -d && docker ps'
```

Node Exporter y cAdvisor quedan caídos solo los segundos entre el `rm -f` y el `up -d`.

Alloy descubre todos los contenedores de Docker y envía sus logs a Loki. A los del microservicio
(servicio `maintenance`) les extrae el nivel (`level`) del JSON en formato ECS.

## 2. Security Group del EC2

Abrí **solo para tu IP** (`<tu-ip>/32`), nunca para `0.0.0.0/0`:

| Puerto | Uso |
|---|---|
| 19100 | Node Exporter |
| 18080 | cAdvisor |
| 18001 | Actuator del microservicio (health con detalle y métricas) |
| 13100 | Loki (no tiene autenticación) |

## 3. En tu máquina: Prometheus y Grafana

```bash
docker compose up -d
```

- Prometheus: `http://localhost:19090` → en *Status > Targets* los tres jobs deben estar `UP`.
- Grafana: `http://localhost:13000` → dashboard **FleetOps · Mantenimiento** (carpeta FleetOps).
  Las fuentes de datos Prometheus y Loki se cargan solas desde `grafana/provisioning`.

## Dashboards incluidos

Los dos se cargan solos al iniciar Grafana (carpeta **FleetOps**), sin importarlos a mano:

| Dashboard | Archivo | Qué muestra |
|---|---|---|
| FleetOps · Mantenimiento | `grafana/dashboards/fleetops-maintenance.json` | Servicio: salud, API, base de datos, MS Vehículos, JVM y logs |
| Node Exporter Full (ID 1860) | `grafana/dashboards/node-exporter-full.json` | Servidor en detalle: CPU, memoria, disco, red, sistema de archivos |

El 1860 es el dashboard de la comunidad que la guía del dojo importa a mano (paso 6.6). Se guardó tal
cual desde `https://grafana.com/api/dashboards/1860/revisions/latest/download` (revisión 105). Para
actualizarlo, se vuelve a descargar ese archivo y se reinicia Grafana.

## Qué muestra el dashboard FleetOps · Mantenimiento

- **Estado general:** si la API responde, salud por componente (`db`, `vehicles`, `diskSpace`)
  y estado del Circuit Breaker hacia MS Vehículos.
- **API:** peticiones por segundo, errores 4xx/5xx y latencia p95 por endpoint.
- **Base de datos:** conexiones activas, libres y en espera del pool, tiempo de adquisición y timeouts.
- **MS Vehículos:** llamadas por resultado y tasa de fallos del Circuit Breaker.
- **JVM y EC2:** heap, CPU, RAM, disco y memoria por contenedor.
- **Logs:** volumen por nivel y visor filtrable por nivel.

Consultas útiles en *Explore > Loki*:

```logql
{service="maintenance", level="ERROR"}
{service="maintenance"} | json | log_logger =~ ".*VehicleHttpAdapter.*"
```
