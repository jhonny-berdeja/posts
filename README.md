# posts

## ¿Para qué es este proyecto?

Este repositorio es la base del servicio de posts del ecosistema jtagram, construido con NestJS. A diferencia de otros repos del ecosistema, todavía es un scaffold recién generado por el Nest CLI: no tiene ninguna funcionalidad de negocio implementada. `src/` contiene únicamente el `AppController` y el `AppService` por defecto (el endpoint `GET /` que devuelve `"Hello World!"`), sin módulos propios del dominio de posts.

## ¿Qué hace cada módulo?

Todavía no existen módulos propios de negocio. El único módulo presente es el `AppModule` por defecto que registra el `AppController` y el `AppService` generados por el scaffold de Nest. No hay carpeta `src/modules/` ni ninguna otra división funcional por el momento.

## ¿Qué variables de entorno necesito?

### Variables para el pipeline de GitHub Actions

Este repo no separa release y deploy: `.github/workflows/deploy.yml` es un único workflow que se dispara con push a `master` y hace build, push de la imagen y deploy en un solo job encadenado. Las variables/secretos que usa son:

- **`DOCKERHUB_USERNAME` y `DOCKERHUB_TOKEN`**: se usan juntas para autenticar contra Docker Hub (`docker/login-action`) y publicar la imagen `posts` en cada deploy. `DOCKERHUB_USERNAME` es el usuario u organización de Docker Hub bajo la que se publica la imagen; `DOCKERHUB_TOKEN` es un access token generado desde Account Settings > Security > New Access Token en Docker Hub, con permisos de lectura y escritura. Detalle completo en `.github/workflows/obtain-secrets.md`.
- **`TS_OAUTH_CLIENT_ID` y `TS_OAUTH_SECRET`**: se usan juntas para unir el runner a la tailnet (`tailscale/github-action`) y así poder llegar al API server de microk8s en pcbox. Se obtienen creando un cliente OAuth en la consola de administración de Tailscale, con scope `Devices: Write` y el tag `tag:continuous-integration`.
- **`KUBECONFIG_PCBOX`**: contenido crudo (no en base64) del kubeconfig usado para autenticar `kubectl` contra el cluster de microk8s en pcbox. El workflow lo escribe directo a `~/.kube/config`. El procedimiento para generarlo, incluyendo el ajuste del certificado del API server y del campo `server:` para apuntar a la IP de Tailscale, está documentado paso a paso en `.github/workflows/obtain-secrets.md`.

El archivo `.github/workflows/env.secrets.example` lista estas mismas variables como referencia rápida para cargarlas como secretos del repositorio en GitHub (Settings > Secrets and variables > Actions).

### Variables para el funcionamiento de la app

Por ahora la app en runtime solo lee `PORT` (en `src/main.ts`, con fallback a `3000` si no está definida). No hay ningún archivo de ejemplo de variables de entorno de aplicación (`.env.example` o similar) en el repo, ni un `ConfigModule` configurado en `app.module.ts`: al no existir todavía funcionalidad de negocio, tampoco hay necesidad de configuración adicional.

## ¿Cómo se ejecuta la app?

A diferencia de otros repos del ecosistema, `posts` no tiene un workflow manual con inputs de tags ni pasa por `infra-hub`. El deploy es automático: cada push a `master` dispara `.github/workflows/deploy.yml`, que en un solo workflow:

1. Construye la imagen Docker y la publica en Docker Hub, taggeada con el SHA del commit.
2. Se conecta al cluster de microk8s en pcbox uniendo el runner a la tailnet vía Tailscale.
3. Aplica los manifiestos de `kubernetes/manifests/` (reemplazando el usuario de Docker Hub y el tag de imagen) y espera a que el rollout del deployment `posts` termine correctamente.

No hay paso de aprobación ni selección de versión: el flujo completo, de commit a pod corriendo en pcbox, ocurre sin intervención manual.
