# Base de datos `lavado_autos` (MySQL)

Base de datos para un negocio de lavado de autos: clientes, vehículos, servicios,
empleados, citas, órdenes, pagos, insumos y promociones.

Contiene **12 tablas**, **13 claves foráneas**, **19 restricciones `CHECK`**,
**34 índices** y **144 filas** de datos de ejemplo. Probada en **MySQL 8.0.46**.

## Contenido del repositorio

| Archivo | Descripción |
|---|---|
| `lavado_autos.sql` | Volcado completo con estructura y datos (`mysqldump`) |
| `README.md` | Documentación, modelo y consultas de ejemplo |

## Requisitos

- **MySQL 8.0.16 o superior** (las restricciones `CHECK` solo se aplican desde
  esa versión).
- MySQL Workbench (opcional, para administrarla gráficamente).

## Restaurar la base

El archivo incluye `CREATE DATABASE IF NOT EXISTS`, `DROP TABLE IF EXISTS` y
`INSERT` con los datos. Se puede ejecutar varias veces y deja la base siempre
en el mismo estado.

> **Atención:** al ejecutarlo se **borran y recrean** las tablas de
> `lavado_autos`. No lo ejecutes sobre una base con datos que quieras conservar.

### Desde MySQL Workbench

1. Abre Workbench y conéctate a tu servidor.
2. Menú **File → Open SQL Script…** y elige `lavado_autos.sql`.
3. Pulsa **Execute SQL Script** (el rayo con el mundo, o `Ctrl+Shift+Enter`).
4. En el panel **SCHEMAS** pulsa refrescar (`F5`).
5. Aparecerá la base **`lavado_autos`** con sus 12 tablas.

### Desde la terminal

```bash
# Como root del sistema (Ubuntu suele usar auth_socket para root)
sudo mysql < lavado_autos.sql

# O con un usuario que tenga permisos
mysql -h 127.0.0.1 -P 3306 -u tu_usuario -p < lavado_autos.sql
```

## Usuario de ejemplo para desarrollo local

El script no crea usuarios. Si quieres uno solo para esta base (por ejemplo,
para MySQL Workbench), ejecuta como administrador:

```sql
CREATE USER IF NOT EXISTS 'lavado_admin'@'localhost' IDENTIFIED BY '123456';
CREATE USER IF NOT EXISTS 'lavado_admin'@'127.0.0.1' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON lavado_autos.* TO 'lavado_admin'@'localhost';
GRANT ALL PRIVILEGES ON lavado_autos.* TO 'lavado_admin'@'127.0.0.1';
FLUSH PRIVILEGES;
```

Datos de conexión:

```text
Host:            127.0.0.1
Port:            3306
User:            lavado_admin
Password:        123456
Default Schema:  lavado_autos
```

> `123456` es una contraseña solo para prácticas locales. No la uses en
> producción ni expongas el puerto de MySQL a Internet.

## Comprobar que la restauración quedó bien

```sql
USE lavado_autos;

SELECT 'clientes' AS tabla, COUNT(*) AS filas FROM clientes
UNION ALL SELECT 'vehiculos', COUNT(*) FROM vehiculos
UNION ALL SELECT 'servicios', COUNT(*) FROM servicios
UNION ALL SELECT 'empleados', COUNT(*) FROM empleados
UNION ALL SELECT 'insumos', COUNT(*) FROM insumos
UNION ALL SELECT 'promociones', COUNT(*) FROM promociones
UNION ALL SELECT 'citas', COUNT(*) FROM citas
UNION ALL SELECT 'ordenes', COUNT(*) FROM ordenes
UNION ALL SELECT 'orden_detalle', COUNT(*) FROM orden_detalle
UNION ALL SELECT 'pagos', COUNT(*) FROM pagos
UNION ALL SELECT 'consumo_insumos', COUNT(*) FROM consumo_insumos
UNION ALL SELECT 'orden_promociones', COUNT(*) FROM orden_promociones
ORDER BY tabla;
```

Resultado esperado:

| tabla | filas |
|---|---:|
| citas | 12 |
| clientes | 12 |
| consumo_insumos | 14 |
| empleados | 10 |
| insumos | 10 |
| orden_detalle | 20 |
| orden_promociones | 10 |
| ordenes | 12 |
| pagos | 12 |
| promociones | 10 |
| servicios | 10 |
| vehiculos | 12 |

**Total: 144 filas.**

## Modelo de datos

```text
clientes ──< vehiculos ──< citas
    │            │
    │            └──< ordenes >── empleados
    │                    │
    │                    ├──< orden_detalle >── servicios
    │                    ├──< pagos
    │                    ├──< consumo_insumos >── insumos
    │                    └──< orden_promociones >── promociones
    └────────────────────┘
```

### Tablas

| Tabla | Descripción |
|---|---|
| `clientes` | Clientes del negocio |
| `vehiculos` | Vehículos registrados por cliente |
| `servicios` | Catálogo de servicios de lavado |
| `empleados` | Personal del negocio |
| `insumos` | Inventario de insumos |
| `promociones` | Promociones y descuentos |
| `citas` | Citas agendadas |
| `ordenes` | Órdenes de servicio |
| `orden_detalle` | Servicios incluidos en cada orden |
| `pagos` | Pagos asociados a las órdenes |
| `consumo_insumos` | Insumos utilizados por orden |
| `orden_promociones` | Promociones aplicadas a cada orden |

### Reglas destacadas

- `ordenes.total <= subtotal` y en los datos `total = subtotal - descuento`.
- Un pago con `estado = 'aprobado'` exige `fecha_pago`.
- `orden_detalle` no repite el mismo servicio en una orden.
- `consumo_insumos` no repite el mismo insumo en una orden.
- Borrados: `CASCADE` en las dependencias de la orden y `RESTRICT` en catálogos.
- Estados y tipos controlados con `ENUM` además de `CHECK`.

## Consultas de ejemplo

```sql
-- Órdenes con cliente y vehículo
SELECT o.id,
       CONCAT(c.nombre, ' ', c.apellido) AS cliente,
       v.placa,
       CONCAT(v.marca, ' ', v.modelo) AS vehiculo,
       o.estado, o.subtotal, o.descuento, o.total
FROM ordenes o
JOIN clientes  c ON c.id = o.cliente_id
JOIN vehiculos v ON v.id = o.vehiculo_id
ORDER BY o.id;

-- Ingresos aprobados por método de pago
SELECT metodo, COUNT(*) AS pagos, SUM(monto) AS total
FROM pagos
WHERE estado = 'aprobado'
GROUP BY metodo
ORDER BY total DESC;

-- Servicios más vendidos
SELECT s.nombre, SUM(d.cantidad) AS veces, SUM(d.subtotal) AS importe
FROM orden_detalle d
JOIN servicios s ON s.id = d.servicio_id
GROUP BY s.nombre
ORDER BY importe DESC;

-- Ocupación por empleado
SELECT CONCAT(e.nombre, ' ', e.apellido) AS empleado,
       COUNT(o.id) AS ordenes,
       SUM(o.total) AS facturado
FROM empleados e
LEFT JOIN ordenes o ON o.empleado_id = e.id
GROUP BY e.id, empleado
ORDER BY facturado DESC;
```

## Regenerar este volcado

Si haces cambios en la base y quieres actualizar `lavado_autos.sql`:

```bash
mysqldump -h 127.0.0.1 -P 3306 -u lavado_admin -p \
  --single-transaction --no-tablespaces --set-gtid-purged=OFF \
  --routines --triggers --events \
  --databases lavado_autos > lavado_autos.sql
```
