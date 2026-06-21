# Lab 2 — Justificación de decisiones (docker-compose)

## 1. ¿Por qué `flyway` usa `service_completed_successfully` y no `service_healthy`?

Porque Flyway no es un servicio que queda corriendo: migra y muere (sale con código 0). Un `healthcheck` sirve para algo que se mantiene vivo, no para un *job* que termina. Por eso espero a que termine bien, no a que esté "sano".

## 2. ¿Por qué la API se conecta a `db:5432` y no a `localhost:5432`?

Porque dentro del contenedor de la API, `localhost` es ella misma, no la base. En la red de Compose cada servicio se resuelve por su nombre, así que uso `db`. Además, la `DATABASE_URL` va en el environment (Factor III): la app no sabe dónde está la base, se lo digo yo.

## 3. ¿Qué pasa si quito el `healthcheck` de `db`?

Que `service_healthy` se queda sin nada en qué basarse y Docker da por "lista" la base apenas arranca el contenedor, aunque Postgres todavía no acepte conexiones. Flyway intenta migrar contra una base que no responde y falla; y como la API depende de Flyway, se cae toda la cadena. El `healthcheck` (`pg_isready`) es lo que distingue "el contenedor existe" de "la base está operativa".

## 4. ¿Qué sobrevive a `down`? ¿Y a `down -v`?

A `down` sobrevive el volumen `pgdata` con todos los datos (servicios, on-call, incidentes); borra solo contenedores y red. A `down -v` se borra también el volumen, así que la base arranca vacía y Flyway vuelve a migrar de cero. El contenedor es desechable; el volumen es lo que persiste.
