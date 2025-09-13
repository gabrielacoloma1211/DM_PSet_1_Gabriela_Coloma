***Volumetría por tramo***
<img width="1117" height="506" alt="Screenshot 2025-09-12 at 11 05 50 PM" src="https://github.com/user-attachments/assets/cdc948c1-174f-407e-8333-bec7dd716bbd" />


***Idempotencia***
Se da porque en el exporter usamos UPSERT, que garantiza que si un registro con la misma clave primaria (id) ya existe en la tabla, no se inserta un duplicado sino que se actualiza el registro existente.

De esta forma:

Si el pipeline se ejecuta dos o más veces con el mismo rango de datos, la cantidad de registros en la tabla no aumenta.

Se evita la duplicación de información.

Siempre se conserva la última versión del payload asociada al mismo id.

Esto asegura que el proceso es idempotente, ya que múltiples ejecuciones producen el mismo estado final en la base de datos.

Datos después de segunda corrida: 

<img width="472" height="381" alt="Screenshot 2025-09-12 at 11 08 09 PM" src="https://github.com/user-attachments/assets/94bf316f-6bf2-4288-b17c-a647df3248cd" />

Vemos que solo hay uno y no hay duplicados

