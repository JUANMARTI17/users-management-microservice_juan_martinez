# Tarea: migrar el microservicio de MySQL a PostgreSQL (Render)

## Contexto

Este microservicio usaba **MySQL**. Ya hice lo siguiente fuera del código:

- Convertí el archivo `.sql` (esquema y datos) a sintaxis **PostgreSQL**.
- Creé la base de datos PostgreSQL en **Render**.
- Ejecuté el `.sql` contra esa base desde mi consola local, usando la **External Database URL** de Render.

**Lo que falta (tu trabajo):** adaptar el código y la configuración del microservicio para que se conecte a PostgreSQL y funcione tanto en local como desplegado (Docker / Render).

## Reglas

- **No modifiques el archivo `.sql`**: ya está migrado y ejecutado.
- **No escribas credenciales reales** en ningún archivo versionado. Todo debe leerse de variables de entorno.
- Haz cambios mínimos: solo lo necesario para la migración.
- Si algo es ambiguo (por ejemplo, una consulta nativa que no sabes si es compatible), pregúntame antes de adivinar.

## Paso 0: reconocimiento (antes de editar)

1. Detecta el stack del proyecto (Spring Boot con Maven/Gradle, Node.js, etc.) y dime qué encontraste.
2. Busca todas las referencias a MySQL: `mysql`, `com.mysql`, `MySQLDialect`, `jdbc:mysql`, `3306`, `mysql2`, imágenes `mysql` en Docker/compose.
3. Lista los archivos que vas a tocar y espera mi confirmación si son más de lo esperado.

## Tareas

### 1. Dependencias
- Quitar el driver de MySQL (`mysql-connector-j` / `mysql-connector-java` / `mysql2`).
- Agregar el driver de PostgreSQL (`org.postgresql:postgresql` en Java, o `pg` en Node).

### 2. Configuración de conexión (`application.properties` / `application.yml` / config de Node)
- Reemplazar la URL por formato PostgreSQL: `jdbc:postgresql://<host>:5432/<db>` (en Node, `postgres://user:pass@host:5432/db`).
- Driver: `org.postgresql.Driver`.
- Eliminar dialectos específicos de MySQL (`MySQLDialect`, `MySQL8Dialect`). Dejar que Hibernate lo detecte o usar `PostgreSQLDialect`.
- Como el esquema ya existe, configurar `spring.jpa.hibernate.ddl-auto=none` (o `validate`), nunca `create`/`update` contra la base de Render.
- Leer todo desde variables de entorno con valor por defecto solo para local (sin secretos), por ejemplo:
  ```properties
  spring.datasource.url=${DB_URL}
  spring.datasource.username=${DB_USER}
  spring.datasource.password=${DB_PASSWORD}
  ```

### 3. Variables de entorno
- Definir `DB_URL`, `DB_USER`, `DB_PASSWORD` (ajustar nombres a los que ya usa el proyecto).
- Crear/actualizar `.env.example` con valores de ejemplo (sin datos reales).
- Verificar que `.env` esté en `.gitignore`.
- Nota de Render:
  - **External URL**: para conectarme desde mi máquina. Normalmente requiere SSL (`?sslmode=require` o `?ssl=true`).
  - **Internal URL**: para cuando el servicio esté desplegado en Render (misma región).
  - Render entrega la URL como `postgres://user:pass@host/db`; en Java hay que convertirla a `jdbc:postgresql://host:5432/db` y pasar usuario y contraseña por separado.

### 4. Dockerfile y docker-compose
- Quitar cualquier referencia a MySQL (servicio `mysql`, puerto 3306, volúmenes, `depends_on`).
- Asegurar que el contenedor reciba las variables `DB_URL`, `DB_USER`, `DB_PASSWORD` por `environment` o `env_file`.
- No instalar clientes ni drivers de MySQL en la imagen.
- Si se mantiene una base local para desarrollo, usar la imagen `postgres` (no una de MySQL).

### 5. Compatibilidad del código con PostgreSQL
Revisar y corregir si aplica:
- Consultas nativas con sintaxis MySQL: backticks, `LIMIT x,y`, `IFNULL`, `ON DUPLICATE KEY UPDATE`, `NOW()` vs `CURRENT_TIMESTAMP`, `GROUP_CONCAT`.
- Nombres de tablas/columnas: PostgreSQL pasa a minúsculas los identificadores sin comillas; las entidades/modelos deben coincidir con lo que quedó en la base.
- Palabras reservadas usadas como nombre de tabla (por ejemplo `user`, `order`).
- Generación de IDs: `AUTO_INCREMENT` pasó a `SERIAL`/`IDENTITY`; revisar `@GeneratedValue` (`IDENTITY` o `SEQUENCE` según cómo quedó el `.sql`).
- Tipos: `TINYINT(1)` → `BOOLEAN`, `DATETIME` → `TIMESTAMP`, etc.; confirmar que las entidades coincidan.

### 6. Pruebas y verificación
1. Compilar/construir el proyecto sin errores.
2. Levantar el servicio apuntando a la base de Render (con la External URL, por variables de entorno) y confirmar que conecta.
3. Probar los endpoints principales (listar, crear, actualizar, eliminar) y reportar cualquier error SQL.
4. Construir la imagen Docker y verificar que arranca con las variables de entorno.

## Entregable

Al terminar, dame un resumen corto con:
- Archivos modificados y qué cambió en cada uno.
- Variables de entorno que debo configurar (local y en Render).
- Cualquier consulta o entidad que tuviste que adaptar.
- Pendientes o dudas que requieran mi decisión.
