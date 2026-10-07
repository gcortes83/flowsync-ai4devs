# Prompts

---

## Prompt 1

**Modelo:** Opus 5.5
**Herramienta:** Claude Code

```
Haz la migración del proyecto de SQLite a PostgreSQL corriendo en Docker la aplicación, con dos bases de datos un para desarrollo y otra para pruebas.
Restricciones:
  El fichero de Compose se llama compose.yaml y los dos servicios se llaman db y db-test.
  La imagen es pgvector/pgvector:pg17. Es la imagen oficial de PostgreSQL con la extensión de vectores ya dentro.
  Los puertos son 54410 para desarrollo y 54411 para pruebas. No el 5432: quien tenga un PostgreSQL suyo levantado se lo encontraría ocupado, y el error que vería no menciona a Docker por ningún lado.
  La base de pruebas va en memoria, sin volumen. Es efímera a propósito: una batería de pruebas que depende de lo que dejó la anterior no es una batería de pruebas.
  Los dos servicios llevan comprobación de salud, y el arranque espera a que estén sanos. La propia imagen avisa de que, la primera vez, crea la base y no acepta conexiones mientras tanto, y de que eso rompe a quien levanta varios contenedores a la vez.
  Sin la clave version: en el fichero de Compose: está obsoleta y Docker imprime un aviso.
  La batería de pruebas apunta a la otra base por su propio fichero de entorno, que el framework carga solo cuando el entorno es de pruebas.
  Y deja atajos en el Makefile para levantar las bases, pararlas, migrar las dos y correr las pruebas.
```
**Qué salió:**

⏺ La base de datos ya corre en PostgreSQL dentro de Docker, con una base para desarrollo y otra para pruebas. Las 23 pruebas pasan contra la base de pruebas, y está commiteado en motor-gc como 7b25efa.

Qué monté:
- compose.yaml: dos servicios, db en el puerto 54410 y db-test en el 54411, los dos con pgvector/pgvector:pg17. No lleva la clave version:.
  - db guarda sus datos en un volumen.
  - db-test guarda sus datos solo en memoria, sin volumen.
  - Los dos tienen comprobación de salud. Conecta por red a propósito: mientras la imagen crea la base por primera vez solo acepta conexiones por el socket local, así que una comprobación a secas daría «sano» antes de tiempo.
- Backend: config/database.ts usa ahora el driver pg, que sustituye a better-sqlite3, y lee DB_* del entorno. start/env.ts exige esas variables al arrancar.
  - .env.example apunta al puerto 54410.
  - .env.test apunta al puerto 54411. El framework solo lo carga cuando NODE_ENV=test, y tiene prioridad sobre .env.
- Makefile:
  - make db-up espera a que las dos bases estén sanas.
  - make db-down las para.
  - make migrate migra las dos.
  - make test levanta las bases, migra la de pruebas y luego corre las pruebas.
  - setup y start levantan ahora las bases primero.

Qué comprobé:
- Después de las pruebas, la base de desarrollo seguía con 0 usuarios, así que las pruebas no la tocaron.
- Tras make db-down y make db-up, la base de pruebas volvió vacía y la de desarrollo conservó sus tablas.
- Arrancando desde cero, las dos migraciones funcionaron a la primera, sin «connection refused».
- No se cambió ninguna migración, npm run typecheck pasa, y el backend/database/schema.ts regenerado es idéntico al de antes.

Lo que conviene saber:
- Añadí las variables DB_* a tu backend/.env local. Ese fichero no se commitea. Quien tenga un .env antiguo tiene que añadirlas a mano: make env no las añade, y sin ellas el backend no arranca.
- make clean ahora también borra el volumen de datos de desarrollo (docker compose down -v), igual que antes borraba el fichero de SQLite.
- Dejé prompts.md y frontend/package-lock.json fuera del commit porque ya estaban modificados antes de empezar.
- Actualicé CLAUDE.md, que todavía describía SQLite.
- No hice nada de la parte B ni de HALLAZGOS.md. Un diff vacío de schema.ts es justo el caso del que avisa el ejercicio, así que es por donde empezar a mirar.
