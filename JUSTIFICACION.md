# Lab 2 — Justificación del `compose.yaml`

## ¿Por qué `flyway` usa `service_completed_successfully` y no `service_healthy`?

Porque Flyway no es un servicio de larga duración: es un **job que corre una vez y termina**. Su trabajo es aplicar las migraciones pendientes (`migrate`) y salir con código 0. `service_healthy` sirve para procesos que se quedan corriendo y exponen un healthcheck que dice "estoy vivo y listo" (como Postgres), pero Flyway no tiene nada que mantener sano: cuando termina, ya hizo su trabajo. La condición correcta es entonces "espera a que **termine exitosamente**". Así garantizamos que la `api` solo arranque **después** de que el schema esté completamente migrado; si Flyway falla (exit ≠ 0), la `api` ni siquiera intenta arrancar.

## ¿Por qué la api se conecta a `db:5432` y no a `localhost:5432`?

Porque cada contenedor tiene su **propio `localhost`**. Dentro del contenedor de la `api`, `localhost` apunta al contenedor de la api misma — ahí no hay ningún Postgres escuchando, así que la conexión fallaría. Compose crea una **red interna** donde cada servicio es un host resoluble **por su nombre**: el DNS interno traduce `db` a la IP del contenedor de Postgres. Por eso el `DATABASE_URL` usa `db` como host. Es lo mismo que muestra el `nginx.conf`, que proxea a `http://api:8080/` usando el nombre del servicio `api`, no `localhost`.

## ¿Qué pasa si quitas el `healthcheck` de `db`?

El `healthcheck` (`pg_isready`) es lo que le permite a Postgres reportar estado **healthy**. Si lo quitas, el servicio `db` nunca puede alcanzar ese estado, y la condición `service_healthy` de Flyway **deja de tener sentido**: en Compose moderno, depender de `service_healthy` sobre un servicio que no tiene healthcheck es un error de configuración. Sin esa señal, Compose solo sabe que el **contenedor arrancó** (`service_started`), no que la base esté realmente lista.

## ¿Por qué falla la api?

Por un problema de **timing**. Arrancar el contenedor de Postgres es instantáneo, pero el proceso adentro tarda un par de segundos en inicializar el cluster y empezar a aceptar conexiones TCP. Sin healthcheck, Compose arranca Flyway apenas el contenedor `db` existe, no cuando la base acepta conexiones. Flyway intenta conectarse a un Postgres que todavía no responde, la migración falla, y como la `api` depende de que Flyway **termine bien** (`service_completed_successfully`), la cadena entera se cae en el arranque. El healthcheck es justamente lo que convierte un "arrancó el contenedor" en un "la base **ya acepta conexiones**", que es la garantía real que la api necesita.

## ¿Qué sobrevive a `docker compose down`?

`docker compose down` para y elimina los **contenedores** (y la red), pero **conserva los volúmenes nombrados**. El volumen `pgdata` —donde Postgres guarda sus datos en `/var/lib/postgresql/data`— sobrevive. Por eso, tras un `down` seguido de un nuevo `up`, los servicios, on-call e incidentes que cargaste **siguen ahí**: el contenedor es desechable, el dato persiste en el volumen.

## ¿Y a `docker compose down -v`?

El flag `-v` agrega la eliminación de los **volúmenes**. Eso borra `pgdata` por completo: la próxima vez que levantes, Postgres arranca con un cluster vacío y Flyway vuelve a aplicar las migraciones desde cero (V1…V5), pero **sin ningún dato**. En resumen: `down` = se reinicia el cómputo, se conservan los datos; `down -v` = borrón y cuenta nueva, también de los datos.
