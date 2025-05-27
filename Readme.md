# OlaClick DBA Challenge resultados

## Parte 1: Modelado de Datos Relacional

**Requerimiento**:

Diseña un modelo de datos relacional en PostgreSQL para soportar:

* Restaurantes con múltiples sucursales
* Clientes que realizan pedidos
* Pedidos que contienen uno o más productos
* Estado de cada pedido (initiated, sent, delivered)
* Registro de cambios de estado (con timestamp)

**Entregables**:

1. Script SQL con CREATE TABLE y relaciones (FK, PK, constraints, indexes)
    ``` sql
    --Creación base de datos
    CREATE DATABASE olaclick_orders;

    -- Tabla restaurants
    CREATE TABLE restaurants (
        id SERIAL PRIMARY KEY,
        name VARCHAR(100) NOT NULL,
        timezone VARCHAR(50) NOT NULL
    );

    -- Tabla branches
    CREATE TABLE branches (
        id SERIAL PRIMARY KEY,
        restaurant_id INT NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
        name VARCHAR(100) NOT NULL,
        address TEXT
        UNIQUE(restaurant_id, name)
    );

    --Tabla clients
    CREATE TABLE clients (
        id SERIAL PRIMARY KEY,
        name VARCHAR(100) NOT NULL,
        phone VARCHAR(20),
        email VARCHAR(100)
    );

    -- Tabla products
    CREATE TABLE products(
        id SERIAL PRIMARY KEY,
        name VARCHAR(100) NOT NULL,
        price NUMERIC(10, 2) NOT NULL,
        restaurant_id INT NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE
    );

    --Tabla orders
    CREATE TABLE orders (
        id SERIAL PRIMARY KEY,
        client_id INT NOT NULL REFERENCES clients(id),
        branch_id INT NOT NULL REFERENCES branches(id),
        status VARCHAR(20) NOT NULL CHECK (status IN ('initiated', 'sent', 'delivered')) --Se pueden cambiar por valores en español
        total NUMERIC(10, 2) NOT NULL, --Precio total de la orden con todos los productos
        created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP --Timestamp en zona horaria
    );

    --Tabla order_items
    CREATE TABLE order_items (
        id SERIAL PRIMARY KEY,
        order_id INT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
        product_id INT NOT NULL REFERENCES products(id),
        quantity INT NOT NULL CHECK (quantity > 0),
        price NUMERIC(10, 2) NOT NULL --Precio total de la orden
    );

    --Tabla order_status_history
    CREATE TABLE order_status_history (
        id SERIAL PRIMARY KEY,
        order_id INT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
        status VARCHAR(20) NOT NULL CHECK (status IN ('initiated', 'sent', 'delivered')) --Se pueden cambiar por valores en español
        changed_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP --Timestamp en zona horaria
    );

    -- Índices
    CREATE INDEX idx_orders_status_created_at ON orders(status, created at);
    CREATE INDEX idx_order_status_history_order_id ON order_status_history(order_id);
    CREATE INDEX idx_branches_restaurant_id ON branches(restaurant_id)
    CREATE INDEX idx_products_restaurant_id ON products(restaurant_id)
    CREATE INDEX idx_orders_client_id ON orders(client_id)
    CREATE INDEX idx_orders_branch_id ON orders(branch_id)
    CREATE INDEX idx_orders_items_order_id ON order_items(order_id)
    CREATE INDEX idx_orders_items_product_id ON order_items(product_id)
    CREATE INDEX idx_orders_status_history_order_id ON order_status_history(order_id)
    ```

2. Justificación de decisiones de modelado (ej. normalización, índices, tipos de datos) 

    Para realizar la base de datos, primero se realiza un diagrama que permita identificar las tablas y sus relaciones.

    ![Diagrama base de datos](./img/Diagrama%20OlaClick.png)

    A este diagrama se le aplica de una vez los principios de normalización hasta la tercera forma normal para evitar redundancias, inconsistencias y facilitar el mantenimiento:

    * 1FN: Todos los atributos tienen valores atómicos. Es decir, los productos se descomponen en una tabla separada de los pedidos, y cada producto tiene un registro propio en `order:items`
    * 2FN: Se eliminan dependencias parciales. El total del pedido se calcula en base de los productos y cantidades, pero se almacena explicitamente en `orders.total` para eficiencia en lectura
    * 3FN: Cada tabla representa una entidad única, sin dependencias transitivas. Por ejemplo, el restaurante y la sucursal están separadas para manejar correctamente sucursales múltiples sin duplicar información.

    Como se puede observar en el diagrama, cada tabla tiene una llave primaria, que garantiza unicidad y permite identificar de forma inequívoca cada registro. Adicionalmente se usan llaves foráneas para mantener la integridad referencial entre las tablas de la siguiente manera:

    * `orders.client.id` -> `clients.id`
    * `orders.branch_id` -> `branches.id`
    * `branches.restaurant_id` -> `restaurants.id`
    * `order_items.order_id` -> `orders_id`
    * `order_items.product_id` -> `products_id`
    * `products.restaurant_id` -> `restaurants.id`
    * `order_status_history.order_id` -> `orders.id`

    De esta manera nos aseguramos que no hayan pedidos sin cliente o sucursal valida y que no hayan productos huérfanos o sin relación con un restaurante.

    Se usa `ON DELETE CASCADE` en relaciones que deben eliminarse en cascada, como cuando se elimina un pedido y sus items.

    Finalmente, se crean índices para mejorar el rendimiento en operaciones de lectura y filtrado. Así pues se crean dos indices, el primero para la consulta de ordenes creadas en un período especifico y la segunda para consultar el historial por order_id. Adicionalmente, se crean índices para las llaves foráneas con el fin de mejorar su rendimiento en los joins, en las operaciones de integridad referencial, eliminaciones en cascada y consultas de búsqueda por relación.

    Se usan las siguientes restricciones:
    * `CHECK (status IN(...))`: asegura que solo se usen estados válidos
    * `NOT NULL`: Impide valores nulos en campos obligatorios
    * `UNIQUE (restaurant_id, name)` en sucursales: evita que un restaurante tenga dos sucursales con el mismo nombre.

## Parte 2: Optimización de Consultas

Dada la siguiente consulta:
```sql
SELECT p.id, p.total, c.name, r.name
FROM orders p
JOIN clients c ON p.client_id = c.id
JOIN restaurants r ON p.restaurant_id = r.id
WHERE p.status = 'sent'
AND p.created_at >= NOW() - interval '7 days';
```

Tareas:

1. Explica cómo identificarías cuellos de botella (Usa EXPLAIN ANALYZE o pg_stat_statements)

    Solución: para una sola query usaría EXPLAIN ANALYZE, ya que esta sentencia permite ver el plan de ejecución real de la consulta. De esta manera se puede ver el tipo de escaneo que se hace, los costos estimados conta los costos reales, los tiempos de operación, filtrado de filas, estrategias de uniones, etc. De esta manera, se puede evidenciar en qué parte del proceso hay un cuello de botella.

    pg_stat_statements lo usaría para evaluar el rendimiento general de la base de datos, ya que esta provee estadisticas acumulativas e individuales de todos los procesos relacionados en la base de datos.

    Eventualmente, usaría ambas. Primero `pg_stat_statements` para encontrar las queries que tienen un tiempo de ejecución alto. Y `EXPLAIN ANALYZE` para revisar profundamente aquellas queries.

2. Propón al menos 2 formas de optimizar la consulta (índices, particionamiento, etc.)

- **Propuesta 1**: Índices adecuados

    Dado que se filtra por `status` y `created_at` y se hace un join por `client_id` y `restaurant_id`, crearía índices en estas columnas para acelerar el filtrado y mejorar el rendimiento de los joins

    ```sql
    CREATE INDEX idx_orders_status_created_at ON orders (status, created_at);
    CREATE INDEX idx_orders_client_id ON orders (client_id);
    CREATE INDEX idx_orders_restaurant_id ON orders (restaurant_id);
    ```

- **Propuesta 2**: Particionamiento por rango de fecha

    La tabla orders puede crecer demasiado debido a que contiene los registros de todos los pedidos realizados, llegando a obtener millones de datos. Así pues, realizaría un particionamiento por rango de fecha, es decir por mes o por semana de la siguiente manera:

    ```sql
    CREATE TABLE orders (
        id SERIAL PRIMARY KEY,
        ...
        created_at TIMESTAMPTZ NOT NULL
    ) PARTITION BY RANGE (created_at);
    ```
    Y después se particiona, de manera continua, por el periodo seleccionado. Por ejemplo de manera mensual quedaría así:

    ```sql
    CREATE TABLE orders_2025_01 PARTITION OF orders
        FOR VALUES FROM ('2025-01-01') TO ('2025-01-31')
    CREATE TABLE orders_2025_02 PARTITION OF orders
        FOR VALUES FROM ('2025-02-01') TO ('2025-02-28')
    ```

3. Justifica cómo afectaría la solución al sistema en producción

- Optimización 1:

    Al aplicar los índices podemos aumentar la velocidad de lectura de los datos. Mejorando así la respuesta del sistema a las consultas de los usuarios o sistemas que dependen de esta consulta. De igual manera, al ser las consultas más eficientes se obtiene un consumo menor de CPU, mejorando el rendimiento del servidors.

    Sin embargo, se obtiene una sobrecarga en escrituras, ya que cada índice nuevo añade una sobrecarga a las operaciones INSERT, UPDATE y DELETE en las tablas que los contienen, aunque esto es aceptable y se compensa con las ganancias en lectura. De igual manera, los índices consumen espacio adicional en disco. Si la tabla no tiene índices y es demasiado grande, se va a consumir una importante cantidad de tiempo al crearlos, lo cual puede bloquear las escrituras en la tabla durante su creación. Esto se puede evitar usando `CREATE INDEX CONCURRENTLY`, aunque es más lenta y consume más recursos temporalmente.

- Optimización 2:

    Para la consulta dada se obtiene una mejora en el tiempo de lectura, ya que la cantidad de datos a escanear disminuye considerablemente, en caso de que esta tabla sea muy grande. Las particiones también pueden ayudar a que el mantenimiento de los datos y el archivado de estos sea más manejable y rápido. Aunque, esto es un procedimiento más complejo que añadir índices. Transformar un tabla existente a una tabla particionada es un proceso complejo, así pues se hace necesario realizar una planificación minuciosa para minimizar el tiempo de inactividad o utilizar técnicas de migración en línea

## Parte 3: Redis
1. ¿Cómo usarías Redis para mejorar el rendimiento del sistema en lectura de órdenes por estado?

    Usaría redis como un cache de las consultas que se hacen a la base de datos en PostgreSQL. Almacenando listas o conjuntos de ordenes agrupados por su estado, de esta manera si la aplicación necesita leer órdenes primero consulta redis, aunque en caso de que no esten ahí consultará PostgreSQL y creará el cache en redis

2. ¿Qué estrategias usarías para evitar inconsistencias entre Redis y PostgreSQL?

    De lo que he leído sobre redis, podría mencionar

    - Escribir primero en la DB, luego actualizar o invalidar caché
    - Leer la caché, rellenar los cache miss con consultas de postgres
    - Establecer TTL cortos
    
3. ¿Cuándo preferirías evitar usar Redis en un sistema de alta concurrencia?

    - Cuando se requiere que los datos sean consistentes y precisos en el tiempo y no con retraso
    - Cuando los datos no se leen constantemente a pesar de que se actualizan frecuentemente
    - Datos extremadamente grandes
    - Si tanto `EXPLAIN ANALYZE` como `pg_stat_statements` arrojan buenos resultados

##  Parte 4: Respaldo y Recuperación

1. Describe tu estrategia de backup para una base de datos de 100 GB que no puede tener más de 5 minutos de pérdida de datos (RPO).
    
    Después de leer sobre el tema, usaría una combinación de Write-Ahead Log (WAL) en PostgreSQL, usando un script que copie los segmentos a una ubicación de almacenamiento segura y separada, ya sea en la nube o en un servidor dedicado o una réplica en streaming (hot standby), ya que esta se realiza casi en tiempo real. Junto a un backup diario usando las herramientas estándar de PostgreSQL

2. ¿Cómo configurarías una réplica de solo lectura en PostgreSQL?

    No he hecho replicas en PostgreSQL, sin embargo, al leer sobre el tema, el proceso que usaría es el siguiente:

    En el servidor principal:

    1. Modificar los archivos `pg_hba.config` y `postgresql.conf` para aplicar la configuración de replicación
    2. Crear un usuario en la base de datos que se encargue de la replicación
    ```sql
    CREATE ROLE repuser WITH REPLICATION PASSWORD `secretPassword` LOGIN;
    ```
    3. Reiniciar PostgreSQL
    
    En el servidor secundario

    4. Eliminar, mediante consola de comandos el directorio dejando solo los archivos `pg_hba.conf` y `pg_ident.conf`
    5. Usar `pg_basebackup` para conectar la base de datos y el usuario (`repuser`) ej:
    ```bash
    sudo pg_basebackup -h 192.168.68.64 -U repuser -D /etc/postgresql/16/main
    ```
    Esto depende del tipo de distribución de linux que se tenga

    6. En `pg_hba.conf` del servidor secundario sustituir la dirección ip del servidor secundario por la del principal
    7.  Iniciar el servidor secundario usando
    ```bash
    sudo sytemctl start postgresql
    ```

3. ¿Qué herramienta o enfoque usarías para automatizar y validar los backups?

    Para la automatización

    - `pgbackrest.conf` -> ya que permite realizar respaldos completos, incrementales, validación y cifrado. De igual manera, se puede integrar en `cron` o con el programador de tareas de windows para realizar ejecuciones periódicas automáticas.

    - Barman -> Herramienta de código abierto que realiza procesos similares a `pgbackrest`

    Para la validación:
    
    - Ejecución de `pg_restore --list` para la validación de respaldos lógicos
    - `pgBackRest check` para verificar consistencia automaticamente
    - Pruebas de restauración regulares mediante una automatización parcial usando un script que seleccione el backup, restaure la base de datos en un servidor de prueba y ejecute consultas de verificación básicas y envíe un informe del resultado

## Parte 5: Seguridad y Acceso
1. ¿Qué prácticas aplicarías para proteger las credenciales de conexión a la base de datos?

    Guardaría las credenciales en un archivo .env, evitando dejarlas en el código. De igual manera, implementaría una rotación periódica de contraseñas y aplicaría una auditoría y control de acceso así como la implementación de listas blancas de IP

2. ¿Cómo controlarías el acceso a los datos entre entornos (producción, staging, desarrollo)?

    - Mediante el uso de credenciales diferentes para cada entorno
    - Bases de datos separadas para cada entorno
    - Uso de datos sinteticos o anonimizados en entornos de no producción

3. ¿Cómo implementarías auditoría de acceso a datos sensibles?

    Primero identificaría aquellas columnas que tengan datos sensibles, en nuestro caso son la identificación de los usuarios, nombres, direcciones, direcciones de correo, telefonos, métodos de pago, etc. Estos datos deberían estar anonimizados y encriptados.

    Usaría `pgAudit` para ver quién accede qué datos y cuándo, de esta manera se puede configurar alertas para actividades sospechosas o no autorizadas en los datos sensibles identificados. También implementaría un log de conexiones, de esta manera y junto a las listas blancas se puede identificar una conexión ip desconocida.