# AI_LOG

Herramienta: Claude Code (Anthropic). Modelo en el Sprint 2: Claude Fable.

# Sprint 2

### 1. Primer despliegue en Docker: el compose (`feat/compose`)

- **Pregunta:** cómo componer MongoDB, la API y la web con un solo `docker compose`, en local y en el servidor, a partir de lo que hicieron en el Seminario 10.
- **Prompt:** «Según la planificación establecida previamente, vamos a proceder de esta forma. Con la estructura hecha y el login funcionando, toca el primer despliegue en Docker (bloque 2 del plan, de la 16 a la 19). Lo hacemos ya, con poco dentro, para ver si hay problemas con el servidor y no dejarlo para el final. Se hace como en el Seminario 10: te paso los repos del seminario y los datos del servidor. La conexión tiene que ir por HTTPS. A simple vista, con lo que tenemos del Seminario 10 ya tendríamos todo; si consideras que falta algo, me lo comentas. La contraseña no se escribe en ningún sitio. Hace falta el Dockerfile de la API, el de la web servida con nginx y el docker compose con MongoDB, la API y la web en el repo de despliegue. Primero que funcione en local y después en el servidor, y al final probar desde fuera que se puede iniciar sesión. Dime el plan de acción.» Después de analizar los cuatro repositorios del Seminario 10 y proponer el plan: «Continúa con la parte local» mientras se consiguen los datos de acceso al servidor.
- **Incoherencias:**
  - Los repositorios del Seminario 10 no despliegan en ningún servidor ni usan HTTPS: cada uno construye su imagen y la publica en ghcr.io, y el compose las junta en local con la API y cada frontend en un puerto distinto y CORS `*`.
  - En el seminario, la URL de la API va compilada dentro de la imagen del frontend (`http://localhost:1337`): para apuntar a otro servidor hay que cambiar el código y reconstruirla.
  - El frontend del seminario usa nginx con la configuración por defecto, y su propio README avisa de que recargar una ruta interna de la SPA da 404.
  - El seed del seminario no va en la imagen, porque lo ejecutan con `tsx`: para llenar la base de datos del servidor hay que tener el código allí.
  - De los datos del servidor se tenían dos direcciones IP y una contraseña, sin usuario; con la VPN de la universidad, solo una de las IP responde (por SSH) y falta el usuario para entrar.
- **Solución:**
  - Un solo origen: nginx (la imagen de la web) sirve la web y reenvía `/api` y `/socket.io` a la API por la red interna de Docker. Así no hay CORS, la URL compilada en la web es `/api` (vale para cualquier servidor) y la cookie del refresh token vive en `/api/auth`, que es para lo que el backend tiene `REFRESH_COOKIE_PATH`. Un `nginx.conf` propio devuelve `index.html` en las rutas de la SPA.
  - El seed va compilado dentro de la imagen de la API, y el compose tiene un servicio `seed` que lo ejecuta en cada arranque (solo inserta si la base de datos está vacía).
  - `docker-compose.yml` con MongoDB en un volumen y sin puerto público, la API sin puerto público, la web en el 80 y las imágenes de ghcr.io (`IMAGE_TAG` elige `latest` o `develop`); `compose.build.yml` para construirlas a partir del código. El compose se niega a arrancar sin los secretos de los JWT ni `CORS_ORIGINS`.
  - HTTPS en dos pasos: primero HTTP con `COOKIE_SECURE=false` para ver los problemas del servidor cuanto antes; el HTTPS, en el mismo nginx con el certificado montado desde fuera, cuando se sepa si hay dominio o solo IP.
  - Probado en local con `compose.build.yml`: los cuatro servicios arrancan, el seed deja los datos, recargar en `/dates` y en `/admin/usuarios` da 200, `/api/ping` responde a través de nginx, el login de una estudiante entra en La Orla y al recargar la sesión se mantiene (el refresh pasa por `/api/auth` con la cookie), y el puerto de MongoDB no está abierto hacia fuera.

### 2. HTTPS preparado y guion de despliegue (`feat/https`)

- **Pregunta:** cómo dejar el HTTPS y los pasos del servidor listos sin tener todavía acceso al servidor.
- **Prompt:** «¿Seguro que no hay nada [sobre la conexión con el servidor] en los repositorios del Seminario 10? Hasta que me respondan, ¿qué?» y, tras la búsqueda: «Haz eso y sigue. Deja esto preparado y listo. De momento continuamos en local; ya lo acabaremos subiendo al servidor cuando podamos.»
- **Incoherencias:**
  - Se revisó el historial completo de los cinco repositorios del Seminario 10 (ramas, commits borrados, mensajes, issues, Actions y las imágenes publicadas en ghcr.io): ninguno llegó a desplegar en un servidor; todas las imágenes llevan `http://localhost:1337`. El servidor con HTTPS es un paso nuevo que el profesor da en clase, y falta el usuario de SSH.
  - Si el server de HTTPS fuera a la imagen de la web, nginx no arrancaría sin certificado, y sin dominio solo hay certificados autofirmados, que Expo Go no acepta.
- **Solución:**
  - Las reglas de nginx de la web se separan en un fichero (`studate-app.conf`, dentro de la imagen) que incluyen dos servers: el de http (en la imagen) y el de https (`nginx/https.conf`, en este repo). `compose.https.yml` monta ese server y la carpeta `certs/` (fuera de git) y abre el 443; el 80 redirige a https. Así la imagen es la misma con y sin HTTPS y se decide en el despliegue.
  - README con el guion del servidor paso a paso (Docker, puertos, clonar, `.env`, imágenes, arrancar, comprobar desde dentro y desde fuera, MongoDB cerrado) y los dos casos de certificado (autofirmado con IP o Let's Encrypt con dominio).
  - Probado en local con un certificado autofirmado: http redirige con 301, la web y `/api/ping` responden por https con TLS 1.3 y HTTP/2, el login deja la cookie con `Secure` y esa cookie renueva la sesión.
