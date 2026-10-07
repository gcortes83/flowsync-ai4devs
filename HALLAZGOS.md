# Hallazgos

Aquí van **las tres líneas** del ejercicio, una por cada punto de abajo. Es lo único que hay que
traer hecho: un cambio de motor a medias con estas tres líneas escritas vale más que lo contrario,
porque lo que se discute en el directo es dónde te chocaste.

Escribe **una sola línea por punto**, con tus palabras y con lo que mediste, no con lo que suponías.

## 1. Las filas que cambian y la rama

Cuántas filas cambian de valor en tu cambio de esquema, medido con una consulta, y en qué rama del
árbol de reversibilidad cae. Si tu migración no toca datos, dilo tal cual: también es una respuesta.

- 0 filas: la migración no toca datos. Ninguna migración se modificó, y en `backend/tmp/db.sqlite3` `select count(*) from users` / `from tasks` daban 0 y 0, así que no había nada que trasladar. Cae en la rama reversible sin coste: volver atrás es devolver la conexión a SQLite. Lo que sí cambió no es el valor guardado (en PostgreSQL `due_date` es `2020-01-01` de tipo `date`) sino cómo se lee.

## 2. Lo que la batería de pruebas no podía ver

Una cosa que la batería de pruebas no podía ver. Si no encontraste ninguna, escribe qué buscaste y dónde.

- `Task.isOverdueOn` (`backend/app/models/task.ts`) ya no detecta ninguna tarea vencida. El driver `pg` devuelve la columna `date` como un `Date` de JS, `schema_rules.ts` la sigue declarando `string` (por eso `typecheck` está en verde y el diff de `database/schema.ts` sale vacío), y `Date < '2026-10-07'` es siempre `false`. El comentario «se compara texto contra texto» ya no es cierto. Lo comprobé con una tarea con `dueDate` `2020-01-01` y `today=2026-10-07`: el `PUT /tasks/1/due-date` responde `isOverdue: true` porque usa el valor en memoria, pero `GET /tasks/1` responde `isOverdue: false` y `dueDate: "2020-01-01T00:00:00.000Z"`. Ningún test toca `dueDate` ni `isOverdue`.

## 3. Tu duda

De qué dudaste, o qué no pudiste comprobar.

- Dudo del huso horario: `pg` interpreta el `date` como medianoche local del proceso. Con el `TZ=UTC` del `.env` sale `T00:00:00.000Z`, pero en una máquina en UTC-6 sale `T06:00:00.000Z`, y al este de UTC debería salir el día anterior. Esto último no lo probé. Tampoco comprobé si la solución correcta es registrar un parser de tipo para OID 1082 que devuelva texto o convertirlo en el modelo, y la dejo sin aplicar.
