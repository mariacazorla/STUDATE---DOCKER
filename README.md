# Studate: despliegue

El `docker compose` que levanta [Studate](https://studateapp.blogspot.com/) entero, en local o en el
servidor: la base de datos, la API y la web. Es el proyecto del Grupo 3 de EA (María Cazorla, Diego
Soledispa y Miguel Ángel Quiroga).

Este repositorio no tiene código de aplicación: solo compone las imágenes que publican los otros dos,
como en el Seminario 10.

| Servicio | Imagen | Qué es |
|---|---|---|
| `mongodb` | `mongo:8` | La base de datos. En un volumen y **sin puerto público** |
| `api` | `ghcr.io/mariacazorla/stududate---backend` | La API de Express ([STUDUDATE---BACKEND](https://github.com/mariacazorla/STUDUDATE---BACKEND)). Sin puerto público |
| `web` | `ghcr.io/mariacazorla/studate---frontend` | nginx con la web compilada ([STUDATE---FRONTEND](https://github.com/mariacazorla/STUDATE---FRONTEND)). **El único abierto**: sirve la web y reenvía `/api` y `/socket.io` a la API |
| `seed` | la de la API | Llena la base de datos con los datos de ejemplo si está vacía, y termina |

```
navegador ──> :80 nginx (web) ──┬── /            la web (React)
                                ├── /api/        ──> api:1337 (Express)  ──> mongodb:27017
                                └── /socket.io/  ──> api:1337 (chat)
```

Desde fuera solo se ve nginx. La API y MongoDB solo existen dentro de la red de Docker `studate-net`, y
se llaman por su nombre de servicio. Al salir todo por el mismo origen no hay CORS, y la URL de la API
que lleva la web compilada es simplemente `/api`: la misma imagen sirve para cualquier servidor.

## Cómo arrancarlo

Hace falta Docker con Compose (en el servidor, Docker Engine; en Windows o Mac, Docker Desktop).

```
cp .env.example .env     # y cambia los dos secretos de los JWT y CORS_ORIGINS
docker compose up -d
```

El compose **no arranca** si faltan `JWT_SECRET`, `JWT_REFRESH_SECRET` o `CORS_ORIGINS` en el `.env`: es a
propósito, para no dejar una API con un secreto de mentira.

Después, Studate está en `http://<dirección del servidor>` (o en el puerto de `WEB_PORT`). La primera
vez, el servicio `seed` deja los datos de ejemplo; los usuarios y sus contraseñas están en el README del
backend (todos con `studate26`; `admin@example.com` para el backoffice y `laia@example.com` como estudiante).

Para ver qué pasa:

```
docker compose ps               # estado de los contenedores
docker compose logs -f api      # los logs de la API (o web, mongodb, seed)
docker compose down             # parar (los datos se quedan en el volumen)
docker compose down -v          # parar y borrar los datos
```

## Con las imágenes de ghcr.io o construidas aquí

Las imágenes las publica GitHub Actions en cada push: `:latest` desde `main` y `:develop` desde
`develop`. Con `IMAGE_TAG` en el `.env` se elige cuál se despliega. Los paquetes de ghcr.io nacen
privados: para que el servidor pueda bajarlos hay que ponerlos públicos una vez (en GitHub, Packages >
el paquete > Package settings > Change visibility), o hacer antes `docker login ghcr.io`.

Para construir las imágenes a partir del código (para probar en local lo que todavía no está subido, o
si el servidor no puede bajar de ghcr.io), se añade `compose.build.yml`, que supone los tres
repositorios clonados uno al lado del otro:

```
docker compose -f docker-compose.yml -f compose.build.yml up -d --build
```

## Qué se cambia en el .env

| Variable | Qué es | Por defecto |
|---|---|---|
| `MONGO_DB` | Nombre de la base de datos | `studate` |
| `WEB_PORT` | Puerto del servidor en el que escucha nginx | `80` |
| `CORS_ORIGINS` | Dirección pública de Studate (varias, separadas por comas). **Obligatoria** | |
| `COOKIE_SECURE` | `true` solo con HTTPS. Con http tiene que ser `false` o nadie podrá mantener la sesión | `false` |
| `JWT_SECRET`, `JWT_REFRESH_SECRET` | Secretos de los tokens, distintos entre sí. **Obligatorios** | |
| `JWT_EXPIRES_IN`, `JWT_REFRESH_EXPIRES_IN` | Cuánto dura cada token | `15m`, `7d` |
| `IMAGE_TAG` | Qué imagen se despliega: `latest` (main) o `develop` | `latest` |

La API lee su configuración al arrancar, así que para cambiar algo del `.env` basta con volver a hacer
`docker compose up -d`: Compose recrea el contenedor con los valores nuevos, sin construir nada. La web
no tiene nada que configurar: la URL de la API es `/api` y la resuelve nginx.

## Actualizar el servidor

Cuando hay una versión nueva en `main` (o en `develop`, según `IMAGE_TAG`):

```
docker compose pull
docker compose up -d
```

Compose baja las imágenes nuevas y recrea solo los contenedores que han cambiado. La base de datos no
se toca: está en el volumen `mongo-data`.

## HTTPS

Pendiente del primer despliegue. El plan es terminar el HTTPS en el mismo nginx de la web, con el
certificado montado desde fuera de la imagen (carpeta `certs/`, que no se sube al repositorio) y
`COOKIE_SECURE=true`. Qué certificado depende de si el servidor tiene un nombre de dominio o solo IP.
