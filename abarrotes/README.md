# Base de datos `abarrotes` (MySQL)

Copia en MySQL del esquema `abarrotes` creado originalmente en **Supabase
(PostgreSQL)**, preparada para abrirse y administrarse desde **MySQL Workbench**.

Contiene **6 tablas**, **5 claves foráneas**, **10 restricciones `CHECK`** y
**44 filas** de datos de ejemplo. Probada en **MySQL 8.0.46**.

> MySQL Workbench solo habla MySQL. Si quieres ver la base original de
> Supabase, usa el SQL Editor de Supabase o un cliente PostgreSQL (pgAdmin,
> DBeaver). Esta carpeta es una copia independiente.

## Contenido

| Archivo | Descripción |
|---|---|
| `abarrotes.sql` | Base completa: estructura, claves, restricciones y datos |
| `README.md` | Documentación, modelo y consultas de ejemplo |

## Requisitos

- MySQL **8.0.16 o superior** (para que se apliquen los `CHECK`).
- MySQL Workbench (opcional).

## Restaurar la base

El archivo incluye `CREATE DATABASE IF NOT EXISTS` y `INSERT IGNORE`, así que
se puede ejecutar varias veces sin duplicar datos.

### Desde MySQL Workbench

1. Conéctate a tu servidor MySQL.
2. **File → Open SQL Script…** y elige `abarrotes.sql`.
3. Pulsa **Execute SQL Script** (rayo con el mundo, o `Ctrl+Shift+Enter`).
4. Refresca el panel **SCHEMAS** (`F5`) y aparecerá `abarrotes`.

### Desde la terminal

```bash
# Servidor con contraseña
mysql -u tu_usuario -p < abarrotes.sql

# Instancia local del proyecto (puerto 3307)
mysql -h 127.0.0.1 -P 3307 -u lavado_admin -p123456 < abarrotes.sql
```

## Crear un usuario para Workbench

El script no crea usuarios. Si hace falta:

```sql
CREATE USER IF NOT EXISTS 'lavado_admin'@'localhost' IDENTIFIED BY '123456';
CREATE USER IF NOT EXISTS 'lavado_admin'@'127.0.0.1' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON abarrotes.* TO 'lavado_admin'@'localhost';
GRANT ALL PRIVILEGES ON abarrotes.* TO 'lavado_admin'@'127.0.0.1';
FLUSH PRIVILEGES;
```

Conexión en Workbench:

```text
Host:            127.0.0.1
Port:            3307
User:            lavado_admin
Password:        123456
Default Schema:  abarrotes
```

> `123456` es una contraseña solo para prácticas locales. No la uses en
> producción ni expongas el puerto de MySQL a Internet.

## Comprobar la carga

```sql
USE abarrotes;

SELECT 'categorias' AS tabla, COUNT(*) AS filas FROM categorias
UNION ALL SELECT 'proveedores',      COUNT(*) FROM proveedores
UNION ALL SELECT 'productos',        COUNT(*) FROM productos
UNION ALL SELECT 'clientes',         COUNT(*) FROM clientes
UNION ALL SELECT 'ventas',           COUNT(*) FROM ventas
UNION ALL SELECT 'detalle_ventas',   COUNT(*) FROM detalle_ventas
ORDER BY tabla;
```

Resultado esperado:

| tabla | filas |
|---|---:|
| categorias | 5 |
| clientes | 6 |
| detalle_ventas | 12 |
| productos | 10 |
| proveedores | 5 |
| ventas | 6 |

**Total: 44 filas.** Además, el total de cada venta debe ser igual a la suma de
su detalle:

```sql
SELECT COUNT(*) AS ventas_con_total_incorrecto
FROM ventas v
WHERE v.total <> (
  SELECT COALESCE(SUM(d.subtotal), 0)
  FROM detalle_ventas d
  WHERE d.venta_id = v.id
);
-- Debe devolver 0
```

## Modelo de datos

```text
categorias ──< productos >── proveedores
                  │
                  └──< detalle_ventas >── ventas >── clientes
```

| Tabla | Descripción |
|---|---|
| `categorias` | Bebidas, snacks, limpieza, alimentos básicos y cuidado personal |
| `proveedores` | Proveedores de la tienda |
| `productos` | Catálogo con costo, precio de venta e inventario |
| `clientes` | Clientes de la tienda |
| `ventas` | Ventas con método de pago y estado |
| `detalle_ventas` | Productos, cantidades y subtotales de cada venta |

### Reglas destacadas

- `precio_venta >= precio_costo` en todos los productos.
- `stock >= 0` y `stock_minimo >= 0`.
- `cantidad > 0` y `precio_unitario > 0` en el detalle.
- No se repite el mismo producto en una misma venta (`UNIQUE (venta_id, producto_id)`).
- `metodo_pago` y `estado` están controlados con `ENUM`.
- Borrados: `CASCADE` en el detalle de venta y `RESTRICT` en catálogos.

## Consultas de ejemplo

```sql
-- Ventas con cliente y número de productos
SELECT v.id,
       CONCAT(c.nombre, ' ', c.apellido) AS cliente,
       v.fecha,
       v.metodo_pago,
       v.total,
       COUNT(d.id) AS productos_distintos
FROM ventas v
JOIN clientes c ON c.id = v.cliente_id
LEFT JOIN detalle_ventas d ON d.venta_id = v.id
GROUP BY v.id, c.nombre, c.apellido
ORDER BY v.id;

-- Ingresos por método de pago
SELECT metodo_pago, COUNT(*) AS ventas, SUM(total) AS ingresos
FROM ventas
WHERE estado = 'completada'
GROUP BY metodo_pago
ORDER BY ingresos DESC;

-- Productos con stock bajo
SELECT p.codigo, p.nombre, p.stock, p.stock_minimo
FROM productos p
WHERE p.stock <= p.stock_minimo
ORDER BY p.stock;

-- Productos más vendidos
SELECT p.nombre, SUM(d.cantidad) AS unidades, SUM(d.subtotal) AS importe
FROM detalle_ventas d
JOIN productos p ON p.id = d.producto_id
GROUP BY p.id, p.nombre
ORDER BY unidades DESC;
```

## Diferencias respecto a la versión PostgreSQL de Supabase

| PostgreSQL (Supabase) | MySQL |
|---|---|
| `BIGINT GENERATED BY DEFAULT AS IDENTITY` | `BIGINT UNSIGNED AUTO_INCREMENT` |
| `TIMESTAMPTZ` | `DATETIME` |
| `BOOLEAN` | `TINYINT(1)` |
| `NUMERIC(10,2)` | `DECIMAL(10,2)` |
| `TEXT` + `CHECK IN (...)` | `ENUM(...)` |

Para regenerar este archivo desde tu instancia local:

```bash
mysqldump -h 127.0.0.1 -P 3307 -u lavado_admin -p \
  --single-transaction --no-tablespaces --set-gtid-purged=OFF \
  --databases abarrotes > abarrotes.sql
```
