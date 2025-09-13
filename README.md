# DM_PSet_1_Gabriela_Coloma

Problem Set 1 - Data Mining

Nombre: Gabriela Coloma
Código: 00325312

***Resumen***
EL proyecto lo que hace es construir tres pipelines de backfill histórico desde QuickBooks Online (QBO) para las entidades: Invoices, Customers y Items. Los datos extraídos se guardan en Postgres (esquema raw). Los pipelines se orquestraron en Mage, se despliega con Docker Compose y se gestionaron las credenciales con Mage Secrets.

***Arquitectura***


***Pasos para levantar contenedores y configurar proyecto***

1. Clonar este repositorio con los siguientes comandos:
    git clone <repo_url>
    cd <repo>
    
2. Levantar servicios con :
     docker compose up -d
   
Los servicios son: 
- warehouse: PostgreSQL (5432)
- warehouseui: pgAdmin (8081)
- scheduler: Mage (6789)

3. Para apagar los servicios usar el siguiente comando:
     docker compose down

***Gestión de secretos***
Todos los secretos se configuraron en Mage Secrets, no fueron puestos directamente en el código sino llamados al código con la función get_secret_value(). 

Secretos: 
| Nombre secreto   | Para qué sirve                    | Cada cuánto se cambia | Quién lo actualiza |
|------------------|-----------------------------------|-----------------------|---------------------|
| `qb_client_id`   | Identificador de la app en QBO    | Solo si cambia la app | Equipo de data |
| `qb_client_secret` | Llave secreta de la app QBO     | Cada 180 días o si hay fuga | Equipo de data |
| `qb_refresh`     | Token para pedir nuevos accesos a QBO | Cada 101 días (por QBO) | Equipo de data |
| `qb_realm`       | ID de la empresa en QBO           | Nunca cambia          | Equipo de data |
| `pg_host`        | Dirección del servidor Postgres   | Solo si cambia infra  | Equipo de infraestructura |
| `pg_port`        | Puerto de conexión a Postgres     | Nunca (fijo)          | Equipo de infraestructura |
| `pg_db`          | Nombre de la base de datos        | Solo si se crea otra  | Equipo de infraestructura |
| `pg_user`        | Usuario para entrar a Postgres    | Solo si se compromete | Equipo de infraestructura |
| `pg_password`    | Contraseña de Postgres            | Cada 90 días o si hay fuga | Equipo de infraestructura |




