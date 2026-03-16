# Informe de Comandos Docker en Kamal

Este documento cataloga todos los comandos Docker que Kamal ejecuta en servidores remotos,
incluyendo sus parámetros exactos y su correspondencia con la documentación oficial de Docker.

---

## Tabla de Contenidos

1. [Arquitectura de Ejecución](#arquitectura-de-ejecución)
2. [Comandos de Instalación y Setup](#1-instalación-y-setup-de-docker)
3. [Comandos de Red](#2-red-docker-network)
4. [Comandos de Registro](#3-registro-docker-loginlogout)
5. [Comandos de Contenedores - App](#4-gestión-de-contenedores---aplicación)
6. [Comandos de Contenedores - Accessory](#5-gestión-de-contenedores---accessories)
7. [Comandos de Contenedores - Proxy](#6-gestión-de-contenedores---proxy)
8. [Comandos de Ejecución en Contenedores](#7-ejecución-de-comandos-en-contenedores)
9. [Comandos de Logs](#8-logs)
10. [Comandos de Imágenes](#9-gestión-de-imágenes)
11. [Comandos de Build (Buildx)](#10-build-con-buildx)
12. [Comandos de Contextos Docker](#11-contextos-docker)
13. [Comandos de Limpieza/Pruning](#12-limpieza-y-pruning)
14. [Comandos de Inspección y Diagnóstico](#13-inspección-y-diagnóstico)
15. [Resumen de Flags Utilizados](#resumen-de-flags-utilizados)

---

## Arquitectura de Ejecución

Kamal construye comandos Docker mediante un patrón builder en Ruby. Todos los comandos
se generan como arrays en `Kamal::Commands::Base` (`lib/kamal/commands/base.rb:83-85`):

```ruby
def docker(*args)
  args.compact.unshift :docker
end
```

Estos arrays se ejecutan remotamente vía SSH usando SSHKit, convirtiéndose en cadenas de texto
como `docker run --detach --name myapp ...`.

---

## 1. Instalación y Setup de Docker

**Archivo:** `lib/kamal/commands/docker.rb`

### `docker -v`
- **Línea:** 9
- **Método:** `installed?`
- **Propósito en Kamal:** Verificar si Docker CLI está instalado en el servidor.
- **Documentación oficial:** Muestra la versión del cliente Docker instalado. Equivale a `docker version --format '{{.Client.Version}}'` pero más simple. Si Docker no está instalado, el comando falla.

### `docker version`
- **Línea:** 14
- **Método:** `running?`
- **Propósito en Kamal:** Verificar que el daemon Docker está activo y respondiendo.
- **Documentación oficial:** Muestra información detallada de versión tanto del cliente como del servidor Docker. A diferencia de `docker -v`, este comando requiere que el daemon esté corriendo ya que consulta al servidor.

---

## 2. Red (docker network)

**Archivo:** `lib/kamal/commands/docker.rb`

### `docker network create kamal`
- **Línea:** 39
- **Método:** `create_network`
- **Propósito en Kamal:** Crear la red `kamal` que conecta todos los contenedores (app, proxy, accessories) entre sí.
- **Documentación oficial:** `docker network create` crea una nueva red Docker. Sin especificar `--driver`, usa el driver `bridge` por defecto, que permite comunicación entre contenedores en el mismo host. Los contenedores conectados a la misma red pueden comunicarse usando sus nombres como hostnames.
- **Parámetros:**
  - `kamal` — nombre de la red

---

## 3. Registro (docker login/logout)

**Archivo:** `lib/kamal/commands/registry.rb`

### `docker login <server> -u <username> -p <password>`
- **Línea:** 7
- **Método:** `login`
- **Propósito en Kamal:** Autenticarse en el registro Docker (Docker Hub, GHCR, ECR, etc.) para poder hacer push/pull de imágenes.
- **Documentación oficial:** `docker login` autentica al usuario contra un registro Docker. Las credenciales se almacenan en `~/.docker/config.json`.
- **Parámetros:**
  - `<server>` — URL del servidor de registro (ej: `ghcr.io`)
  - `-u <username>` — nombre de usuario
  - `-p <password>` — contraseña o token de acceso (Kamal lo marca como `sensitive`)
- **Nota de seguridad:** Pasar `-p` en línea de comandos puede exponer la contraseña en el historial del shell. Docker recomienda `--password-stdin`. Kamal mitiga esto usando SSHKit que no guarda historial, pero las credenciales podrían aparecer en logs del proceso.

### `docker logout <server>`
- **Línea:** 16
- **Método:** `logout`
- **Propósito en Kamal:** Cerrar sesión del registro Docker.
- **Documentación oficial:** `docker logout` elimina las credenciales almacenadas para el registro especificado de `~/.docker/config.json`.
- **Parámetros:**
  - `<server>` — URL del servidor de registro

### `docker start kamal-docker-registry`
- **Línea:** 23
- **Método:** `setup` (primer intento)
- **Propósito en Kamal:** Intentar arrancar un registro local ya existente.
- **Documentación oficial:** `docker start` inicia uno o más contenedores detenidos.

### `docker run --detach -p 127.0.0.1:<port>:5000 --name kamal-docker-registry registry:3`
- **Línea:** 24
- **Método:** `setup` (si start falla, con `||`)
- **Propósito en Kamal:** Crear y ejecutar un registro Docker local para builds sin push a registro remoto.
- **Parámetros:**
  - `--detach` — ejecutar en segundo plano
  - `-p 127.0.0.1:<port>:5000` — mapear puerto solo en localhost (seguridad)
  - `--name kamal-docker-registry` — nombre fijo del contenedor
  - `registry:3` — imagen oficial de Docker Registry v3

### `docker stop kamal-docker-registry`
- **Línea:** 30
- **Método:** `remove` (paso 1)
- **Propósito en Kamal:** Detener el registro local.

### `docker rm kamal-docker-registry`
- **Línea:** 31
- **Método:** `remove` (paso 2)
- **Propósito en Kamal:** Eliminar el contenedor del registro local.

---

## 4. Gestión de Contenedores - Aplicación

**Archivo:** `lib/kamal/commands/app.rb`

### `docker run` (contenedor de aplicación)
- **Línea:** 17-35
- **Método:** `run`
- **Propósito en Kamal:** Crear y ejecutar el contenedor principal de la aplicación.
- **Comando completo:**
  ```
  docker run \
    --detach \
    --restart unless-stopped \
    --name <service>-<role>-<destination>-<version> \
    --network kamal \
    [--hostname <hostname>] \
    --env KAMAL_CONTAINER_NAME="<name>" \
    --env KAMAL_VERSION="<version>" \
    --env KAMAL_HOST="<host>" \
    [--env KAMAL_DESTINATION="<dest>"] \
    [--env <key>=<value> ...] \
    [--log-driver <driver> --log-opt <opts>] \
    [--volume <host>:<container> ...] \
    [--label <key>=<value> ...] \
    [opciones adicionales del role ...] \
    <image> \
    [<cmd>]
  ```
- **Documentación oficial de cada flag:**
  - `--detach` (`-d`) — Ejecutar el contenedor en segundo plano y devolver el ID.
  - `--restart unless-stopped` — Política de reinicio: reiniciar siempre excepto cuando se detiene manualmente. Sobrevive reinicios del daemon y del host.
  - `--name` — Asignar nombre al contenedor. Debe ser único en el host.
  - `--network kamal` — Conectar el contenedor a la red `kamal`. Permite comunicación con proxy y accessories.
  - `--hostname` — Establecer el hostname dentro del contenedor.
  - `--env` (`-e`) — Establecer variables de entorno. Kamal inyecta automáticamente `KAMAL_CONTAINER_NAME`, `KAMAL_VERSION`, `KAMAL_HOST` y opcionalmente `KAMAL_DESTINATION`.
  - `--log-driver` / `--log-opt` — Configurar driver de logs (ej: `json-file`, `syslog`).
  - `--volume` (`-v`) — Montar volúmenes del host en el contenedor.
  - `--label` (`-l`) — Añadir metadatos como labels. Kamal usa labels `service`, `role` y `destination` para filtrar contenedores.

### `docker start <container_name>`
- **Línea:** 38
- **Método:** `start`
- **Propósito en Kamal:** Reiniciar un contenedor de app que fue detenido.
- **Documentación oficial:** `docker start` inicia contenedores previamente creados/detenidos. A diferencia de `docker run`, no crea un nuevo contenedor.

### `docker stop <container_name>`
- **Línea:** 48
- **Método:** `stop`
- **Propósito en Kamal:** Detener el contenedor de la app de forma elegante (envía SIGTERM, espera grace period, luego SIGKILL).
- **Parámetros adicionales:** Acepta argumentos de `role.stop_args` que pueden incluir `--time <seconds>` para ajustar el grace period.
- **Documentación oficial:** `docker stop` envía señal SIGTERM al proceso principal, espera un período de gracia (10s por defecto), y envía SIGKILL si no se detiene.

### `docker ps` (listar contenedores de app)
- **Línea:** 52, 72, 96
- **Método:** `info`, `list_versions`, `latest_container`
- **Propósito en Kamal:** Listar contenedores del servicio, obtener versiones activas.
- **Variantes usadas:**
  ```
  docker ps --filter label=service=<name> --filter label=destination=<dest> --filter label=role=<role>
  docker ps --filter ... --format "{{.Names}}"
  docker ps --latest --quiet --filter ... --filter status=running --filter status=restarting
  docker ps --latest --format '{{.Names}}' --filter ... --filter ancestor=$(<image_id>)
  ```
- **Documentación oficial de flags:**
  - `--filter` (`-f`) — Filtrar contenedores por condiciones. Soporta: `label=`, `status=`, `name=`, `ancestor=`.
  - `--format` — Formato de salida usando plantillas Go (`{{.Names}}`, `{{.ID}}`).
  - `--latest` (`-l`) — Mostrar solo el último contenedor creado.
  - `--quiet` (`-q`) — Mostrar solo IDs numéricos.
  - `--all` (`-a`) — Mostrar todos los contenedores (no solo los activos).

---

## 5. Gestión de Contenedores - Accessories

**Archivo:** `lib/kamal/commands/accessory.rb`

### `docker run` (accessory)
- **Línea:** 16-30
- **Método:** `run`
- **Propósito en Kamal:** Crear y ejecutar contenedores de servicios auxiliares (bases de datos, Redis, etc.).
- **Comando:**
  ```
  docker run \
    --name <service_name> \
    --detach \
    --restart unless-stopped \
    [--network kamal] \
    [--log-driver ... --log-opt ...] \
    [-p <host_port>:<container_port> ...] \
    [--env KAMAL_HOST="<host>"] \
    [--env <key>=<value> ...] \
    [--volume <host>:<container> ...] \
    [--label <key>=<value> ...] \
    [opciones adicionales ...] \
    <image> \
    [<cmd>]
  ```
- **Flags adicionales vs App:**
  - `-p` / `--publish` — Publicar puertos del contenedor al host. Los accessories pueden exponer puertos directamente.

### `docker container start <service_name>`
- **Línea:** 33
- **Método:** `start`

### `docker container stop <service_name>`
- **Línea:** 37
- **Método:** `stop`

### `docker ps [-a] [-q] --filter label=service=<name>`
- **Línea:** 41
- **Método:** `info`

### `docker image pull <image>`
- **Línea:** 95
- **Método:** `pull_image`
- **Propósito en Kamal:** Descargar la imagen del accessory desde el registro.
- **Documentación oficial:** `docker image pull` (equivalente a `docker pull`) descarga una imagen o repositorio de un registro.

### `docker image rm --force <image>`
- **Línea:** 107
- **Método:** `remove_image`
- **Propósito en Kamal:** Eliminar la imagen del accessory del servidor.
- **Documentación oficial:** `docker image rm` (equivalente a `docker rmi`) elimina imágenes locales. `--force` fuerza la eliminación incluso si hay contenedores derivados.

---

## 6. Gestión de Contenedores - Proxy

**Archivo:** `lib/kamal/commands/proxy.rb`

### `docker run` (proxy)
- **Línea:** 12-21, 137-144
- **Método:** `run`, `docker_run` (privado)
- **Propósito en Kamal:** Crear y ejecutar el contenedor de kamal-proxy (reverse proxy HTTP).
- **Comando:**
  ```
  docker run \
    --name kamal-proxy \
    --network kamal \
    --detach \
    --restart unless-stopped \
    --volume kamal-proxy-config:/home/kamal-proxy/.config/kamal-proxy \
    [opciones adicionales ...] \
    <proxy_image> \
    [<run_command>]
  ```
- **Flag destacado:**
  - `--volume kamal-proxy-config:/home/kamal-proxy/.config/kamal-proxy` — Volumen nombrado para persistir la configuración del proxy entre reinicios/recreaciones.

### `docker container start <proxy_name>`
- **Línea:** 28
- **Método:** `start`

### `docker container stop <proxy_name>`
- **Línea:** 32
- **Método:** `stop`

### `docker ps --filter 'name=^<proxy_name>$'`
- **Línea:** 40
- **Método:** `info`
- **Nota:** Usa regex anchored (`^...$`) para match exacto del nombre.

---

## 7. Ejecución de Comandos en Contenedores

### `docker exec` (en contenedor existente)

**Archivo:** `lib/kamal/commands/app/execution.rb:2-8`, `lib/kamal/commands/accessory.rb:57-62`

```
docker exec [-it | -i] [--env <key>=<value> ...] <container_name> <command...>
```

- **Propósito en Kamal:** Ejecutar comandos dentro de un contenedor que ya está corriendo (ej: `rails console`, migraciones).
- **Documentación oficial:**
  - `-i` (`--interactive`) — Mantener STDIN abierto.
  - `-t` (`--tty`) — Asignar pseudo-TTY. Kamal usa `-it` si STDIN es un TTY, solo `-i` si no lo es.
  - `--env` — Pasar variables de entorno adicionales al proceso ejecutado.

### `docker run` (ejecución temporal)

**Archivo:** `lib/kamal/commands/app/execution.rb:10-24`, `lib/kamal/commands/accessory.rb:64-74`

```
docker run [-it | -i] [--detach | --rm] \
  --name <prefix>-exec-<version>-<random> \
  --network kamal \
  [--env <key>=<value> ...] \
  [--volume ...] \
  [opciones adicionales ...] \
  <image> \
  <command...>
```

- **Propósito en Kamal:** Ejecutar un comando en un nuevo contenedor (útil cuando no hay contenedor corriendo).
- **Flags destacados:**
  - `--rm` — Eliminar automáticamente el contenedor al terminar (modo no-detach).
  - `--detach` — Ejecutar en segundo plano (modo detach, no usa `--rm`).

### `docker exec <proxy_name> kamal-proxy <subcommand>`

**Archivo:** `lib/kamal/commands/app/proxy.rb:29-31`, `lib/kamal/commands/accessory/proxy.rb:13-15`

- **Propósito en Kamal:** Ejecutar comandos del binario `kamal-proxy` dentro del contenedor proxy para gestionar deploys, routing, etc.
- **Subcomandos:** `deploy`, `remove`, `resume`, `stop`

---

## 8. Logs

### `docker logs` (app)

**Archivo:** `lib/kamal/commands/app/logging.rb`

```
docker logs [--timestamps] [--since <timestamp>] [--tail <lines>] <container_id> 2>&1
docker logs [--timestamps] [--tail <lines>] --follow <container_id> 2>&1
```

- **Nota:** El comando se pasa como string a `xargs` porque el container_id viene de un pipe.
- **Documentación oficial:**
  - `--timestamps` — Mostrar timestamps en cada línea de log.
  - `--since` — Mostrar logs desde un timestamp (ej: `2024-01-01`, `10m`).
  - `--tail` — Número de líneas a mostrar desde el final.
  - `--follow` (`-f`) — Seguir la salida de logs en tiempo real (streaming).
  - `2>&1` — Redirigir stderr a stdout ya que `docker logs` emite stderr y stdout por separado.

### `docker logs` (proxy y accessories)

**Archivos:** `lib/kamal/commands/proxy.rb:49-59`, `lib/kamal/commands/accessory.rb:44-55`

```
docker logs <container_name> [--since <time>] [--tail <lines>] [--timestamps] 2>&1
docker logs <container_name> [--timestamps] --tail 10 --follow 2>&1
```

- Mismos flags que los de app, pero pasan el nombre del contenedor directamente (no mediante pipe).

---

## 9. Gestión de Imágenes

### `docker image ls`

**Archivos:** `lib/kamal/commands/app/images.rb:2-4`, `lib/kamal/commands/app.rb:82`

```
docker image ls <repository>
docker image ls --filter reference=<image> --format '{{.ID}}'
```

- **Propósito en Kamal:** Listar imágenes disponibles localmente para el servicio.
- **Documentación oficial:** `docker image ls` (equivale a `docker images`) lista imágenes locales. `--filter reference=` filtra por nombre/tag. `--format` permite plantillas Go.

### `docker tag <source> <target>`

**Archivo:** `lib/kamal/commands/app/images.rb:10-12`

```
docker tag <registry>/<repo>:<version> <registry>/<repo>:latest
```

- **Propósito en Kamal:** Etiquetar la imagen recién desplegada como `latest`.
- **Documentación oficial:** `docker tag` crea una nueva referencia (tag) que apunta a la misma imagen. No duplica datos, solo crea un alias.

### `docker image prune`

**Archivos:** `lib/kamal/commands/app/images.rb:6-8`, `lib/kamal/commands/proxy.rb:67`, `lib/kamal/commands/prune.rb:6`

```
docker image prune --all --force --filter label=service=<name>
docker image prune --all --force --filter label=org.opencontainers.image.title=kamal-proxy
docker image prune --force --filter label=service=<name>
```

- **Propósito en Kamal:** Limpiar imágenes no utilizadas.
- **Documentación oficial:**
  - `--all` (`-a`) — Eliminar todas las imágenes no usadas (no solo las dangling/sin tag).
  - `--force` (`-f`) — No pedir confirmación.
  - `--filter label=` — Solo eliminar imágenes con el label especificado.

### `docker image rm --force <image>`

**Archivos:** `lib/kamal/commands/builder/base.rb:14`, `lib/kamal/commands/accessory.rb:107`

```
docker image rm --force <registry>/<repo>:<version>
```

- **Propósito en Kamal:** Eliminar una imagen específica del servidor.
- **Documentación oficial:** Equivale a `docker rmi`. `--force` fuerza la eliminación incluso si la imagen está siendo usada por contenedores detenidos.

### `docker pull <image>`

**Archivo:** `lib/kamal/commands/builder/base.rb:30`

```
docker pull <registry>/<repo>:<version>
```

- **Propósito en Kamal:** Descargar la imagen construida al servidor donde se ejecutará.
- **Documentación oficial:** `docker pull` descarga una imagen o repositorio de un registro. Sin especificar tag, usa `latest`.

### `docker push <image>`

**Archivo:** `lib/kamal/commands/builder/pack.rb:35-36`

```
docker push <registry>/<repo>:<version>
docker push <registry>/<repo>:latest
```

- **Propósito en Kamal:** Subir imágenes construidas con Cloud Native Buildpacks al registro.
- **Documentación oficial:** `docker push` sube una imagen o repositorio a un registro. Requiere autenticación previa con `docker login`.

### `docker rmi <tag>` (inline en shell)

**Archivo:** `lib/kamal/commands/prune.rb:13`

```
while read image tag; do docker rmi $tag; done
```

- **Propósito en Kamal:** Eliminar imágenes antiguas taggeadas durante el proceso de pruning.
- **Documentación oficial:** `docker rmi` es alias de `docker image rm`. Elimina una o más imágenes.

---

## 10. Build con Buildx

**Archivos:** `lib/kamal/commands/builder/base.rb`, `local.rb`, `remote.rb`, `hybrid.rb`, `cloud.rb`

### `docker buildx build`

**Archivo:** `lib/kamal/commands/builder/base.rb:17-27`

```
docker buildx build \
  --output=type=<registry|docker> \
  [--platform linux/amd64,linux/arm64] \
  [--builder <builder_name>] \
  -t <registry>/<repo>:<version> \
  -t <registry>/<repo>:latest \
  [--cache-to <spec>] \
  [--cache-from <spec>] \
  [--label service=<name>] \
  [--build-arg <key>=<value> ...] \
  [--secret id=<name> ...] \
  [--file <dockerfile>] \
  [--target <stage>] \
  [--ssh <spec>] \
  [--provenance <bool>] \
  [--sbom <bool>] \
  [--no-cache] \
  <context> \
  2>&1
```

- **Propósito en Kamal:** Construir imágenes Docker, opcionalmente multi-arquitectura.
- **Documentación oficial de cada flag:**
  - `--output=type=registry` — Enviar resultado directamente al registro (sin almacenar localmente).
  - `--output=type=docker` — Cargar resultado al Docker local.
  - `--platform` — Arquitecturas objetivo. Permite builds multi-arch (ej: `linux/amd64,linux/arm64`).
  - `--builder` — Seleccionar builder específico (no usado con driver `docker`).
  - `-t` (`--tag`) — Nombre y tag para la imagen. Se pueden especificar múltiples.
  - `--cache-to` / `--cache-from` — Configurar caché de build externo (ej: registro, local).
  - `--label` — Añadir metadatos a la imagen.
  - `--build-arg` — Pasar variables de build (disponibles solo durante el build).
  - `--secret` — Montar secretos durante el build (no quedan en la imagen final).
  - `--file` (`-f`) — Ruta al Dockerfile.
  - `--target` — Build stage específico en multi-stage Dockerfiles.
  - `--ssh` — Forwarding de agente SSH durante el build.
  - `--provenance` — Generar attestation de procedencia (SLSA).
  - `--sbom` — Generar Software Bill of Materials.
  - `--no-cache` — No usar caché del build anterior.

### `docker buildx create` (local)

**Archivo:** `lib/kamal/commands/builder/local.rb:5`

```
docker buildx create --name kamal-local-<driver> --driver=<driver> [--driver-opt network=host]
```

- **Propósito en Kamal:** Crear un builder local para builds multi-arch.
- **Documentación oficial:**
  - `--name` — Nombre del builder.
  - `--driver` — Driver del builder (`docker-container`, `remote`, `cloud`).
  - `--driver-opt` — Opciones del driver. `network=host` se usa con registros locales.

### `docker buildx create` (remote)

**Archivo:** `lib/kamal/commands/builder/remote.rb:69`

```
docker buildx create --name kamal-remote-<suffix> [--driver-opt network=host] <context_name>
```

- **Propósito en Kamal:** Crear builder que usa un host Docker remoto como backend.
- El último argumento es el contexto Docker que apunta al host remoto.

### `docker buildx create` (hybrid)

**Archivo:** `lib/kamal/commands/builder/hybrid.rb:14-19`

```
docker buildx create --platform linux/<local_arch> --name kamal-hybrid-<suffix> --driver=<driver> [--driver-opt ...]
docker buildx create --platform linux/<remote_arch> --append --name kamal-hybrid-<suffix> [--driver-opt ...] <remote_context>
```

- **Propósito en Kamal:** Crear builder híbrido que usa local para una arquitectura y remoto para otra.
- **Flag destacado:**
  - `--append` — Añadir un nodo al builder existente en lugar de crear uno nuevo. Cada nodo maneja diferentes plataformas.

### `docker buildx create` (cloud)

**Archivo:** `lib/kamal/commands/builder/cloud.rb:5`

```
docker buildx create --driver "cloud <org>/<builder>"
```

- **Propósito en Kamal:** Crear builder usando Docker Build Cloud.

### `docker buildx rm <builder_name>`

**Archivos:** `local.rb:9`, `remote.rb:73`, `cloud.rb:9`

- **Propósito en Kamal:** Eliminar un builder previamente creado.
- **Documentación oficial:** Elimina un builder de buildx y todos sus nodos asociados.

### `docker buildx ls`

**Archivo:** `lib/kamal/commands/builder/base.rb:36`

- **Propósito en Kamal:** Listar builders disponibles (información/diagnóstico).
- **Documentación oficial:** Lista todos los builders de buildx y sus nodos.

### `docker buildx inspect <builder_name>`

**Archivo:** `lib/kamal/commands/builder/base.rb:40`

- **Propósito en Kamal:** Verificar estado y configuración de un builder.
- **Documentación oficial:** Muestra información detallada del builder (plataformas soportadas, nodos, endpoints).

### `docker buildx version`

**Archivo:** `lib/kamal/commands/base.rb:140`

- **Propósito en Kamal:** Verificar que buildx está instalado.

---

## 11. Contextos Docker

**Archivo:** `lib/kamal/commands/builder/remote.rb`

### `docker context create`

```
docker context create <name> --description '<description>' --docker 'host=<ssh://user@host>'
```

- **Línea:** 61
- **Propósito en Kamal:** Crear un contexto Docker que apunta a un host remoto para builds remotos.
- **Documentación oficial:** `docker context create` crea un contexto nombrado que almacena la configuración de conexión al daemon Docker. Permite cambiar fácilmente entre diferentes hosts Docker.
  - `--description` — Descripción del contexto.
  - `--docker` — Configuración del endpoint Docker. `host=ssh://user@host` usa SSH para conectar al daemon remoto.

### `docker context ls`

- **Línea:** 16
- **Propósito en Kamal:** Listar contextos disponibles (información/diagnóstico).

### `docker context inspect <name> --format '{{.Endpoints.docker.Host}}'`

- **Línea:** 56
- **Propósito en Kamal:** Verificar que un contexto existente apunta al host remoto correcto.

### `docker context rm <name>`

- **Línea:** 65
- **Propósito en Kamal:** Eliminar un contexto remoto al desmantelar el builder.

---

## 12. Limpieza y Pruning

**Archivo:** `lib/kamal/commands/prune.rb`, varios otros.

### `docker container prune --force --filter label=service=<name>`

**Archivos:** `lib/kamal/commands/app/containers.rb:23`, `lib/kamal/commands/accessory.rb:103`

- **Propósito en Kamal:** Eliminar contenedores detenidos del servicio.
- **Documentación oficial:** `docker container prune` elimina todos los contenedores detenidos. `--force` omite confirmación. `--filter` limita a contenedores con labels específicos.

### `docker container prune --force --filter label=org.opencontainers.image.title=kamal-proxy`

**Archivo:** `lib/kamal/commands/proxy.rb:63`

- **Propósito en Kamal:** Limpiar contenedores viejos del proxy.

### `docker ps -q -a --filter label=service=<name> --filter status=created --filter status=exited --filter status=dead`

**Archivo:** `lib/kamal/commands/prune.rb:18`

- **Propósito en Kamal:** Listar contenedores detenidos para eliminación selectiva (manteniendo N contenedores recientes).
- **Nota:** Se combina con `tail -n +<retain+1>` para saltar los más recientes.

### `docker rm <container_id>` (inline en shell)

**Archivo:** `lib/kamal/commands/prune.rb:20`

```
while read container_id; do docker rm $container_id; done
```

- **Propósito en Kamal:** Eliminar contenedores individualmente del pipe.

### `docker container ls -a --format '{{.Image}}' --filter label=service=<name>`

**Archivo:** `lib/kamal/commands/prune.rb:32`

- **Propósito en Kamal:** Obtener lista de imágenes en uso por contenedores activos (para evitar eliminarlas durante el pruning).

---

## 13. Inspección y Diagnóstico

### `docker inspect --format <template> <target>`

**Archivos:** `lib/kamal/commands/app.rb:42`, `lib/kamal/commands/app/containers.rb:29`, `lib/kamal/commands/proxy.rb:45`, `lib/kamal/commands/builder/base.rb:53`

**Variantes:**
```
docker inspect --format '{{if .State.Health}}{{.State.Health.Status}}{{else}}{{.State.Status}}{{end}}' <container>
docker inspect --format '{{json .State.Health}}' <container>
docker inspect --format '{{.Config.Image}}' <container>
docker inspect -f '{{ .Config.Labels.service }}' <image>
```

- **Propósito en Kamal:**
  - Verificar estado de salud de contenedores (health checks).
  - Obtener log de health checks en formato JSON.
  - Obtener la imagen/versión que ejecuta un contenedor.
  - Validar que una imagen tiene el label `service` correcto.
- **Documentación oficial:** `docker inspect` devuelve información detallada en JSON sobre objetos Docker. `--format` / `-f` permite extraer campos específicos usando plantillas Go. Funciona tanto con contenedores como con imágenes.

### `docker info --format '{{index .RegistryConfig.Mirrors 0}}'`

**Archivo:** `lib/kamal/commands/builder/base.rb:61`

- **Propósito en Kamal:** Obtener el primer mirror de registro configurado.
- **Documentación oficial:** `docker info` muestra información del sistema Docker (storage driver, kernel, OS, registros configurados, etc.). `--format` permite extraer campos específicos.

---

## 14. Gestión de Contenedores - Assets

**Archivo:** `lib/kamal/commands/app/assets.rb`

### `docker container rm <asset_container>`

- **Línea:** 7, 10
- **Propósito en Kamal:** Limpiar contenedor temporal usado para extracción de assets.

### `docker container create --name <asset_container> <image>`

- **Línea:** 8
- **Propósito en Kamal:** Crear contenedor sin ejecutarlo para poder copiar archivos de él.
- **Documentación oficial:** `docker container create` (equivale a `docker create`) crea un contenedor nuevo sin iniciarlo. Útil para preparar un contenedor para inspección o copia de archivos.

### `docker container cp -L <container>:<path>/. <host_path>`

- **Línea:** 9
- **Propósito en Kamal:** Copiar assets compilados desde el contenedor al host.
- **Documentación oficial:**
  - `docker container cp` (equivale a `docker cp`) copia archivos entre contenedor y host.
  - `-L` — Seguir symlinks en el contenedor origen.
  - `<container>:<path>/.` — El `/.` al final copia el contenido del directorio, no el directorio en sí.

---

## 15. Contenedores - Operaciones Adicionales

### `docker container ls --all --filter ... [--format ...]`

**Archivo:** `lib/kamal/commands/app/containers.rb:4-9`, `lib/kamal/commands/base.rb:17-19`

- **Propósito en Kamal:** Listar contenedores (alias de `docker ps --all`).
- **Variante con `--quiet`:** Solo devuelve IDs de contenedores (para uso en pipes).

### `docker container rm <container>`

**Archivo:** `lib/kamal/commands/app/containers.rb:15`

- **Propósito en Kamal:** Eliminar un contenedor específico por versión.

### `docker rename <old_name> <new_name>`

**Archivo:** `lib/kamal/commands/app/containers.rb:19`

- **Propósito en Kamal:** Renombrar contenedor durante deploys (ej: marcar como versión antigua).
- **Documentación oficial:** `docker rename` cambia el nombre de un contenedor existente. El contenedor puede estar corriendo o detenido.

### `docker container stop traefik`

**Archivo:** `lib/kamal/commands/proxy.rb:72`

- **Propósito en Kamal:** Detener el contenedor legacy de Traefik durante migración a kamal-proxy.

---

## Resumen de Flags Utilizados

| Flag | Comandos donde se usa | Descripción oficial |
|------|----------------------|---------------------|
| `--detach` / `-d` | `run` | Ejecutar en segundo plano |
| `--restart unless-stopped` | `run` | Reiniciar siempre excepto parada manual |
| `--name` | `run`, `create` | Nombre del contenedor |
| `--network` | `run` | Conectar a red Docker |
| `--hostname` | `run` | Hostname dentro del contenedor |
| `--env` / `-e` | `run`, `exec` | Variable de entorno |
| `--volume` / `-v` | `run` | Montar volumen |
| `--label` / `-l` | `run`, `buildx build` | Metadatos/labels |
| `--publish` / `-p` | `run` | Mapear puertos |
| `--rm` | `run` | Eliminar al terminar |
| `-it` / `-i` | `exec`, `run` | Interactivo + TTY |
| `--all` / `-a` | `ps`, `container ls`, `image prune` | Incluir todos (no solo activos) |
| `--quiet` / `-q` | `ps`, `container ls` | Solo IDs |
| `--filter` / `-f` | `ps`, `prune`, `image ls` | Filtrar resultados |
| `--format` | `ps`, `inspect`, `info`, `image ls` | Plantilla Go de salida |
| `--latest` / `-l` | `ps` | Solo el más reciente |
| `--force` | `prune`, `image rm` | Sin confirmación |
| `--timestamps` | `logs` | Mostrar timestamps |
| `--since` | `logs` | Logs desde timestamp |
| `--tail` | `logs` | N líneas desde el final |
| `--follow` / `-f` | `logs` | Streaming en tiempo real |
| `--output` | `buildx build` | Destino del build |
| `--platform` | `buildx build`, `buildx create` | Arquitecturas objetivo |
| `--builder` | `buildx build` | Builder específico |
| `--cache-to/from` | `buildx build` | Caché externo de build |
| `--build-arg` | `buildx build` | Variable de build |
| `--secret` | `buildx build` | Secreto de build |
| `--file` | `buildx build` | Ruta Dockerfile |
| `--target` | `buildx build` | Stage de multi-stage build |
| `--ssh` | `buildx build` | SSH agent forwarding |
| `--provenance` | `buildx build` | SLSA attestation |
| `--sbom` | `buildx build` | Software Bill of Materials |
| `--no-cache` | `buildx build` | Ignorar caché |
| `--driver` | `buildx create` | Driver del builder |
| `--driver-opt` | `buildx create` | Opciones del driver |
| `--append` | `buildx create` | Añadir nodo a builder |
| `--description` | `context create` | Descripción del contexto |
| `--docker` | `context create` | Config endpoint Docker |
| `-u` | `login` | Usuario |
| `-p` | `login` | Contraseña |
| `-L` | `container cp` | Seguir symlinks |

---

## Resumen Estadístico

| Categoría | Nº de comandos únicos |
|-----------|----------------------|
| Contenedores (run/start/stop/rm/rename/create) | 8 |
| Listado/inspección (ps/ls/inspect/info) | 6 |
| Imágenes (pull/push/tag/ls/rm/prune) | 6 |
| Build (buildx build/create/rm/ls/inspect) | 5 |
| Logs | 1 |
| Red (network create) | 1 |
| Registro (login/logout) | 2 |
| Contextos (create/ls/inspect/rm) | 4 |
| Limpieza (container prune, image prune) | 2 |
| Otros (version, cp, rmi) | 3 |
| **Total** | **38** |

---

## Observaciones

1. **Todos los contenedores de app y accessories usan `--restart unless-stopped`**, lo que garantiza que se reinicien automáticamente tras reinicios del servidor, excepto si fueron detenidos manualmente.

2. **La red `kamal` es central**: todos los contenedores (app, proxy, accessories) se conectan a ella, permitiendo comunicación por nombre de contenedor.

3. **Labels como mecanismo de filtrado**: Kamal usa extensivamente labels (`service`, `role`, `destination`) para filtrar y gestionar contenedores de forma selectiva con `--filter label=`.

4. **Buildx es el builder por defecto**: Kamal no usa `docker build` clásico, sino `docker buildx build` que soporta builds multi-arquitectura y output directo a registros.

5. **Seguridad de credenciales**: El flag `-p` en `docker login` pasa la contraseña en línea de comandos. Aunque se ejecuta vía SSH (no queda en historial local), podría aparecer en `/proc` o logs del sistema en el servidor.

6. **`docker container` vs `docker`**: Kamal usa tanto la forma corta (`docker ps`, `docker stop`) como la forma larga (`docker container ls`, `docker container stop`) de manera inconsistente. Ambas son equivalentes según la documentación oficial.

7. **Inline shell commands**: En el módulo de pruning (`prune.rb`), algunos comandos Docker se construyen como strings dentro de pipes de shell (`while read ... do docker rmi ... done`) en lugar de usar el helper `docker()`, lo que dificulta su tracking y potencialmente los expone a problemas de shell injection si las variables no están sanitizadas.
