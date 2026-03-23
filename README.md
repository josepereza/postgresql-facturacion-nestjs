PostgreSQL y NestJS: Sistema de Facturación de Productos Informáticos 

 

1. Diseño de la base de datos 

Tablas 

Categorias 

id_categoria (PK) 

nombre 

descripcion 

Proveedores 

id_proveedor (PK) 

nombre 

contacto 

telefono 

email 

Articulos 

id_articulo (PK) 

nombre 

descripcion 

precio 

stock 

id_categoria (FK) 

id_proveedor (FK) 

Clientes 

id_cliente (PK) 

nombre 

direccion 

telefono 

email 

Pedidos 

id_pedido (PK) 

id_cliente (FK) 

fecha_pedido 

estado 

total 

Lineas_Pedido 

id_linea (PK) 

id_pedido (FK) 

id_articulo (FK) 

cantidad 

precio_unitario 

subtotal 

Facturas 

id_factura (PK) 

fecha_factura 

total 

Facturas_Pedidos 

id_factura (FK) 

id_pedido (FK) 

 

Creación de la base de datos 

CREATE DATABASE facturacion_informatica; 

Conectarse a ella y ejecutar todo lo siguiente. 

 

3️⃣ Tablas maestras 

Categorías 

CREATE TABLE categorias ( 

    id_categoria SERIAL PRIMARY KEY, 

    nombre VARCHAR(100) NOT NULL, 

    descripcion TEXT 

); 

 

Proveedores 

CREATE TABLE proveedores ( 

    id_proveedor SERIAL PRIMARY KEY, 

    nombre VARCHAR(150) NOT NULL, 

    contacto VARCHAR(100), 

    telefono VARCHAR(30), 

    email VARCHAR(100) 

); 

 

Artículos (control de stock incluido) 

CREATE TABLE articulos ( 

    id_articulo SERIAL PRIMARY KEY, 

    nombre VARCHAR(150) NOT NULL, 

    descripcion TEXT, 

    precio NUMERIC(10,2) NOT NULL CHECK (precio >= 0), 

    stock INT NOT NULL CHECK (stock >= 0), 

    id_categoria INT REFERENCES categorias(id_categoria), 

    id_proveedor INT REFERENCES proveedores(id_proveedor) 

); 

 

Clientes 

CREATE TABLE clientes ( 

    id_cliente SERIAL PRIMARY KEY, 

    nombre VARCHAR(150) NOT NULL, 

    direccion TEXT, 

    telefono VARCHAR(30), 

    email VARCHAR(100) 

); 

 

4️⃣ Pedidos y líneas de pedido 

Pedidos 

CREATE TABLE pedidos ( 

    id_pedido SERIAL PRIMARY KEY, 

    id_cliente INT REFERENCES clientes(id_cliente), 

    fecha_pedido TIMESTAMP DEFAULT CURRENT_TIMESTAMP, 

    estado VARCHAR(20) NOT NULL DEFAULT 'ABIERTO', 

    total NUMERIC(10,2) DEFAULT 0 

); 

📌 Estados posibles: 

ABIERTO 

FACTURADO 

 

Líneas de pedido 

CREATE TABLE lineas_pedido ( 

    id_linea SERIAL PRIMARY KEY, 

    id_pedido INT REFERENCES pedidos(id_pedido) ON DELETE CASCADE, 

    id_articulo INT REFERENCES articulos(id_articulo), 

    cantidad INT NOT NULL CHECK (cantidad > 0), 

    precio_unitario NUMERIC(10,2) NOT NULL, 

    subtotal NUMERIC(10,2) NOT NULL 

); 

5️⃣ Facturas 

CREATE TABLE facturas ( 

    id_factura SERIAL PRIMARY KEY, 

    id_pedido INT UNIQUE REFERENCES pedidos(id_pedido), 

    fecha_factura TIMESTAMP DEFAULT CURRENT_TIMESTAMP, 

    total NUMERIC(10,2) NOT NULL 

); 

2. Diagrama Entidad-Relación 

Entidades y relaciones 

categorias → 1 a N → articulos 

proveedores → 1 a N → articulos 

clientes → 1 a N → pedidos 

pedidos → 1 a N → lineas_pedido 

pedidos → 1 a 1 → facturas 

articulos → 1 a N → lineas_pedido 

3. Funciones PL/pgSQL 

Crear un pedido 

CREATE OR REPLACE FUNCTION crear_pedido(p_id_cliente INT) 

RETURNS INT AS $$ 

DECLARE 

    v_id_pedido INT; 

BEGIN 

    INSERT INTO pedidos (id_cliente) 

    VALUES (p_id_cliente) 

    RETURNING id_pedido INTO v_id_pedido; 

 

    RETURN v_id_pedido; 

END; 

$$ LANGUAGE plpgsql; 

$$ LANGUAGE plpgsql; 

 

 

 

Añadir línea de pedido (con control de stock) 

CREATE OR REPLACE FUNCTION agregar_linea_pedido( 

    p_id_pedido INT, 

    p_id_articulo INT, 

    p_cantidad INT 

) 

RETURNS VOID AS $$ 

DECLARE 

    v_precio NUMERIC(10,2); 

    v_stock INT; 

BEGIN 

    SELECT precio, stock 

    INTO v_precio, v_stock 

    FROM articulos 

    WHERE id_articulo = p_id_articulo; 

 

    IF v_stock < p_cantidad THEN 

        RAISE EXCEPTION 'Stock insuficiente'; 

    END IF; 

 

    INSERT INTO lineas_pedido ( 

        id_pedido, 

        id_articulo, 

        cantidad, 

        precio_unitario, 

        subtotal 

    ) 

    VALUES ( 

        p_id_pedido, 

        p_id_articulo, 

        p_cantidad, 

        v_precio, 

        v_precio * p_cantidad 

    ); 

 

    UPDATE pedidos 

    SET total = total + (v_precio * p_cantidad) 

    WHERE id_pedido = p_id_pedido; 

END; 

$$ LANGUAGE plpgsql; 

 

 

 

 

 

Facturar pedido (descuenta stock y cierra pedido) 

CREATE OR REPLACE FUNCTION facturar_pedido(p_id_pedido INT) 

RETURNS VOID AS $$ 

DECLARE 

    r RECORD; 

    v_total NUMERIC(10,2); 

BEGIN 

    SELECT estado, total INTO r 

    FROM pedidos 

    WHERE id_pedido = p_id_pedido; 

 

    IF r.estado <> 'ABIERTO' THEN 

        RAISE EXCEPTION 'El pedido ya está facturado'; 

    END IF; 

 

    FOR r IN 

        SELECT id_articulo, cantidad 

        FROM lineas_pedido 

        WHERE id_pedido = p_id_pedido 

    LOOP 

        UPDATE articulos 

        SET stock = stock - r.cantidad 

        WHERE id_articulo = r.id_articulo; 

    END LOOP; 

 

    INSERT INTO facturas (id_pedido, total) 

    VALUES (p_id_pedido, (SELECT total FROM pedidos WHERE id_pedido = p_id_pedido)); 

 

    UPDATE pedidos 

    SET estado = 'FACTURADO' 

    WHERE id_pedido = p_id_pedido; 

END; 

$$ LANGUAGE plpgsql; 

 

  

Datos de ejemplo 

INSERT INTO categorias (nombre) VALUES 

('Portátiles'), 

('Componentes'), 

('Periféricos'); 

 

INSERT INTO proveedores (nombre) VALUES 

('TechSupplier'), 

('HardwarePro'); 

 

INSERT INTO articulos (nombre, precio, stock, id_categoria, id_proveedor) VALUES 

('Laptop Lenovo', 850.00, 10, 1, 1), 

('Mouse Logitech', 25.00, 50, 3, 2), 

('SSD 1TB', 120.00, 20, 2, 1); 

 

INSERT INTO clientes (nombre, email) VALUES 

('Juan Pérez', 'juan@email.com'), 

('Empresa XYZ', 'ventas@xyz.com'); 

Ejemplo de uso 

-- Crear pedido 

SELECT crear_pedido(1); 

 

-- Agregar líneas 

SELECT agregar_linea_pedido(1, 1, 1); 

SELECT agregar_linea_pedido(1, 2, 2); 

 

-- Facturar 

SELECT facturar_pedido(1); 

 

 

 

5. Consultas útiles 

Pedidos abiertos de un cliente 

Detalle de un pedido 

Factura de un pedido 

Stock bajo 

Ventas por artículo 

Estadísticas por rango de fechas, mes, cliente, categoría 

Todo documentado y explicado en los pasos anteriores del chat. 

Consultas útiles (para API NestJS) 

Estas son queries reales que usarías en servicios (@nestjs/typeorm o pg). 

 

🔹 Pedidos abiertos de un cliente 

SELECT p.id_pedido, p.fecha_pedido, p.total 

FROM pedidos p 

WHERE p.id_cliente = $1 

  AND p.estado = 'ABIERTO'; 

 

🔹 Detalle completo de un pedido 

SELECT 

    a.nombre, 

    lp.cantidad, 

    lp.precio_unitario, 

    lp.subtotal 

FROM lineas_pedido lp 

JOIN articulos a ON a.id_articulo = lp.id_articulo 

WHERE lp.id_pedido = $1; 

 

🔹 Factura con datos del cliente 

SELECT 

    f.id_factura, 

    f.fecha_factura, 

    c.nombre AS cliente, 

    f.total 

FROM facturas f 

JOIN pedidos p ON p.id_pedido = f.id_pedido 

JOIN clientes c ON c.id_cliente = p.id_cliente 

WHERE f.id_factura = $1; 

 

🔹 Stock bajo (alertas) 

SELECT id_articulo, nombre, stock 

FROM articulos 

WHERE stock < 5 

ORDER BY stock ASC; 

 

🔹 Ventas por artículo 

SELECT 

    a.nombre, 

    SUM(lp.cantidad) AS unidades_vendidas, 

    SUM(lp.subtotal) AS total_vendido 

FROM lineas_pedido lp 

JOIN articulos a ON a.id_articulo = lp.id_articulo 

GROUP BY a.nombre 

ORDER BY total_vendido DESC; 

 

3️⃣ Consultas estadísticas usando rangos 📊 

Muy típicas para panel de administración. 

 

🔹 Facturación por rango de fechas 

SELECT 

    DATE(fecha_factura) AS dia, 

    SUM(total) AS total_facturado 

FROM facturas 

WHERE fecha_factura BETWEEN $1 AND $2 

GROUP BY DATE(fecha_factura) 

ORDER BY dia; 

 

🔹 Ventas mensuales 

SELECT 

    DATE_TRUNC('month', fecha_factura) AS mes, 

    SUM(total) AS total 

FROM facturas 

GROUP BY mes 

ORDER BY mes; 

 

🔹 Clientes que más compran (ranking) 

SELECT 

    c.nombre, 

    SUM(f.total) AS total_comprado 

FROM facturas f 

JOIN pedidos p ON p.id_pedido = f.id_pedido 

JOIN clientes c ON c.id_cliente = p.id_cliente 

GROUP BY c.nombre 

ORDER BY total_comprado DESC 

LIMIT 10; 

 

🔹 Productos más vendidos en un rango 

SELECT 

    a.nombre, 

    SUM(lp.cantidad) AS unidades 

FROM lineas_pedido lp 

JOIN articulos a ON a.id_articulo = lp.id_articulo 

JOIN pedidos p ON p.id_pedido = lp.id_pedido 

WHERE p.fecha_pedido BETWEEN $1 AND $2 

GROUP BY a.nombre 

ORDER BY unidades DESC; 

 

🔹 Ingresos por categoría 

SELECT 

    c.nombre AS categoria, 

    SUM(lp.subtotal) AS total 

FROM lineas_pedido lp 

JOIN articulos a ON a.id_articulo = lp.id_articulo 

JOIN categorias c ON c.id_categoria = a.id_categoria 

GROUP BY c.nombre 

ORDER BY total DESC; 

 

 

Funciones de ventana: 

🧠 Recordatorio rápido (qué hacen) 

Función 

Para qué sirve 

ROW_NUMBER() 

Numeración única por fila 

RANK() 

Ranking con saltos 

DENSE_RANK() 

Ranking sin saltos 

PARTITION BY 

Agrupar sin perder filas 

SUM() OVER() 

Totales acumulados 

AVG() OVER() 

Medias 

LAG() 

Valor anterior 

LEAD() 

Valor siguiente 

 

1️⃣ Ranking de clientes por facturación total 

🏆 Top clientes (ranking global) 

SELECT 

    c.nombre, 

    SUM(f.total) AS total_facturado, 

    RANK() OVER (ORDER BY SUM(f.total) DESC) AS ranking 

FROM facturas f 

JOIN facturas_pedidos fp ON fp.id_factura = f.id_factura 

JOIN pedidos p ON p.id_pedido = fp.id_pedido 

JOIN clientes c ON c.id_cliente = p.id_cliente 

GROUP BY c.nombre; 

📌 Usa: 

RANK() → si hay empate, salta números 

 

2️⃣ Ranking de clientes SIN saltos 

SELECT 

    c.nombre, 

    SUM(f.total) AS total_facturado, 

    DENSE_RANK() OVER (ORDER BY SUM(f.total) DESC) AS ranking 

FROM facturas f 

JOIN facturas_pedidos fp ON fp.id_factura = f.id_factura 

JOIN pedidos p ON p.id_pedido = fp.id_pedido 

JOIN clientes c ON c.id_cliente = p.id_cliente 

GROUP BY c.nombre; 

 

3️⃣ Ranking de productos más vendidos por categoría 

SELECT 

    c.nombre AS categoria, 

    a.nombre AS articulo, 

    SUM(lp.cantidad) AS unidades_vendidas, 

    DENSE_RANK() OVER ( 

        PARTITION BY c.nombre 

        ORDER BY SUM(lp.cantidad) DESC 

    ) AS ranking_categoria 

FROM lineas_pedido lp 

JOIN articulos a ON a.id_articulo = lp.id_articulo 

JOIN categorias c ON c.id_categoria = a.id_categoria 

GROUP BY c.nombre, a.nombre; 

📌 Aquí PARTITION BY crea un ranking independiente por categoría. 

 

4️⃣ Ventas acumuladas por fecha (running total) 

SELECT 

    DATE(f.fecha_factura) AS fecha, 

    SUM(f.total) AS total_dia, 

    SUM(SUM(f.total)) OVER ( 

        ORDER BY DATE(f.fecha_factura) 

    ) AS acumulado 

FROM facturas f 

GROUP BY DATE(f.fecha_factura) 

ORDER BY fecha; 

📊 Ideal para gráficas. 

 

5️⃣ Facturación mensual + promedio móvil 

SELECT 

    DATE_TRUNC('month', fecha_factura) AS mes, 

    SUM(total) AS total_mes, 

    AVG(SUM(total)) OVER ( 

        ORDER BY DATE_TRUNC('month', fecha_factura) 

        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW 

    ) AS promedio_3_meses 

FROM facturas 

GROUP BY mes 

ORDER BY mes; 

📌 Media móvil de 3 meses. 

 

6️⃣ Numerar pedidos de cada cliente (historial) 

SELECT 

    p.id_cliente, 

    p.id_pedido, 

    p.fecha_pedido, 

    ROW_NUMBER() OVER ( 

        PARTITION BY p.id_cliente 

        ORDER BY p.fecha_pedido 

    ) AS numero_pedido_cliente 

FROM pedidos p; 

📌 Cada cliente tiene su propio contador. 

 

7️⃣ Comparar ventas entre días (LAG) 

SELECT 

    fecha, 

    total_dia, 

    total_dia - LAG(total_dia) OVER (ORDER BY fecha) AS diferencia 

FROM ( 

    SELECT 

        DATE(fecha_factura) AS fecha, 

        SUM(total) AS total_dia 

    FROM facturas 

    GROUP BY DATE(fecha_factura) 

) t 

ORDER BY fecha; 

📊 Muestra si se vende más o menos que el día anterior. 

 

8️⃣ Detectar caídas o subidas grandes de ventas 

SELECT 

    fecha, 

    total_dia, 

    ROUND( 

        (total_dia - LAG(total_dia) OVER (ORDER BY fecha)) 

        / NULLIF(LAG(total_dia) OVER (ORDER BY fecha), 0) * 100, 

        2 

    ) AS variacion_porcentaje 

FROM ( 

    SELECT 

        DATE(fecha_factura) AS fecha, 

        SUM(total) AS total_dia 

    FROM facturas 

    GROUP BY DATE(fecha_factura) 

) t; 

 

9️⃣ Stock relativo por proveedor (comparación interna) 

SELECT 

    pr.nombre AS proveedor, 

    a.nombre AS articulo, 

    a.stock, 

    AVG(a.stock) OVER (PARTITION BY pr.id_proveedor) AS stock_promedio_proveedor 

FROM articulos a 

JOIN proveedores pr ON pr.id_proveedor = a.id_proveedor; 

📌 Compara cada artículo contra la media de su proveedor. 

 

🔟 Artículo más caro y más barato por categoría 

SELECT * 

FROM ( 

    SELECT 

        c.nombre AS categoria, 

        a.nombre AS articulo, 

        a.precio, 

        RANK() OVER ( 

            PARTITION BY c.nombre 

            ORDER BY a.precio DESC 

        ) AS rank_caro, 

        RANK() OVER ( 

            PARTITION BY c.nombre 

            ORDER BY a.precio ASC 

        ) AS rank_barato 

    FROM articulos a 

    JOIN categorias c ON c.id_categoria = a.id_categoria 

) t 

WHERE rank_caro = 1 OR rank_barato = 1; 

 

1️⃣1️⃣ Peso de cada pedido dentro de una factura 

SELECT 

    fp.id_factura, 

    p.id_pedido, 

    p.total, 

    ROUND( 

        p.total / SUM(p.total) OVER (PARTITION BY fp.id_factura) * 100, 

        2 

    ) AS porcentaje_factura 

FROM facturas_pedidos fp 

JOIN pedidos p ON p.id_pedido = fp.id_pedido; 

📌 Muy útil para facturas consolidadas. 

 

1️⃣2️⃣ Ranking de pedidos de un cliente por importe 

SELECT 

    p.id_cliente, 

    p.id_pedido, 

    p.total, 

    DENSE_RANK() OVER ( 

        PARTITION BY p.id_cliente 

        ORDER BY p.total DESC 

    ) AS ranking_pedido_cliente 

FROM pedidos p; 

 

🧠 Resumen conceptual (para memorizar) 

OVER ( 

  PARTITION BY grupo 

  ORDER BY criterio 

) 

PARTITION BY → separa grupos 

ORDER BY → orden interno 

OVER → no colapsa filas (clave) 

 

Uso práctico con NestJS 🧠 

📦 Arquitectura recomendada 

src/ 

 ├─ clientes/ 

 ├─ articulos/ 

 ├─ pedidos/ 

 │   ├─ pedidos.service.ts 

 │   ├─ pedidos.controller.ts 

 ├─ facturas/ 

 ├─ estadisticas/ 

 

🔹 Llamar a funciones PostgreSQL desde NestJS 

Ejemplo usando pg: 

await client.query( 

  'SELECT agregar_linea_pedido($1, $2, $3)', 

  [pedidoId, articuloId, cantidad], 

); 

await client.query( 

  'SELECT facturar_pedido($1)', 

  [pedidoId], 

); 

✔️ Ideal porque: 

La lógica vive en la BD 

NodeJS solo orquesta 

Evitas inconsistencias de stock 

 

🔐 Buenas prácticas 

Usa transacciones en operaciones críticas 

No permitas modificar pedidos FACTURADO 

Índices recomendados: 

CREATE INDEX idx_pedidos_fecha ON pedidos(fecha_pedido); 

CREATE INDEX idx_facturas_fecha ON facturas(fecha_factura); 

CREATE INDEX idx_articulos_stock ON articulos(stock); 

 

 

Funcion que te agrupe todos los pedidos abiertos de un cliente por fechas y todos los abiertos en una unica factura. 

Objetivo de la función 

Dado: 

un cliente 

un rango de fechas 

La función debe: 

Buscar todos los pedidos ABIERTOS de ese cliente en ese rango 

Agruparlos 

Generar UNA ÚNICA FACTURA 

Descontar stock 

Marcar todos los pedidos como FACTURADOS 

Devolver el id de la factura creada 

 

🧠 Decisión de diseño (importante) 

Tu modelo actual tiene: 

facturas.id_pedido UNIQUE 

👉 Eso no permite una factura con varios pedidos. 

✅ Solución correcta (mínimo cambio) 

1️⃣ Eliminar la relación directa factura → pedido 
2️⃣ Crear una tabla intermedia: 

facturas_pedidos 

Esto es 100% correcto a nivel contable y técnico. 

 

1️⃣ Ajuste del modelo (necesario) 

🔹 Modificar tabla facturas 

ALTER TABLE facturas 

DROP COLUMN id_pedido; 

 

🔹 Tabla intermedia facturas_pedidos 

CREATE TABLE facturas_pedidos ( 

    id_factura INT REFERENCES facturas(id_factura) ON DELETE CASCADE, 

    id_pedido INT REFERENCES pedidos(id_pedido), 

    PRIMARY KEY (id_factura, id_pedido) 

); 

✔️ Ahora: 

1 factura → N pedidos 

1 pedido → 1 factura (en la práctica) 

 

2️⃣ Función: facturar pedidos abiertos por cliente y rango 

📌 Firma de la función 

facturar_pedidos_cliente( 

    p_id_cliente INT, 

    p_fecha_inicio DATE, 

    p_fecha_fin DATE 

) RETURNS INT 

Devuelve → id_factura 

 

3️⃣ Función completa (PL/pgSQL) 

CREATE OR REPLACE FUNCTION facturar_pedidos_cliente( 

    p_id_cliente INT, 

    p_fecha_inicio DATE, 

    p_fecha_fin DATE 

) 

RETURNS INT AS $$ 

DECLARE 

    v_id_factura INT; 

    v_total NUMERIC(10,2); 

    r_pedido RECORD; 

    r_linea RECORD; 

BEGIN 

    -- Calcular total de todos los pedidos abiertos del cliente 

    SELECT COALESCE(SUM(total), 0) 

    INTO v_total 

    FROM pedidos 

    WHERE id_cliente = p_id_cliente 

      AND estado = 'ABIERTO' 

      AND fecha_pedido BETWEEN p_fecha_inicio AND p_fecha_fin; 

 

    IF v_total = 0 THEN 

        RAISE EXCEPTION 'No hay pedidos abiertos para facturar en ese rango'; 

    END IF; 

 

    -- Crear factura 

    INSERT INTO facturas (fecha_factura, total) 

    VALUES (CURRENT_TIMESTAMP, v_total) 

    RETURNING id_factura INTO v_id_factura; 

 

    -- Procesar cada pedido 

    FOR r_pedido IN 

        SELECT id_pedido 

        FROM pedidos 

        WHERE id_cliente = p_id_cliente 

          AND estado = 'ABIERTO' 

          AND fecha_pedido BETWEEN p_fecha_inicio AND p_fecha_fin 

    LOOP 

        -- Descontar stock por cada línea 

        FOR r_linea IN 

            SELECT id_articulo, cantidad 

            FROM lineas_pedido 

            WHERE id_pedido = r_pedido.id_pedido 

        LOOP 

            UPDATE articulos 

            SET stock = stock - r_linea.cantidad 

            WHERE id_articulo = r_linea.id_articulo; 

        END LOOP; 

 

        -- Relacionar pedido con la factura 

        INSERT INTO facturas_pedidos (id_factura, id_pedido) 

        VALUES (v_id_factura, r_pedido.id_pedido); 

 

        -- Marcar pedido como facturado 

        UPDATE pedidos 

        SET estado = 'FACTURADO' 

        WHERE id_pedido = r_pedido.id_pedido; 

    END LOOP; 

 

    RETURN v_id_factura; 

END; 

$$ LANGUAGE plpgsql; 

 

 

4️⃣ Ejemplo de uso 

SELECT facturar_pedidos_cliente( 

    1, 

    '2026-01-01', 

    '2026-01-31' 

); 

✔️ Resultado: 

Se crea 1 factura 

Incluye todos los pedidos abiertos de enero 

Stock actualizado 

Pedidos cerrados 

 

5️⃣ Consultas útiles tras la factura consolidada 

🔹 Ver pedidos incluidos en una factura 

SELECT p.id_pedido, p.fecha_pedido, p.total 

FROM facturas_pedidos fp 

JOIN pedidos p ON p.id_pedido = fp.id_pedido 

WHERE fp.id_factura = $1; 

 

🔹 Total facturado a un cliente en un período 

SELECT SUM(f.total) 

FROM facturas f 

JOIN facturas_pedidos fp ON fp.id_factura = f.id_factura 

JOIN pedidos p ON p.id_pedido = fp.id_pedido 

WHERE p.id_cliente = $1 

  AND f.fecha_factura BETWEEN $2 AND $3; 

 

6️⃣ Llamada desde NestJS (ejemplo) 

const result = await client.query( 

  'SELECT facturar_pedidos_cliente($1, $2, $3) AS id_factura', 

  [clienteId, fechaInicio, fechaFin], 

); 

 

const idFactura = result.rows[0].id_factura; 

Variables con : $1 $2 etc 

¿Qué son $1, $2, $3 en SQL? 

En PostgreSQL NO son variables SQL, son parámetros posicionales de una consulta preparada (prepared statement). 

👉 Significan: 

$1 → primer valor que envías 

$2 → segundo valor 

$3 → tercer valor 

…y así sucesivamente 

Sirven para: 

Evitar SQL Injection 

Reutilizar la consulta 

Separar SQL de datos 

 

🔹 Ejemplo simple (PostgreSQL puro) 

SELECT * 

FROM clientes 

WHERE id_cliente = $1; 

Si ejecutas eso, PostgreSQL espera que le pases un valor para $1. 

 

🔹 Ejemplo real desde NodeJS / NestJS 

Código NestJS (pg) 

await client.query( 

  'SELECT * FROM clientes WHERE id_cliente = $1', 

  [5] 

); 

🔹 Lo que ocurre internamente: 

SQL 

Valor 

$1 

5 

PostgreSQL ejecuta: 

SELECT * FROM clientes WHERE id_cliente = 5; 

 

🔹 Ejemplo con varios parámetros 

SELECT * 

FROM pedidos 

WHERE id_cliente = $1porque en node lleva doble corchetes :  [[2, 4, 7] 

  AND fecha_pedido BETWEEN $2 AND $3; 

await client.query( 

  ` 

  SELECT * 

  FROM pedidos 

  WHERE id_cliente = $1 

    AND fecha_pedido BETWEEN $2 AND $3 

  `, 

  [1, '2026-01-01', '2026-01-31'] 

); 

Correspondencia: 

Parámetro 

Valor 

$1 

1 

$2 

'2026-01-01' 

$3 

'2026-01-31' 

⚠️ El orden importa, no el nombre. 

 

🔹 ¿Dónde se pueden usar $1, $2? 

✔️ En: 

SELECT 

INSERT 

UPDATE 

DELETE 

Llamadas a funciones 

❌ NO se usan: 

Como nombres de columnas 

Como nombres de tablas 

Para SQL dinámico estructural 

 

🔹 Ejemplo con funciones PL/pgSQL 

SQL 

SELECT facturar_pedidos_cliente($1, $2, $3); 

NestJS 

await client.query( 

  'SELECT facturar_pedidos_cliente($1, $2, $3)', 

  [clienteId, fechaInicio, fechaFin] 

); 

Aquí: 

$1 → clienteId 

$2 → fechaInicio 

$3 → fechaFin 

 

🔹 ¿En PL/pgSQL también existen? 

❌ No de la misma forma 

En PL/pgSQL: 

Usas parámetros con nombre 

CREATE FUNCTION ejemplo(p_id INT, p_fecha DATE) 

Dentro de la función: 

WHERE id_cliente = p_id; 

👉 Los $1, $2 solo existen fuera, cuando llamas a SQL desde la app. 

 

🔹 ¿Por qué NO concatenar strings? 

❌ MAL (peligroso): 

'SELECT * FROM clientes WHERE id_cliente = ' + id 

✔️ BIEN (seguro): 

'SELECT * FROM clientes WHERE id_cliente = $1', [id] 

Protege contra: 

1; DROP TABLE clientes; 

 

🔹 Relación con TypeORM 

Si usas TypeORM QueryBuilder: 

.createQueryBuilder('p') 

.where('p.id_cliente = :id', { id: 1 }) 

TypeORM internamente lo convierte a: 

... WHERE id_cliente = $1 

 

🧠 Resumen rápido 

$1, $2, $3 = parámetros posicionales 

Se reemplazan por valores que manda NodeJS 

Evitan SQL Injection 

El orden es clave 

En funciones PL/pgSQL se usan nombres, no $1 

 

Qué hacer en la consola 

✔️ Opción 1: Sustituir $1 por un valor real 

SELECT * FROM clientes WHERE id_cliente = 1; 

 

✔️ Opción 2: Probar funciones (esto sí funciona) 

SELECT facturar_pedidos_cliente(1, '2026-01-01', '2026-01-31'); 

Aquí no hay $1, porque los parámetros van directamente. 

 

🔹 ¿Entonces cómo prueba uno consultas con $1? 

En NodeJS / NestJS 

await client.query( 

  'SELECT * FROM clientes WHERE id_cliente = $1', 

  [1] 

); 

Ahí el driver pg: 

Prepara la consulta 

Sustituye $1 

La envía al motor 

 

🔹 ¿Existe algo parecido en psql? 

Sí, pero es otra cosa: variables de psql. 

Ejemplo en psql: 

\set id_cliente 1 

 

SELECT * FROM clientes WHERE id_cliente = :id_cliente; 

⚠️ Esto: 

❌ NO es $1 

❌ NO es SQL estándar 

✅ Es solo para la consola 

 

🧠 Diferencia visual clara 

❌ Esto NO funciona en consola 

SELECT * FROM pedidos WHERE id_cliente = $1; 

✅ Esto SÍ funciona en consola 

SELECT * FROM pedidos WHERE id_cliente = 1; 

 

🧠 Cuándo usar cada cosa 

En desarrollo (psql / pgAdmin) 

Usa valores reales (= 1, fechas fijas) 

O llama funciones directamente 

En aplicación (NestJS) 

Usa $1, $2, $3 

 

🔥 Truco profesional 

Cuando desarrollas: 

Prueba la lógica con valores fijos 

Luego reemplaza por $1, $2 en el código NodeJS 

 

📌 Resumen final 

$1 ❌ no existe en la consola SQL 

$1 ✅ existe solo en consultas preparadas (NodeJS) 

En consola: 

usa valores reales 

o variables psql (:variable) 

No es un error tuyo, es cómo funciona PostgreSQL 

 

 

* Funcion que pase solamente un pedido a factura 

La idea es sencilla: 

🎯 Objetivo 

Recibir un solo pedido 

Verificar que esté ABIERTO 

Crear una factura 

Descontar stock 

Relacionar pedido ↔ factura 

Marcar el pedido como FACTURADO 

Devolver el id de la factura 

 

 

Función completa (PL/pgSQL) 

CREATE OR REPLACE FUNCTION facturar_pedido_individual( 

    p_id_pedido INT 

) 

RETURNS INT AS $$ 

DECLARE 

    v_estado VARCHAR(20); 

    v_total NUMERIC(10,2); 

    v_id_factura INT; 

    r_linea RECORD; 

BEGIN 

    -- Verificar estado del pedido 

    SELECT estado, total 

    INTO v_estado, v_total 

    FROM pedidos 

    WHERE id_pedido = p_id_pedido; 

 

    IF NOT FOUND THEN 

        RAISE EXCEPTION 'El pedido no existe'; 

    END IF; 

 

    IF v_estado <> 'ABIERTO' THEN 

        RAISE EXCEPTION 'El pedido no está abierto'; 

    END IF; 

 

    -- Crear factura 

    INSERT INTO facturas (fecha_factura, total) 

    VALUES (CURRENT_TIMESTAMP, v_total) 

    RETURNING id_factura INTO v_id_factura; 

 

    -- Descontar stock por líneas 

    FOR r_linea IN 

        SELECT id_articulo, cantidad 

        FROM lineas_pedido 

        WHERE id_pedido = p_id_pedido 

    LOOP 

        UPDATE articulos 

        SET stock = stock - r_linea.cantidad 

        WHERE id_articulo = r_linea.id_articulo; 

    END LOOP; 

 

    -- Relacionar pedido con factura 

    INSERT INTO facturas_pedidos (id_factura, id_pedido) 

    VALUES (v_id_factura, p_id_pedido); 

 

    -- Marcar pedido como facturado 

    UPDATE pedidos 

    SET estado = 'FACTURADO' 

    WHERE id_pedido = p_id_pedido; 

 

    RETURN v_id_factura; 

END; 

$$ LANGUAGE plpgsql; 

Ejemplo de uso (consola) 

SELECT facturar_pedido_individual(5); 

Resultado: 

Devuelve el id_factura 

Pedido 5 → FACTURADO 

Stock actualizado 

Factura creada correctamente 

Consultas útiles después 

🔹 Ver factura de un pedido 

SELECT f.* 

FROM facturas f 

JOIN facturas_pedidos fp ON fp.id_factura = f.id_factura 

WHERE fp.id_pedido = 5; 

 

🔹 Ver líneas incluidas en la factura 

SELECT a.nombre, lp.cantidad, lp.subtotal 

FROM facturas_pedidos fp 

JOIN pedidos p ON p.id_pedido = fp.id_pedido 

JOIN lineas_pedido lp ON lp.id_pedido = p.id_pedido 

JOIN articulos a ON a.id_articulo = lp.id_articulo 

WHERE fp.id_factura = $1; 

(en consola reemplazas $1 por el id real) 

Versión PRO (opcional, recomendada) 

Si quieres blindarlo aún más, se puede añadir: 

✔️ Verificación de stock negativo 

✔️ Bloqueo del pedido (FOR UPDATE) 

✔️ Transacción explícita 

✔️ Registro histórico de stock 

Ejemplo de bloqueo: 

SELECT estado, total 

FROM pedidos 

WHERE id_pedido = p_id_pedido 

FOR UPDATE; 

 

🧠 Resumen 

Esta función factura solo un pedido 

Respeta el modelo con facturas_pedidos 

Es segura y reutilizable 

Ideal para llamarla desde NestJS 

* Funcion que te pase solamente los pedidos que quieras a factura. 

🎯 Objetivo 

Reciba una lista de pedidos concretos 

Facture solo esos pedidos 

Cree UNA ÚNICA FACTURA 

Descuente stock 

Marque esos pedidos como FACTURADO 

Devuelva el id de la factura 

👉 Ideal para cuando en la app el usuario selecciona pedidos con checkboxes. 

🧠 Decisión técnica clave 

La forma correcta en PostgreSQL es pasar: 

un ARRAY de ids de pedido 

Ejemplo desde SQL / NodeJS: 

ARRAY[1,3,5,8] 

 

1️⃣ Firma de la función 

facturar_pedidos_seleccionados(p_ids_pedidos INT[]) 

RETURNS INT 

 

 

2️⃣ Función completa (PL/pgSQL) 

CREATE OR REPLACE FUNCTION facturar_pedidos_seleccionados( 

    p_ids_pedidos INT[] 

) 

RETURNS INT AS $$ 

DECLARE 

    v_id_factura INT; 

    v_total NUMERIC(10,2); 

    r_pedido RECORD; 

    r_linea RECORD; 

BEGIN 

    -- Verificar que existan pedidos abiertos 

    SELECT COALESCE(SUM(total), 0) 

    INTO v_total 

    FROM pedidos 

    WHERE id_pedido = ANY(p_ids_pedidos) 

      AND estado = 'ABIERTO'; 

 

    IF v_total = 0 THEN 

        RAISE EXCEPTION 'No hay pedidos abiertos para facturar'; 

    END IF; 

 

    -- Crear factura 

    INSERT INTO facturas (fecha_factura, total) 

    VALUES (CURRENT_TIMESTAMP, v_total) 

    RETURNING id_factura INTO v_id_factura; 

 

    -- Procesar pedidos uno a uno 

    FOR r_pedido IN 

        SELECT id_pedido 

        FROM pedidos 

        WHERE id_pedido = ANY(p_ids_pedidos) 

          AND estado = 'ABIERTO' 

    LOOP 

        -- Descontar stock 

        FOR r_linea IN 

            SELECT id_articulo, cantidad 

            FROM lineas_pedido 

            WHERE id_pedido = r_pedido.id_pedido 

        LOOP 

            UPDATE articulos 

            SET stock = stock - r_linea.cantidad 

            WHERE id_articulo = r_linea.id_articulo; 

        END LOOP; 

 

        -- Relacionar pedido con factura 

        INSERT INTO facturas_pedidos (id_factura, id_pedido) 

        VALUES (v_id_factura, r_pedido.id_pedido); 

 

        -- Marcar pedido como facturado 

        UPDATE pedidos 

        SET estado = 'FACTURADO' 

        WHERE id_pedido = r_pedido.id_pedido; 

    END LOOP; 

 

    RETURN v_id_factura; 

END; 

$$ LANGUAGE plpgsql; 

 

3️⃣ Ejemplos de uso 

🔹 En consola PostgreSQL 

SELECT facturar_pedidos_seleccionados(ARRAY[2,4,7]); 

 

🔹 Desde NestJS / NodeJS (pg) 

await client.query( 

  'SELECT facturar_pedidos_seleccionados($1)', 

  [[2, 4, 7]] 

); 

📌 Ojo: el array se pasa como array JS, no como string. 

 

4️⃣ Consultas útiles después 

🔹 Ver pedidos incluidos en la factura 

SELECT p.id_pedido, p.fecha_pedido, p.total 

FROM facturas_pedidos fp 

JOIN pedidos p ON p.id_pedido = fp.id_pedido 

WHERE fp.id_factura = 10; 

5️⃣ Validaciones que ya tienes (y por qué está bien) 

✔️ Ignora pedidos ya facturados 
✔️ No crea factura vacía 
✔️ Una sola factura para N pedidos 
✔️ Stock consistente 

6️⃣ Versión PRO (recomendada si es producción) 

Si quieres subir nota o hacerlo “real world”, se puede añadir: 

🔒 SELECT ... FOR UPDATE para bloquear pedidos 

🔄 Rollback automático si algún stock queda negativo 

🧾 Verificar que todos los pedidos sean del mismo cliente 

📦 Tabla de movimientos de stock 

Ejemplo de validación de cliente: 

IF (SELECT COUNT(DISTINCT id_cliente) 

    FROM pedidos 

    WHERE id_pedido = ANY(p_ids_pedidos)) > 1 THEN 

    RAISE EXCEPTION 'Los pedidos deben ser del mismo cliente'; 

END IF; 

🧠 Resumen mental 

1 pedido → facturar_pedido_individual 

Pedidos de un cliente por fechas → facturar_pedidos_cliente 

Pedidos seleccionados manualmente → facturar_pedidos_seleccionados 

Con esto tienes todos los escenarios de facturación cubiertos 💪 

Rollback automático si algún stock queda negativo 

🧠 Idea clave (importantísima) 

En PostgreSQL: 

🔥 Si una función PL/pgSQL lanza una EXCEPTION, 
TODA la transacción se hace ROLLBACK automáticamente 

No tienes que escribir ROLLBACK a mano si la función se ejecuta dentro de una transacción (y desde NestJS SIEMPRE lo está, aunque no lo veas). 

❓ Entonces, ¿cómo forzamos el rollback? 

👉 Detectando stock insuficiente y lanzando una excepción 

RAISE EXCEPTION 'Stock insuficiente'; 

Eso hace que: 

❌ No se descuente stock 

❌ No se cree factura 

❌ No se cambie el estado de pedidos 

✅ La base de datos queda exactamente como estaba 

 

🚨 Problema de la versión anterior 

Antes hacíamos: 

UPDATE articulos 

SET stock = stock - cantidad; 

❌ Eso permite que el stock quede negativo si no se valida. 

 

✅ Solución correcta (nivel pro) 

Bloquear el artículo 

Comprobar stock 

Si no alcanza → EXCEPTION 

PostgreSQL hace rollback solo 

 

1️⃣ Función con ROLLBACK automático por stock negativo 

Te dejo la versión segura de 
facturar_pedidos_seleccionados 

CREATE OR REPLACE FUNCTION facturar_pedidos_seleccionados( 

    p_ids_pedidos INT[] 

) 

RETURNS INT AS $$ 

DECLARE 

    v_id_factura INT; 

    v_total NUMERIC(10,2); 

    r_pedido RECORD; 

    r_linea RECORD; 

    v_stock INT; 

BEGIN 

    -- Validar que todos los pedidos estén abiertos 

    SELECT COALESCE(SUM(total), 0) 

    INTO v_total 

    FROM pedidos 

    WHERE id_pedido = ANY(p_ids_pedidos) 

      AND estado = 'ABIERTO'; 

 

    IF v_total = 0 THEN 

        RAISE EXCEPTION 'No hay pedidos abiertos para facturar'; 

    END IF; 

 

    -- Crear factura 

    INSERT INTO facturas (fecha_factura, total) 

    VALUES (CURRENT_TIMESTAMP, v_total) 

    RETURNING id_factura INTO v_id_factura; 

 

    -- Procesar pedidos 

    FOR r_pedido IN 

        SELECT id_pedido 

        FROM pedidos 

        WHERE id_pedido = ANY(p_ids_pedidos) 

          AND estado = 'ABIERTO' 

        FOR UPDATE 

    LOOP 

        -- Procesar líneas del pedido 

        FOR r_linea IN 

            SELECT id_articulo, cantidad 

            FROM lineas_pedido 

            WHERE id_pedido = r_pedido.id_pedido 

        LOOP 

            -- Bloquear artículo y leer stock 

            SELECT stock 

            INTO v_stock 

            FROM articulos 

            WHERE id_articulo = r_linea.id_articulo 

            FOR UPDATE; 

 

            IF v_stock < r_linea.cantidad THEN 

                RAISE EXCEPTION 

                  'Stock insuficiente para el artículo %, disponible %, requerido %', 

                  r_linea.id_articulo, v_stock, r_linea.cantidad; 

            END IF; 

 

            -- Descontar stock (seguro) 

            UPDATE articulos 

            SET stock = stock - r_linea.cantidad 

            WHERE id_articulo = r_linea.id_articulo; 

        END LOOP; 

 

        -- Relacionar pedido con factura 

        INSERT INTO facturas_pedidos (id_factura, id_pedido) 

        VALUES (v_id_factura, r_pedido.id_pedido); 

 

        -- Marcar pedido como facturado 

        UPDATE pedidos 

        SET estado = 'FACTURADO' 

        WHERE id_pedido = r_pedido.id_pedido; 

    END LOOP; 

 

    RETURN v_id_factura; 

END; 

$$ LANGUAGE plpgsql; 

 

🔥 Qué pasa si falla el stock (paso a paso) 

Supón: 

Pedido A necesita 10 SSD 

Stock real = 5 

1️⃣ La función empieza 
2️⃣ Crea la factura 
3️⃣ Empieza a descontar 
4️⃣ Detecta v_stock < cantidad 
5️⃣ RAISE EXCEPTION 
6️⃣ PostgreSQL hace ROLLBACK TOTAL 

Resultado final: 

Elemento 

Estado 

Factura 

❌ No existe 

Stock 

✅ Sin cambios 

Pedidos 

✅ Siguen ABIERTOS 

BD 

✅ Consistente 

💯 Atomicidad perfecta 

 

🔐 Por qué usamos FOR UPDATE 

SELECT stock FROM articulos ... FOR UPDATE; 

Esto: 

🔒 Bloquea la fila 

Evita que otro pedido facture el mismo stock al mismo tiempo 

Imprescindible en producción 

 

🧪 Cómo probar que funciona 

BEGIN; 

 

SELECT facturar_pedidos_seleccionados(ARRAY[1,2,3]); 

 

-- Si falla, prueba: 

ROLLBACK; 

O simplemente ejecútalo normal: 
la excepción ya hace rollback automático. 

🧠 Resumen mental 

❌ No hace falta escribir ROLLBACK 

✅ RAISE EXCEPTION = rollback automático 

🔒 FOR UPDATE evita carreras de stock 

🧾 Base de datos siempre consistente 

 
