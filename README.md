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
| `qb_client_secret` | Llave secreta de la app QBO     | Solo si cambia la app | Equipo de data |
| `qb_refresh`     | Token para pedir nuevos accesos a QBO | Cada 101 días (por QBO) | Equipo de data |
| `qb_realm`       | ID de la empresa en QBO           | Nunca cambia          | Equipo de data |
| `pg_host`        | Dirección del servidor Postgres   | Solo si cambia infra  | Equipo de infraestructura |
| `pg_port`        | Puerto de conexión a Postgres     | Nunca (fijo)          | Equipo de infraestructura |
| `pg_db`          | Nombre de la base de datos        | Solo si se crea otra  | Equipo de infraestructura |
| `pg_user`        | Usuario para entrar a Postgres    | Solo si se compromete | Equipo de infraestructura |
| `pg_password`    | Contraseña de Postgres            | Cada 90 días o si hay fuga | Equipo de infraestructura |

***Pipelines***
Existen 3 pipelines, uno para cada entidad. Es decir hay uno para Invoice, Items y Customers.

Parámetros: 
- fecha_inicio (UTC)
- fecha_fin (UTC)
- page_size (default: 100)

Segmentación: 
- Se divide en chunks de 7 días
- La paginación se realiza con las variables startposition y maxresults. Se suma al starposition el page_size para saltar al siguiente segmento.

Límites y reintentos:
- Manejo el HTTP 429 que es "Too Many Requests" y con esto sabemos que sobrepaso el límite de llamadas seguidas y usamos backoff exponencial.
- Reintentos hasta 5 veces antes de fallar y backoff_time de 2 segundos.

Runbook:
- Ejecutar pipeline con rango deseado (fecha_inicio, fecha_fin)
- Si falla:
      - Revisar logs en Mage
      - Reintentar ejecución (idempotente, entonces no duplica registros)
      - Si el fallo es por auth, refrescar los tokens en Mage Secrets

***Trigger one-time***

Configurados para las tres entidades:
- trigger_invoice
- trigger_customers
- trigger_items

Frecuencia: @once
Ejecución registrada: 2025-09-12 (UTC)
Equivalencia Guayaquil: UTC -5 → 17:00 hora local ≈ 22:00 UTC
Política: deshabilitados automáticamente tras ejecución (completed).

***Esquema RAW***
Cada entidad se almacena en su tabla respectivo en el esquema raw.
Existen tres tablas:
- raw.qb_invoices
- raw.qb_customers
- raw.qb_items



Las tablas tienen lo siguiente:

id (TEXT, PRIMARY KEY): Identificador único de cada registro (el Id que viene de QBO).

payload (JSONB, NOT NULL): Objeto completo en formato JSON con todos los datos originales del registro.

ingested_at_utc (TIMESTAMPTZ, NOT NULL): Momento en que el registro fue insertado en la base (timestamp en UTC).

extract_window_start_utc (TIMESTAMPTZ, NOT NULL): Fecha/hora de inicio del rango de extracción (en UTC).

extract_window_end_utc (TIMESTAMPTZ, NOT NULL): Fecha/hora de fin del rango de extracción (en UTC).

page_number (INT, NOT NULL): Número de página en la paginación de la API.

page_size (INT, NOT NULL): Cantidad de registros solicitados por página.

request_payload (JSONB): Parámetros de la request usada en la extracción (útil para trazabilidad y debugging).



Características

PK: id

Payload: JSONB completo de QBO

Metadatos obligatorios: ventana, timestamps, paginación, payload de request.

Idempotencia: garantizada con ON CONFLICT (id) DO UPDATE.



***Validaciones/Volumetría***
Durante la ejecución, el pipeline imprime mensajes (print) que muestran cuántas filas se cargaron, el número de IDs únicos y el rango de fechas procesado.

Estas impresiones sirven como validación rápida para confirmar que los datos fueron extraídos correctamente y sin duplicados.

La idempotencia se asegura en la capa de exportación: el exporter usa UPSERT, lo que significa que si se vuelve a procesar un mismo tramo, no se generan duplicados, sino que los registros existentes se actualizan.

Para validar:

Una reejecución de un mismo rango de fechas debe dar el mismo número de filas.

El conteo de IDs únicos debe coincidir con el total de filas, garantizando que no hay duplicados.

Revisando los campos extract_window_start_utc y extract_window_end_utc se pueden detectar huecos o solapamientos en la extracción histórica.



***Troubleshooting***
Auth (401 / 403): refresh token expirado → actualizar en Mage Secrets.

Paginación: revisar startposition y maxresults.

Rate limits (429): esperar y dejar que backoff lo maneje.

Timezones: siempre usar UTC en parámetros y metadatos.

Permisos: verificar usuario Postgres y credenciales en Mage Secrets.

***Checklist de aceptación***

- [x] Mage y Postgres se comunican por nombre de servicio.  
- [x] Todos los secretos (QBO y Postgres) están en Mage Secrets; no hay secretos en el repo/entorno expuesto.  
- [x] Pipelines `qb_<entidad>_backfill` aceptan `fecha_inicio` y `fecha_fin` (UTC) y segmentan el rango.  
- [x] Trigger one-time configurado, ejecutado y luego deshabilitado/marcado como completado.  
- [x] Esquema `raw` con tablas por entidad, payload completo y metadatos obligatorios.  
- [x] Idempotencia verificada: reejecución de un tramo no genera duplicados.  
- [x] Paginación y rate limits manejados y documentados.  
- [x] Volumetría y validaciones mínimas registradas y archivadas como evidencia.  
- [x] Runbook de reanudación y reintentos disponible y seguido.


***Evidencias***
Se encuentran en la carpeta evidencias/ con:
- Configuración de Mage Secrets (nombres visibles, valores ocultos).
- Triggers one-time configurados y ejecutados (completed).
- Tablas raw con registros en pgAdmin.
- Logs de volumetría y evidencia de idempotencia.


