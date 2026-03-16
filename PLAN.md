# Plan: Capa de Abstracción Docker con Adaptadores

## Visión General

Diseñar un módulo `DockerEngine` que exponga cada comando Docker como un método Ruby
con parámetros tipados, y delegue la ejecución a adaptadores intercambiables (CLI local,
SSH remoto, API HTTP de Docker). El diseño debe permitir añadir adaptadores futuros
sin modificar el código existente (Open/Closed Principle).

## Arquitectura

```
DockerEngine::Client (interfaz pública)
  │
  ├── #containers  → DockerEngine::Resources::Container
  ├── #images      → DockerEngine::Resources::Image
  ├── #networks    → DockerEngine::Resources::Network
  ├── #registries  → DockerEngine::Resources::Registry
  ├── #volumes     → DockerEngine::Resources::Volume
  ├── #builders    → DockerEngine::Resources::Builder
  ├── #contexts    → DockerEngine::Resources::Context
  ├── #system      → DockerEngine::Resources::System
  │
  └── adapter (inyectado)
        ├── DockerEngine::Adapters::Cli      (llamadas al sistema local)
        ├── DockerEngine::Adapters::Ssh      (ejecución remota vía SSH)
        ├── DockerEngine::Adapters::HttpApi  (Docker Engine REST API)
        └── (futuros adaptadores)
```

## Estructura de Archivos

```
lib/docker_engine/
├── client.rb                    # Punto de entrada principal
├── adapter.rb                   # Clase base abstracta para adaptadores
├── adapters/
│   ├── cli.rb                   # Adaptador CLI local (Open3/system)
│   ├── ssh.rb                   # Adaptador SSH remoto (net-ssh)
│   └── http_api.rb              # Adaptador Docker Engine API (HTTP)
├── resources/
│   ├── container.rb             # Operaciones de contenedores
│   ├── image.rb                 # Operaciones de imágenes
│   ├── network.rb               # Operaciones de red
│   ├── registry.rb              # Login/logout de registros
│   ├── volume.rb                # Operaciones de volúmenes
│   ├── builder.rb               # Operaciones buildx
│   ├── context.rb               # Contextos Docker
│   └── system.rb                # Info, version, etc.
├── result.rb                    # Objeto resultado estandarizado
├── errors.rb                    # Jerarquía de errores
└── config.rb                    # Configuración del cliente
```

## Diseño Detallado

### 1. `DockerEngine::Client` — Punto de entrada

```ruby
client = DockerEngine::Client.new(adapter: :cli)
client = DockerEngine::Client.new(adapter: :ssh, host: "deploy@server.com", port: 22)
client = DockerEngine::Client.new(adapter: :http_api, url: "unix:///var/run/docker.sock")
client = DockerEngine::Client.new(adapter: :http_api, url: "https://remote:2376", tls: { ... })

# Uso
client.containers.run("nginx:latest", name: "web", detach: true, network: "kamal")
client.containers.stop("web", timeout: 10)
client.images.pull("myapp:v1.0")
```

### 2. `DockerEngine::Adapter` — Clase base abstracta

Define la interfaz que todo adaptador debe implementar. Los adaptadores traducen
la operación abstracta al mecanismo concreto.

```ruby
class DockerEngine::Adapter
  # Ejecutar un comando Docker y devolver Result
  def execute(command, *args, **options) → Result
    raise NotImplementedError
  end

  # Ejecutar y hacer streaming de la salida (para logs --follow)
  def stream(command, *args, **options, &block) → void
    raise NotImplementedError
  end

  # Copiar archivos desde/hacia contenedor
  def copy_from(container, path) → IO
    raise NotImplementedError
  end

  def copy_to(container, path, archive) → Result
    raise NotImplementedError
  end
end
```

### 3. `DockerEngine::Result` — Respuesta estandarizada

```ruby
class DockerEngine::Result
  attr_reader :output     # String - stdout o body JSON parseado
  attr_reader :exit_code  # Integer - 0 para éxito (HTTP: 2xx → 0)
  attr_reader :error      # String - stderr o mensaje de error

  def success? → Boolean
  def to_s → String       # output
  def parsed → Hash/Array # JSON.parse(output) si aplica
end
```

### 4. Resources — Métodos por recurso

#### 4.1 `DockerEngine::Resources::Container`

| Método | Parámetros | CLI | SSH | HTTP API |
|--------|-----------|-----|-----|----------|
| `run` | `image:, name:, detach: true, restart: nil, network: nil, hostname: nil, env: {}, volumes: [], labels: {}, ports: [], options: [], cmd: nil` | `docker run --detach --name ...` | igual vía SSH | `POST /containers/create` + `POST /containers/{id}/start` |
| `start` | `name:` | `docker start <name>` | igual | `POST /containers/{id}/start` |
| `stop` | `name:, timeout: nil, signal: nil` | `docker stop [--time N] <name>` | igual | `POST /containers/{id}/stop?t=N` |
| `remove` | `name:, force: false, volumes: false` | `docker rm [--force] <name>` | igual | `DELETE /containers/{id}?force=&v=` |
| `rename` | `name:, new_name:` | `docker rename <old> <new>` | igual | `POST /containers/{id}/rename?name=` |
| `list` | `all: false, filters: {}, format: nil, quiet: false, latest: false` | `docker ps [--all] [--filter ...] [--format ...] [--quiet] [--latest]` | igual | `GET /containers/json?all=&filters=` |
| `inspect` | `name:, format: nil` | `docker inspect [--format ...] <name>` | igual | `GET /containers/{id}/json` |
| `logs` | `name:, timestamps: false, since: nil, tail: nil, follow: false` | `docker logs [flags] <name>` | igual | `GET /containers/{id}/logs?stdout=1&stderr=1&...` |
| `exec` | `name:, command:, interactive: false, tty: false, env: {}, detach: false` | `docker exec [-it] [--env ...] <name> <cmd>` | igual | `POST /containers/{id}/exec` + `POST /exec/{id}/start` |
| `create` | `image:, name:, **opts` | `docker container create --name ... <image>` | igual | `POST /containers/create` |
| `copy_from` | `name:, path:` | `docker container cp <name>:<path> -` | igual | `GET /containers/{id}/archive?path=` |
| `copy_to` | `name:, path:, archive:` | `docker container cp - <name>:<path>` | igual | `PUT /containers/{id}/archive?path=` |
| `prune` | `filters: {}, force: true` | `docker container prune --force [--filter ...]` | igual | `POST /containers/prune?filters=` |

#### 4.2 `DockerEngine::Resources::Image`

| Método | Parámetros | CLI | HTTP API |
|--------|-----------|-----|----------|
| `list` | `repository: nil, filters: {}, format: nil` | `docker image ls [repo] [--filter ...] [--format ...]` | `GET /images/json?filters=` |
| `pull` | `image:, tag: nil, platform: nil` | `docker pull <image>[:<tag>]` | `POST /images/create?fromImage=&tag=` |
| `push` | `image:, tag: nil` | `docker push <image>[:<tag>]` | `POST /images/{name}/push?tag=` |
| `tag` | `source:, target:` | `docker tag <source> <target>` | `POST /images/{name}/tag?repo=&tag=` |
| `remove` | `image:, force: false` | `docker image rm [--force] <image>` | `DELETE /images/{name}?force=` |
| `inspect` | `image:, format: nil` | `docker inspect [-f ...] <image>` | `GET /images/{name}/json` |
| `prune` | `all: false, filters: {}, force: true` | `docker image prune [--all] --force [--filter ...]` | `POST /images/prune?filters=` |
| `build` | `context:, tags: [], platform: nil, builder: nil, file: nil, target: nil, build_args: {}, secrets: [], cache_from: nil, cache_to: nil, output: nil, ssh: nil, provenance: nil, sbom: nil, no_cache: false, labels: {}` | `docker buildx build [flags] <context>` | N/A (buildx no tiene API HTTP directa) |

#### 4.3 `DockerEngine::Resources::Network`

| Método | Parámetros | CLI | HTTP API |
|--------|-----------|-----|----------|
| `create` | `name:, driver: nil, labels: {}` | `docker network create [--driver ...] <name>` | `POST /networks/create` |

#### 4.4 `DockerEngine::Resources::Registry`

| Método | Parámetros | CLI | HTTP API |
|--------|-----------|-----|----------|
| `login` | `server:, username:, password:` | `docker login <server> -u <user> -p <pass>` | `POST /auth` |
| `logout` | `server:` | `docker logout <server>` | N/A (no hay endpoint, es local) |

#### 4.5 `DockerEngine::Resources::Builder`

| Método | Parámetros | CLI | HTTP API |
|--------|-----------|-----|----------|
| `create` | `name:, driver: nil, platform: nil, append: false, driver_opts: [], context: nil` | `docker buildx create --name ... [--driver ...] [--append] [context]` | N/A |
| `remove` | `name:` | `docker buildx rm <name>` | N/A |
| `list` | — | `docker buildx ls` | N/A |
| `inspect` | `name:` | `docker buildx inspect <name>` | N/A |

> **Nota:** Buildx no tiene API HTTP. El adaptador HttpApi lanzará
> `DockerEngine::UnsupportedOperationError` para estas operaciones.

#### 4.6 `DockerEngine::Resources::Context`

| Método | Parámetros | CLI | HTTP API |
|--------|-----------|-----|----------|
| `create` | `name:, description: nil, docker_host:` | `docker context create <name> --docker 'host=...'` | N/A |
| `list` | — | `docker context ls` | N/A |
| `inspect` | `name:, format: nil` | `docker context inspect <name> [--format ...]` | N/A |
| `remove` | `name:` | `docker context rm <name>` | N/A |

> **Nota:** Contextos Docker son locales al cliente. Solo CLI y SSH los soportan.

#### 4.7 `DockerEngine::Resources::System`

| Método | Parámetros | CLI | HTTP API |
|--------|-----------|-----|----------|
| `version` | — | `docker version` | `GET /version` |
| `client_version` | — | `docker -v` | N/A |
| `info` | `format: nil` | `docker info [--format ...]` | `GET /system/info` |

### 5. Adaptadores — Implementación

#### 5.1 `DockerEngine::Adapters::Cli`

- Ejecuta comandos mediante `Open3.capture3`
- Construye strings de comando a partir de los parámetros del método
- Soporta todas las operaciones (buildx, context, etc.)

```ruby
class DockerEngine::Adapters::Cli < DockerEngine::Adapter
  def execute(command, *args, **options)
    cmd = build_command(command, *args, **options)
    stdout, stderr, status = Open3.capture3(cmd_env, cmd)
    Result.new(output: stdout, error: stderr, exit_code: status.exitstatus)
  end

  def stream(command, *args, **options, &block)
    cmd = build_command(command, *args, **options)
    Open3.popen3(cmd_env, cmd) do |stdin, stdout, stderr, wait_thr|
      stdout.each_line { |line| block.call(line) }
    end
  end

  private

  def build_command(command, *args, **options)
    # Construye: "docker <command> <args> <options>"
  end
end
```

#### 5.2 `DockerEngine::Adapters::Ssh`

- Ejecuta comandos Docker en un servidor remoto vía `net-ssh`
- Misma construcción de comandos que CLI, pero ejecución remota
- Configuración: host, user, port, keys, proxy

```ruby
class DockerEngine::Adapters::Ssh < DockerEngine::Adapter
  def initialize(host:, user: "root", port: 22, keys: [], proxy: nil)
    @connection_options = { host:, user:, port:, keys:, proxy: }
  end

  def execute(command, *args, **options)
    cmd = build_command(command, *args, **options)
    output = ""
    error = ""
    exit_code = nil

    with_ssh_connection do |ssh|
      ssh.exec!(cmd) do |ch, stream, data|
        output << data if stream == :stdout
        error << data if stream == :stderr
      end
      exit_code = ssh.exec!(cmd).exitstatus  # simplificado
    end

    Result.new(output:, error:, exit_code:)
  end
end
```

#### 5.3 `DockerEngine::Adapters::HttpApi`

- Conecta al Docker Engine API vía Unix socket o TCP (con TLS opcional)
- Traduce operaciones a llamadas HTTP REST
- No soporta buildx ni context (lanza UnsupportedOperationError)

```ruby
class DockerEngine::Adapters::HttpApi < DockerEngine::Adapter
  API_VERSION = "v1.45"

  def initialize(url: "unix:///var/run/docker.sock", tls: nil)
    @url = url
    @tls = tls
  end

  def execute(command, *args, **options)
    method, path, body = translate(command, *args, **options)
    response = http_request(method, "/#{API_VERSION}#{path}", body:)
    Result.new(
      output: response.body,
      exit_code: response.success? ? 0 : 1,
      error: response.success? ? nil : response.body
    )
  end

  private

  def translate(command, *args, **options)
    # Mapea operación abstracta → HTTP method + path + body
    # Ejemplo: [:container, :run, {image: "nginx", name: "web"}]
    #       → [:post, "/containers/create?name=web", {Image: "nginx", ...}]
  end
end
```

### 6. Manejo de Operaciones No Soportadas

Algunos adaptadores no soportan todas las operaciones:

| Operación | CLI | SSH | HTTP API |
|-----------|-----|-----|----------|
| `builder.*` (buildx) | OK | OK | `UnsupportedOperationError` |
| `context.*` | OK | OK | `UnsupportedOperationError` |
| `registry.logout` | OK | OK | `UnsupportedOperationError` |
| `container.copy_from/to` | OK | OK | OK |
| `container.logs(follow: true)` | OK (streaming) | OK (streaming) | OK (streaming) |

### 7. Jerarquía de Errores

```ruby
module DockerEngine
  class Error < StandardError; end
  class ConnectionError < Error; end              # No se puede conectar al daemon
  class ContainerNotFoundError < Error; end       # Contenedor no existe
  class ImageNotFoundError < Error; end           # Imagen no existe
  class AuthenticationError < Error; end          # Fallo de login en registro
  class CommandError < Error                      # Comando falló (exit code != 0)
    attr_reader :result
  end
  class UnsupportedOperationError < Error; end    # Operación no soportada por adaptador
  class TimeoutError < Error; end                 # Timeout en la operación
end
```

### 8. Ejemplo de Uso Completo

```ruby
# Crear cliente con adaptador CLI local
client = DockerEngine::Client.new(adapter: :cli)

# Crear cliente SSH para servidor remoto
client = DockerEngine::Client.new(
  adapter: :ssh,
  host: "deploy@production.server.com",
  port: 22,
  keys: ["~/.ssh/deploy_key"]
)

# Crear cliente API HTTP via socket Unix
client = DockerEngine::Client.new(
  adapter: :http_api,
  url: "unix:///var/run/docker.sock"
)

# --- Operaciones ---

# Login en registro
client.registries.login(server: "ghcr.io", username: "user", password: "token")

# Crear red
client.networks.create(name: "kamal")

# Ejecutar contenedor
client.containers.run(
  image: "myapp:v1.0",
  name: "myapp-web-abc123",
  detach: true,
  restart: "unless-stopped",
  network: "kamal",
  env: {
    "RAILS_ENV" => "production",
    "KAMAL_VERSION" => "abc123"
  },
  volumes: ["/data:/app/data"],
  labels: { "service" => "myapp", "role" => "web" },
  cmd: "bin/rails server"
)

# Ver estado
result = client.containers.list(filters: { label: ["service=myapp"] })

# Inspeccionar salud
result = client.containers.inspect(name: "myapp-web-abc123")

# Ver logs
client.containers.logs(name: "myapp-web-abc123", tail: 100, timestamps: true)

# Ejecutar comando en contenedor existente
client.containers.exec(
  name: "myapp-web-abc123",
  command: ["rails", "console"],
  interactive: true,
  tty: true
)

# Detener
client.containers.stop(name: "myapp-web-abc123", timeout: 30)

# Limpiar
client.containers.prune(filters: { label: ["service=myapp"] })
client.images.prune(all: true, filters: { label: ["service=myapp"] })

# Build con buildx (solo CLI/SSH)
client.images.build(
  context: ".",
  tags: ["registry.com/myapp:v1.0", "registry.com/myapp:latest"],
  platform: "linux/amd64,linux/arm64",
  builder: "kamal-local",
  file: "Dockerfile",
  build_args: { "RUBY_VERSION" => "3.3" },
  output: "registry"
)

# Logout
client.registries.logout(server: "ghcr.io")
```

## Plan de Implementación

### Fase 1: Core y Adaptador CLI
1. `DockerEngine::Result` y `DockerEngine::Errors`
2. `DockerEngine::Adapter` (clase base abstracta)
3. `DockerEngine::Adapters::Cli` (ejecución local con Open3)
4. `DockerEngine::Client` (inicialización y routing a resources)
5. `DockerEngine::Resources::Container` (todas las operaciones)
6. `DockerEngine::Resources::Image` (todas las operaciones)
7. `DockerEngine::Resources::Network`
8. `DockerEngine::Resources::Registry`
9. `DockerEngine::Resources::System`
10. `DockerEngine::Resources::Builder`
11. `DockerEngine::Resources::Context`
12. Tests unitarios para cada resource con CLI

### Fase 2: Adaptador SSH
13. `DockerEngine::Adapters::Ssh` (ejecución remota)
14. Tests con SSH mockeado

### Fase 3: Adaptador HTTP API
15. `DockerEngine::Adapters::HttpApi` (REST via socket/TCP)
16. Traducción de operaciones a endpoints HTTP
17. Manejo de UnsupportedOperationError para buildx/context
18. Tests con HTTP mockeado

### Fase 4: Volumen (si necesario)
19. `DockerEngine::Resources::Volume` (create, ls, rm, prune)
