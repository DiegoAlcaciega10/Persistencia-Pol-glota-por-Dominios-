# Distribuidora Valle Express - Rescate Políglota

Taller **Persistencia Políglota - Operación Rescate**
Asignatura: Sistemas Avanzados de Base de Datos - BD Distribuidas NoSQL y Réplica
Universidad de las Fuerzas Armadas ESPE · Grupo N.º 4 · NRC 36587 · Docente: Paul Diaz

## 1. Contexto

Distribuidora Valle Express (Sangolquí) corría todo en una sola instancia MySQL 5.7 con una tabla gigante `ventas_todo` (47 columnas, JSON de productos). El Viernes Negro 2025 una consulta `LIKE '%TV%'` bloqueó la tabla 45 s, las cajas dejaron de facturar y el disco se llenó sin réplica: **~$18,000 perdidos**.

Este repositorio contiene la nueva arquitectura **políglota, distribuida y con réplica** y las pruebas que demuestran que sobrevive a fallos.

## 2. Arquitectura

![Arquitectura](docs/img/arquitectura.png)

| Dato | Motor | Teoría | Réplica |
|---|---|---|---|
| Facturas, pagos, stock contable | **PostgreSQL 16** | ACID · CP · PACELC PC/EC | Streaming replication: 1 réplica síncrona (RPO=0) + 1 asíncrona. Failover manual con `pg_promote()` (Patroni como propuesta). |
| Catálogo, clientes, logs de pedidos | **MongoDB 7** | BASE ajustable · CP · PACELC PC/EC | Replica Set de 3 nodos, `w:"majority"` en pedidos, elección automática (~10 s). |
| Carrito, sesiones, caché de stock | **Redis 7** | BASE · AP · PACELC PA/EL | 1 master + 2 réplicas asíncronas + 3 Sentinel, AOF activo. |
| Búsqueda de texto, reportes | **Elasticsearch** | BASE · AP · PACELC PA/EL | *Propuesta de diseño, no incluida en el laboratorio.* |

**Principio:** se elige el nivel de consistencia por tipo de dato, no uno solo para todo. Lo que no se puede perder (dinero) es CP; lo que debe ser rápido y es reconstruible (carrito, caché) es AP.

## 3. Requisitos

- Docker y Docker Compose v2
- ~4 GB de RAM libres
- Puertos libres: 5432, 5050, 27017, 8081, 6379, 8082

## 4. Cómo levantar el laboratorio

```bash
# 1. Permitir replicación en el Postgres primario
docker compose up -d postgres
docker exec distribuidora_postgres sh -c 'echo "host replication all all scram-sha-256" >> /var/lib/postgresql/data/pg_hba.conf'
docker exec distribuidora_postgres psql -U admin -d distribuidora_acid -c "SELECT pg_reload_conf();"

# 2. Levantar todo (compose base + réplicas)
docker compose -f docker-compose.yml -f docker-compose.replicas.yml up -d
docker compose -f docker-compose.yml -f docker-compose.replicas.yml ps

# 3. Mongo: pasar de 1 a 3 nodos
docker exec distribuidora_mongo mongosh --quiet --eval 'rs.add("distribuidora_mongo2:27017"); rs.add("distribuidora_mongo3:27017")'
docker exec distribuidora_mongo mongosh --quiet --eval 'rs.status().members.map(m => m.name+" -> "+m.stateStr)'

# 4. Postgres: réplica síncrona
docker exec distribuidora_postgres psql -U admin -d distribuidora_acid -c "ALTER SYSTEM SET synchronous_standby_names = 'replica_sync';" -c "SELECT pg_reload_conf();"
docker exec distribuidora_postgres psql -U admin -d distribuidora_acid -c "SELECT application_name, state, sync_state FROM pg_stat_replication;"
```

Paneles web: pgAdmin `http://localhost:5050` · mongo-express `http://localhost:8081` · Redis Commander `http://localhost:8082`

> Las credenciales del laboratorio están en el compose solo con fines didácticos. En un entorno real deben ir en un `.env` fuera del repositorio.

## 5. Una venta distribuida

```bash
# 1) Factura en PostgreSQL (ACID)
docker exec -it distribuidora_postgres psql -U admin -d distribuidora_acid -c "
CREATE TABLE IF NOT EXISTS facturas (
  id SERIAL PRIMARY KEY, cliente_id TEXT NOT NULL,
  total NUMERIC(10,2) NOT NULL, estado TEXT DEFAULT 'PAGADA', fecha TIMESTAMP DEFAULT now());"
ID=$(docker exec distribuidora_postgres psql -U admin -d distribuidora_acid -q -t -A -c "INSERT INTO facturas (cliente_id,total) VALUES ('CLI-001',899.99) RETURNING id;")

# 2) Detalle en MongoDB (w: majority)
docker exec distribuidora_mongo mongosh distribuidora --quiet --eval "db.detalle_pedidos.insertOne({factura_id:$ID,cliente_id:'CLI-001',items:[{sku:'TV-55',nombre:'Smart TV 55',pulgadas:55,cantidad:1,precio:899.99}]},{writeConcern:{w:'majority'}})"

# 3) Vaciar carrito en Redis
docker exec distribuidora_redis redis-cli -a Admin123 DEL carrito:CLI-001
```

## 6. Evidencias

| Evidencia | Captura |
|---|---|
| `rs.status()` con 3 miembros | ![rs.status](docs/img/rs_status.png) |
| `SELECT` en PostgreSQL | ![postgres](docs/img/postgres_select.png) |
| `GET` en Redis | ![redis](docs/img/redis_cmds.png) |
| Réplicas Postgres (`pg_stat_replication`) | ![pg-repl](docs/img/pg_replication.png) |

## 7. Prueba de supervivencia (Plan de Caos)

| Escenario | Qué se cayó | Qué siguió funcionando | Qué se perdió | Justificación CAP |
|---|---|---|---|---|
| a) `docker stop distribuidora_postgres` | Escrituras de facturas | Lecturas desde réplica, catálogo (Mongo), carritos (Redis) | Nada confirmado | **CP**: se rechazan escrituras antes que emitir facturas inconsistentes. |
| b) `docker stop distribuidora_mongo` | Primario de Mongo (unos segundos) | Facturación, carrito, y Mongo tras elegir nuevo primario | Nada con `w:majority` | **CP con failover**: la mayoría (2/3) elige primario. Con 1/3 nodos no hay primario. |
| c) `docker stop distribuidora_redis` | Master de Redis | Facturación, catálogo; Sentinel promueve una réplica | Como máximo ~1 s de escrituras (AOF/replicación asíncrona) | **AP/EL**: Redis es caché; la verdad está en PostgreSQL. |

> Completar con las salidas reales y capturas de cada prueba (carpeta `docs/img/`).

Escenarios adicionales del Ingeniero de Caos:

1. Parar `distribuidora_pg_sync` y hacer un `INSERT`: la escritura queda en espera hasta que vuelve la réplica síncrona (costo de PC/EC).
2. Partición de red: `docker network disconnect distribuidora_net distribuidora_mongo3` y reconectar.
3. Matar 2 de 3 Sentinel: sin quórum no hay failover automático de Redis.

## 8. Alcance y limitaciones

- Failover de PostgreSQL **manual** (`pg_promote()`); Patroni queda como mejora propuesta.
- Elasticsearch es diseño propuesto, no está desplegado.
- Todos los nodos corren en un solo host Docker: se simulan fallos de proceso y de red, no de hardware. En producción deben ir en servidores o zonas distintas.

## 9. Video de defensa

[Ver video (Loom, 3 min)](PEGAR_ENLACE_AQUI)

## 10. Equipo

| Integrante | Rol |
|---|---|
| Diego Alcaciega | _(Arquitecto/a CAP, DBA SQL, DBA NoSQL o Ingeniero/a de Caos)_ |
| Jean Pierre Peñaranda | _(asignar)_ |
| _(agregar)_ | _(asignar)_ |

## 11. Estructura del repositorio

```
.
├── docker-compose.yml
├── docker-compose.replicas.yml
├── README.md
└── docs/
    ├── Taller_Poliglota_Grupo4.pdf
    └── img/
```
