# Lab 2 — Justificación de decisiones (docker-compose)

## 1. ¿Por qué `flyway` usa `service_completed_successfully` y no `service_healthy`?

Flyway no es un servicio de larga duración: es un *job* que corre una vez, aplica las migraciones pendientes (`migrate`) y sale con código 0. `service_healthy` está pensado para procesos que se quedan vivos y exponen un `healthcheck` que dice "estoy disponible" (como Postgres), pero eso solo valida disponibilidad, no que la migración haya terminado ni si terminó bien o con errores. Es más: como Flyway termina, ese estado "sano" nunca llegaría y la dependencia se quedaría esperando. Por eso usamos `service_completed_successfully`: esperamos a que el *job* termine correctamente. Así garantizamos que la API arranque solo después de que el schema, las relaciones y los datos iniciales estén creados; y si Flyway falla (exit ≠ 0), la API ni siquiera intenta arrancar.

## 2. ¿Por qué la API se conecta a `db:5432` y no a `localhost:5432`?

Porque cada contenedor tiene su propio `localhost`: dentro del contenedor de la API, `localhost` apunta a la API misma, y ahí no hay ningún Postgres escuchando, así que la conexión fallaría. Compose crea una red interna donde cada servicio es un host resoluble por su nombre; el DNS interno traduce `db` a la IP del contenedor de Postgres, y `5432` es el puerto que Postgres expone dentro de esa red. Por eso el `DATABASE_URL` usa `db` como host. Es el mismo patrón que muestra el `nginx.conf`, que proxea a `http://api:8080/` usando el nombre del servicio `api`, no `localhost`. Además, esa `DATABASE_URL` va en el environment (Factor III): la app no sabe dónde está la base, se la indicamos nosotros desde el compose.

## 3. ¿Qué pasa si quitas el `healthcheck` de `db`? ¿Por qué falla la api?

El `healthcheck` (`pg_isready`) es lo que le permite a Postgres reportar el estado `healthy`. Si lo quitamos, el servicio `db` nunca puede alcanzar ese estado, y la condición `service_healthy` de Flyway deja de tener sentido: en Compose moderno, depender de `service_healthy` sobre un servicio sin `healthcheck` es un error de configuración, y Compose lo rechaza de entrada.

El `healthcheck` existe para resolver un problema de timing. Arrancar el contenedor de Postgres es instantáneo, pero el proceso adentro tarda un par de segundos en inicializar el cluster y empezar a aceptar conexiones TCP. Sin esa señal, lo único que Compose podría saber es que el contenedor existe (`service_started`), no que la base ya acepte conexiones. Flyway intentaría migrar contra un Postgres que todavía no responde, la migración fallaría, y como la api depende de que Flyway termine bien (`service_completed_successfully`), la cadena entera se caería en el arranque. El `healthcheck` es justamente lo que convierte un "arrancó el contenedor" en un "la base ya acepta conexiones", que es la garantía real que la api necesita.

## 4. ¿Qué sobrevive a `docker compose down`? ¿Y a `down -v`?

`down` para y elimina los contenedores y la red que creó Compose, pero conserva los volúmenes nombrados. El volumen `pgdata` —donde Postgres guarda sus datos en `/var/lib/postgresql/data`— sobrevive. Por eso, tras un `down` seguido de un nuevo `up`, los servicios, on-call e incidentes que cargamos siguen ahí: el contenedor es desechable, el dato persiste en el volumen.

El flag `-v` agrega la eliminación de los volúmenes nombrados, así que borra `pgdata` por completo. La próxima vez que levantemos, Postgres arranca con un cluster vacío y Flyway vuelve a aplicar las migraciones desde cero (V1…V5): las tablas se recrean, pero sin ningún dato. En resumen: `down` conserva los datos; `down -v` es como un borrón y cuenta nueva.
