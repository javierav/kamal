# Plan: Capa de Abstracción Docker con Adaptadores

## Visión General

Diseñar un módulo `DockerEngine` que exponga cada comando Docker como un método Ruby
con parámetros tipados, y delegue la ejecución a adaptadores intercambiables (CLI local,
SSH remoto, API HTTP de Docker, WebSocket, etc.). El diseño debe permitir añadir
adaptadores futuros sin modificar el código existente (Open/Closed Principle), y soportar
tanto flujos síncronos como asíncronos.

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
        ├── DockerEngine::Adapters::Cli        (llamadas al sistema local)
        ├── DockerEngine::Adapters::Ssh        (ejecución remota vía SSH con pool)
        ├── DockerEngine::Adapters::HttpApi    (Docker Engine REST API)
        ├── DockerEngine::Adapters::WebSocket  (asíncrono vía WS)
        └── (futuros adaptadores)
```

## Estructura de Archivos

```
lib/docker_engine/
├── client.rb                    # Punto de entrada principal
├── adapter.rb                   # Clase base abstracta para adaptadores
├── future.rb                    # Future/Promise para resultados asíncronos
├── pipeline.rb                  # Secuencias declarativas de operaciones
├── connection_pool.rb           # Pool genérico de conexiones
├── pending_store.rb             # Interfaz para almacenar request_id pendientes
├── pending_stores/
│   ├── memory.rb                # Store en memoria (mismo proceso)
│   └── active_record.rb         # Store en DB (procesos separados)
├── callback_executor.rb         # Thread pool para ejecutar callbacks
├── adapters/
│   ├── cli.rb                   # Adaptador CLI local (Open3/system)
│   ├── ssh.rb                   # Adaptador SSH remoto (net-ssh + pool)
│   ├── http_api.rb              # Adaptador Docker Engine API (HTTP)
│   └── web_socket.rb            # Adaptador asíncrono WebSocket
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

# Uso síncrono
client.containers.run("nginx:latest", name: "web", detach: true, network: "kamal")
client.containers.stop("web", timeout: 10)
client.images.pull("myapp:v1.0")

# Uso asíncrono (cualquier adaptador)
future = client.async.containers.stop("web", timeout: 10)
future.on_success { |result| puts "Stopped: #{result}" }
future.on_failure { |error| puts "Failed: #{error}" }
result = future.value  # bloquear si se necesita el resultado
```

### 2. `DockerEngine::Adapter` — Clase base abstracta

Define la interfaz que todo adaptador debe implementar. Los adaptadores traducen
la operación abstracta al mecanismo concreto.

```ruby
class DockerEngine::Adapter
  # Ejecutar un comando Docker y devolver Result.
  # Los adaptadores síncronos devuelven Result directamente.
  # Los adaptadores asíncronos devuelven Future<Result>.
  def execute(command, *args, **options)
    raise NotImplementedError
  end

  # Ejecutar y hacer streaming de la salida (para logs --follow).
  def stream(command, *args, **options, &block)
    raise NotImplementedError
  end

  # Copiar archivos desde/hacia contenedor.
  def copy_from(container, path)
    raise NotImplementedError
  end

  def copy_to(container, path, archive)
    raise NotImplementedError
  end

  # ¿Es un adaptador asíncrono por naturaleza?
  # Los adaptadores síncronos devuelven false (CLI, SSH con pool).
  # Los adaptadores asíncronos nativos devuelven true (WebSocket).
  def async?
    false
  end

  # Cerrar conexiones / limpiar recursos.
  def close
    # noop por defecto
  end
end
```

### 3. `DockerEngine::Future` — Resultado asíncrono

Envuelve un resultado que puede no estar disponible aún. Permite tanto el uso
con callbacks como el bloqueo explícito. Funciona de forma transparente: si el
adaptador es síncrono, el Future se resuelve inmediatamente.

```ruby
class DockerEngine::Future
  def initialize(&block)
    @callbacks_success = []
    @callbacks_failure = []
    @mutex = Mutex.new
    @condition = ConditionVariable.new
    @resolved = false
    @result = nil
    @error = nil

    # Ejecutar el bloque en un thread (o resolver inmediatamente
    # si se pasa un valor ya resuelto con Future.resolved(value))
    if block_given?
      @thread = Thread.new { execute(&block) }
    end
  end

  # Crear un Future ya resuelto (para adaptadores síncronos)
  def self.resolved(result)
    new.tap { |f| f.send(:resolve!, result) }
  end

  # Crear un Future con error
  def self.failed(error)
    new.tap { |f| f.send(:reject!, error) }
  end

  # Registrar callback de éxito. Se ejecuta inmediatamente si ya resuelto.
  def on_success(&block)
    @mutex.synchronize do
      if @resolved && !@error
        block.call(@result)
      else
        @callbacks_success << block
      end
    end
    self
  end

  # Registrar callback de error. Se ejecuta inmediatamente si ya fallido.
  def on_failure(&block)
    @mutex.synchronize do
      if @resolved && @error
        block.call(@error)
      else
        @callbacks_failure << block
      end
    end
    self
  end

  # Bloquear hasta tener el resultado. Lanza excepción si falló.
  def value(timeout: nil)
    @mutex.synchronize do
      unless @resolved
        if timeout
          @condition.wait(@mutex, timeout)
          raise DockerEngine::TimeoutError, "Future not resolved within #{timeout}s" unless @resolved
        else
          @condition.wait(@mutex) until @resolved
        end
      end
      raise @error if @error
      @result
    end
  end

  # Encadenar transformaciones.
  # Si el bloque devuelve otro Future, se "aplana" automáticamente
  # (flat_map semántica) para evitar Future<Future<Result>>.
  def then(&block)
    Future.new do
      result = block.call(value)
      result.is_a?(Future) ? result.value : result
    end
  end

  def resolved?
    @resolved
  end

  private

  def execute
    result = yield
    resolve!(result)
  rescue => e
    reject!(e)
  end

  def resolve!(result)
    @mutex.synchronize do
      @result = result
      @resolved = true
      @callbacks_success.each { |cb| cb.call(result) }
      @condition.broadcast
    end
  end

  def reject!(error)
    @mutex.synchronize do
      @error = error
      @resolved = true
      @callbacks_failure.each { |cb| cb.call(error) }
      @condition.broadcast
    end
  end
end
```

### 4. `DockerEngine::Pipeline` — Secuencias sin callback hell

Pipeline resuelve el problema de encadenar múltiples operaciones secuenciales
sin caer en callbacks anidados. Define los pasos como datos declarativos y
los ejecuta en orden, pasando el resultado de cada paso al siguiente.

```ruby
class DockerEngine::Pipeline
  Step = Struct.new(:name, :action, :depends_on, keyword_init: true)

  def initialize(client)
    @client = client
    @steps = []
  end

  # Definir un paso. El bloque recibe el client y los resultados anteriores.
  def step(name, depends_on: nil, &action)
    @steps << Step.new(name: name, action: action, depends_on: depends_on)
    self
  end

  # Ejecutar todos los pasos en secuencia. Devuelve Future<Hash> con
  # los resultados indexados por nombre de paso.
  def execute
    Future.new do
      results = {}

      @steps.each do |step|
        begin
          result = step.action.call(@client, results)
          # Si el action devuelve un Future, esperarlo
          result = result.value if result.is_a?(Future)
          results[step.name] = result
        rescue => e
          raise PipelineError.new(
            "Failed at step :#{step.name}: #{e.message}",
            step: step.name,
            results_so_far: results,
            cause: e
          )
        end
      end

      results
    end
  end

  # Ejecutar pasos independientes en paralelo cuando no hay dependencias.
  def execute_parallel
    Future.new do
      results = Concurrent::Map.new
      remaining = @steps.dup

      while remaining.any?
        # Encontrar pasos cuyas dependencias ya están resueltas
        ready = remaining.select do |step|
          step.depends_on.nil? ||
            Array(step.depends_on).all? { |dep| results.key?(dep) }
        end

        raise PipelineError, "Circular dependency detected" if ready.empty?
        remaining -= ready

        # Ejecutar pasos ready en paralelo
        threads = ready.map do |step|
          Thread.new do
            result = step.action.call(@client, results.to_h)
            result = result.value if result.is_a?(Future)
            results[step.name] = result
          end
        end
        threads.each(&:join)
      end

      results.to_h
    end
  end
end

class DockerEngine::PipelineError < DockerEngine::Error
  attr_reader :step, :results_so_far, :cause

  def initialize(message, step: nil, results_so_far: {}, cause: nil)
    super(message)
    @step = step
    @results_so_far = results_so_far
    @cause = cause
  end
end
```

**Uso — secuencia de deploy:**

```ruby
pipeline = DockerEngine::Pipeline.new(client)

pipeline
  .step(:stop_old) { |c, _| c.containers.stop(name: "web-old", timeout: 30) }
  .step(:remove_old) { |c, _| c.containers.remove(name: "web-old") }
  .step(:run_new) { |c, _|
    c.containers.run(image: "myapp:v2", name: "web-new",
                     detach: true, network: "kamal")
  }
  .step(:health_check) { |c, _| c.containers.inspect(name: "web-new") }
  .step(:tag) { |c, _| c.images.tag(source: "myapp:v2", target: "myapp:latest") }

# Ejecutar y manejar resultado
pipeline.execute
  .on_success { |results| puts results[:health_check].parsed.dig("State", "Status") }
  .on_failure { |error| puts "Deploy failed at :#{error.step}" }
```

**Uso — pasos paralelos con dependencias:**

```ruby
pipeline = DockerEngine::Pipeline.new(client)

pipeline
  .step(:pull_app) { |c, _| c.images.pull(image: "myapp:v2") }
  .step(:pull_worker) { |c, _| c.images.pull(image: "myworker:v2") }
  .step(:stop_app, depends_on: :pull_app) { |c, _|
    c.containers.stop(name: "app", timeout: 30)
  }
  .step(:stop_worker, depends_on: :pull_worker) { |c, _|
    c.containers.stop(name: "worker", timeout: 30)
  }
  .step(:start_app, depends_on: :stop_app) { |c, prev|
    c.containers.run(image: "myapp:v2", name: "app", detach: true)
  }
  .step(:start_worker, depends_on: :stop_worker) { |c, prev|
    c.containers.run(image: "myworker:v2", name: "worker", detach: true)
  }

# pull_app y pull_worker corren en paralelo
# stop_app espera a pull_app, stop_worker espera a pull_worker
# start_app espera a stop_app, etc.
pipeline.execute_parallel
```

**Comparación de los 3 enfoques disponibles:**

```ruby
# ❌ Callback hell — NO hacer esto
client.async.containers.stop(name: "old").on_success do
  client.async.containers.remove(name: "old").on_success do
    client.async.containers.run(image: "v2", name: "new").on_success do
      # ... N niveles de indentación
    end
  end
end

# ✅ then encadenado — OK para 2-3 pasos simples
client.async.containers.stop(name: "old")
  .then { |_| client.containers.remove(name: "old") }
  .then { |_| client.containers.run(image: "v2", name: "new", detach: true) }
  .on_success { |result| puts "Done: #{result}" }
  .on_failure { |error| puts "Failed: #{error}" }

# ✅ Pipeline — Recomendado para secuencias complejas
pipeline = DockerEngine::Pipeline.new(client)
pipeline
  .step(:stop)   { |c, _| c.containers.stop(name: "old") }
  .step(:remove) { |c, _| c.containers.remove(name: "old") }
  .step(:run)    { |c, _| c.containers.run(image: "v2", name: "new", detach: true) }
  .execute
```

### 5. `DockerEngine::PendingStore` — Persistencia de operaciones asíncronas

Cuando el adaptador es asíncrono (WebSocket), necesitamos almacenar las
operaciones pendientes (`request_id` → `Future` o metadata) para poder
resolver la respuesta cuando llega. Hay dos estrategias según la arquitectura:

#### Estrategia A: Mismo proceso — `PendingStores::Memory`

Para cuando el proceso que envía es el mismo que recibe las respuestas WS.
Los Futures viven en memoria y se resuelven directamente.

```ruby
class DockerEngine::PendingStores::Memory < DockerEngine::PendingStore
  def initialize
    @store = Concurrent::Map.new  # thread-safe hash
  end

  def register(request_id, future)
    @store[request_id] = future
  end

  def resolve(request_id, result)
    future = @store.delete(request_id)
    return unless future
    future.send(:resolve!, result)
  end

  def reject(request_id, error)
    future = @store.delete(request_id)
    return unless future
    future.send(:reject!, error)
  end

  def pending?(request_id)
    @store.key?(request_id)
  end

  def pending_count
    @store.size
  end

  def reject_all(error)
    @store.each_value { |future| future.send(:reject!, error) }
    @store.clear
  end
end
```

#### Estrategia B: Procesos separados — `PendingStores::ActiveRecord`

Para cuando el emisor (ej: web request) y el receptor (ej: WS listener worker)
son procesos distintos. Los request_id se persisten en DB y la resolución
notifica al proceso original vía un canal de pub/sub.

```ruby
# Migración
# create_table :docker_pending_operations do |t|
#   t.string  :request_id, null: false, index: { unique: true }
#   t.string  :status, default: "pending"  # pending, completed, failed
#   t.jsonb   :operation                    # la Operation serializada
#   t.jsonb   :result                       # el Result cuando se resuelve
#   t.string  :error_message
#   t.string  :callback_channel             # canal pub/sub para notificar
#   t.timestamps
# end

class DockerEngine::PendingStores::ActiveRecord < DockerEngine::PendingStore
  def register(request_id, operation, callback_channel:)
    DockerPendingOperation.create!(
      request_id: request_id,
      status: "pending",
      operation: operation.to_h,
      callback_channel: callback_channel
    )
  end

  def resolve(request_id, result)
    record = DockerPendingOperation.find_by!(request_id: request_id)
    record.update!(status: "completed", result: result.to_h)

    # Notificar al proceso original que la respuesta está lista
    notify(record.callback_channel, {
      request_id: request_id,
      status: "completed",
      result: result.to_h
    })
  end

  def reject(request_id, error)
    record = DockerPendingOperation.find_by!(request_id: request_id)
    record.update!(status: "failed", error_message: error.message)

    notify(record.callback_channel, {
      request_id: request_id,
      status: "failed",
      error: error.message
    })
  end

  private

  # El mecanismo de notificación es intercambiable:
  # - ActionCable para apps Rails con WS al browser
  # - Redis pub/sub para comunicación entre workers
  # - PostgreSQL LISTEN/NOTIFY para apps sin Redis
  def notify(channel, payload)
    ActionCable.server.broadcast(channel, payload)
    # o: Redis.current.publish(channel, payload.to_json)
    # o: ActiveRecord::Base.connection.execute("NOTIFY #{channel}, '#{payload.to_json}'")
  end
end
```

**Flujo completo con procesos separados:**

```
┌─────────────────┐         ┌──────────────┐        ┌─────────────────┐
│   Web Process   │         │   Database   │        │  WS Listener    │
│   (Puma/Rails)  │         │  (Postgres)  │        │  (Sidekiq/etc)  │
└────────┬────────┘         └──────┬───────┘        └────────┬────────┘
         │                         │                          │
    1. client.containers           │                          │
       .stop(name:"web")           │                          │
         │                         │                          │
    2. Genera request_id           │                          │
       INSERT pending_op ────────→ │                          │
         │                         │                          │
    3. Envía msg al WS ────────────────────────────────────→  │
       gateway (vía Redis                                     │
       queue o HTTP)               │                          │
         │                         │                          │
    4. Devuelve request_id         │                          │
       al caller                   │                          │
         │                         │                     5. WS gateway
         │                         │                        recibe respuesta
         │                         │                          │
         │                         │ ←── UPDATE status ───────┤
         │                         │     = "completed"        │
         │                         │                          │
         │  ←── NOTIFY/broadcast ──│──────────────────────────┤
         │     (ActionCable,       │
         │      Redis pub/sub,     │
         │      PG NOTIFY)         │
         │                         │
    6. Callback se ejecuta         │
       con el resultado            │
```

**Configuración del adaptador WebSocket con cada store:**

```ruby
# Mismo proceso — todo en memoria
client = DockerEngine::Client.new(
  adapter: :websocket,
  url: "wss://gateway/ws",
  pending_store: :memory              # default
)

# Procesos separados — persistencia en DB
client = DockerEngine::Client.new(
  adapter: :websocket,
  url: "wss://gateway/ws",
  pending_store: :active_record,
  callback_channel: "docker_ops_#{Process.pid}"
)
```

### 6. `DockerEngine::CallbackExecutor` — Thread pool para callbacks

Cuando el WS listener recibe una respuesta y resuelve un Future, los callbacks
registrados con `on_success`/`on_failure` se ejecutan en el hilo del listener.
Si un callback hace trabajo pesado, bloquea la recepción de otros mensajes.

Solución: despachar callbacks a un thread pool dedicado.

```ruby
class DockerEngine::CallbackExecutor
  def initialize(pool_size: 5)
    @pool = Concurrent::FixedThreadPool.new(pool_size)
  end

  def dispatch(&block)
    @pool.post { block.call }
  end

  def shutdown(timeout: 10)
    @pool.shutdown
    @pool.wait_for_termination(timeout)
  end
end
```

El Future usa el executor cuando está configurado:

```ruby
class DockerEngine::Future
  def initialize(executor: nil, &block)
    @executor = executor
    # ... resto igual
  end

  def resolve!(result)
    @mutex.synchronize do
      @result = result
      @resolved = true
      @callbacks_success.each do |cb|
        if @executor
          @executor.dispatch { cb.call(result) }
        else
          cb.call(result)
        end
      end
      @condition.broadcast
    end
  end
end
```

**Resultado**: el listener WS procesa mensajes a máxima velocidad sin
bloquearse por callbacks lentos. Los callbacks se ejecutan en paralelo
en el thread pool.

```
WS Listener Thread          CallbackExecutor (pool de 5 threads)
      │                           │
      ├── msg arrives ──→ resolve!│
      │   (no bloquea)            ├── thread 1: callback A (pesado)
      │                           ├── thread 2: callback B (rápido)
      ├── msg arrives ──→ resolve!│
      │   (no bloquea)            ├── thread 3: callback C
      │                           │
      ├── msg arrives ──→ resolve!│
      │   (sigue recibiendo)      ├── thread 1: callback D (reusa thread)
      │                           │
```

### 7. `DockerEngine::ConnectionPool` — Pool genérico de conexiones

Pool thread-safe reutilizable por cualquier adaptador que necesite mantener
conexiones persistentes (SSH, WebSocket, TCP para HTTP API).

```ruby
class DockerEngine::ConnectionPool
  def initialize(size:, timeout: 5, idle_timeout: 300, &factory)
    @factory = factory         # bloque que crea una conexión nueva
    @size = size
    @timeout = timeout         # espera máxima para obtener conexión
    @idle_timeout = idle_timeout # tiempo sin uso antes de cerrar
    @mutex = Mutex.new
    @condition = ConditionVariable.new
    @connections = []          # conexiones disponibles [conn, last_used_at]
    @checked_out = Set.new     # conexiones en uso
    @created = 0               # total creadas (disponibles + en uso)
  end

  # Obtener conexión del pool, ejecutar bloque, devolver al pool.
  def with(&block)
    conn = checkout
    begin
      yield conn
    ensure
      checkin(conn)
    end
  end

  # Cerrar todas las conexiones.
  def shutdown
    @mutex.synchronize do
      @connections.each { |conn, _| close_connection(conn) }
      @connections.clear
      @checked_out.each { |conn| close_connection(conn) }
      @checked_out.clear
      @created = 0
    end
  end

  # Número de conexiones disponibles.
  def available
    @mutex.synchronize { @connections.size }
  end

  # Número de conexiones en uso.
  def in_use
    @mutex.synchronize { @checked_out.size }
  end

  private

  def checkout
    @mutex.synchronize do
      # Limpiar conexiones idle expiradas
      reap_idle

      loop do
        # 1. Intentar reutilizar una conexión disponible
        if (entry = @connections.pop)
          conn, _ = entry
          if alive?(conn)
            @checked_out.add(conn)
            return conn
          else
            close_connection(conn)
            @created -= 1
          end
        # 2. Crear una nueva si no se alcanzó el máximo
        elsif @created < @size
          conn = @factory.call
          @created += 1
          @checked_out.add(conn)
          return conn
        # 3. Esperar a que se libere una
        else
          deadline = Process.clock_gettime(Process::CLOCK_MONOTONIC) + @timeout
          @condition.wait(@mutex, @timeout)
          if Process.clock_gettime(Process::CLOCK_MONOTONIC) >= deadline && @connections.empty?
            raise DockerEngine::ConnectionError,
              "Could not obtain connection from pool within #{@timeout}s " \
              "(size: #{@size}, in_use: #{@checked_out.size})"
          end
        end
      end
    end
  end

  def checkin(conn)
    @mutex.synchronize do
      @checked_out.delete(conn)
      if alive?(conn)
        @connections.push([conn, Process.clock_gettime(Process::CLOCK_MONOTONIC)])
      else
        close_connection(conn)
        @created -= 1
      end
      @condition.signal
    end
  end

  def reap_idle
    now = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    @connections.reject! do |conn, last_used|
      if now - last_used > @idle_timeout
        close_connection(conn)
        @created -= 1
        true
      end
    end
  end

  def alive?(conn)
    return conn.open? if conn.respond_to?(:open?)
    return !conn.closed? if conn.respond_to?(:closed?)
    true
  end

  def close_connection(conn)
    conn.close if conn.respond_to?(:close)
  rescue => e
    # log but don't raise
  end
end
```

### 8. `DockerEngine::Result` — Respuesta estandarizada

```ruby
class DockerEngine::Result
  attr_reader :output     # String - stdout o body JSON parseado
  attr_reader :exit_code  # Integer - 0 para éxito (HTTP: 2xx → 0)
  attr_reader :error      # String - stderr o mensaje de error

  def initialize(output: "", exit_code: 0, error: nil)
    @output = output
    @exit_code = exit_code
    @error = error
  end

  def success?
    exit_code == 0
  end

  def to_s
    output
  end

  def parsed
    JSON.parse(output)
  end
end
```

### 9. Resources — Métodos por recurso

Los resources no conocen el adaptador directamente. Construyen una `Operation`
(descripción declarativa) y la pasan al adaptador. Esto permite que cualquier
adaptador (síncrono o asíncrono) la interprete.

```ruby
# Estructura intermedia que describe la operación a ejecutar
class DockerEngine::Operation
  attr_reader :resource    # :container, :image, :network, ...
  attr_reader :action      # :run, :stop, :list, :pull, ...
  attr_reader :params      # Hash con todos los parámetros tipados

  def initialize(resource:, action:, **params)
    @resource = resource
    @action = action
    @params = params
  end
end
```

```ruby
class DockerEngine::Resources::Container
  def initialize(adapter)
    @adapter = adapter
  end

  def run(image:, name:, detach: true, restart: nil, network: nil,
          hostname: nil, env: {}, volumes: [], labels: {}, ports: [],
          options: [], cmd: nil)
    op = Operation.new(
      resource: :container, action: :run,
      image:, name:, detach:, restart:, network:, hostname:,
      env:, volumes:, labels:, ports:, options:, cmd:
    )
    @adapter.execute(op)
  end

  def stop(name:, timeout: nil, signal: nil)
    op = Operation.new(resource: :container, action: :stop, name:, timeout:, signal:)
    @adapter.execute(op)
  end

  # ... demás métodos igual
end
```

Cada adaptador recibe `Operation` y la traduce a su mecanismo:

- **Cli** → convierte Operation en string `"docker run --detach ..."` y ejecuta con Open3
- **Ssh** → igual pero ejecuta sobre una conexión SSH del pool
- **HttpApi** → convierte Operation en `POST /v1.45/containers/create` + body JSON
- **WebSocket** → serializa Operation como mensaje JSON, envía por WS, espera respuesta

#### 9.1 `DockerEngine::Resources::Container`

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

#### 9.2 `DockerEngine::Resources::Image`

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

#### 9.3 `DockerEngine::Resources::Network`

| Método | Parámetros | CLI | HTTP API |
|--------|-----------|-----|----------|
| `create` | `name:, driver: nil, labels: {}` | `docker network create [--driver ...] <name>` | `POST /networks/create` |

#### 9.4 `DockerEngine::Resources::Registry`

| Método | Parámetros | CLI | HTTP API |
|--------|-----------|-----|----------|
| `login` | `server:, username:, password:` | `docker login <server> -u <user> -p <pass>` | `POST /auth` |
| `logout` | `server:` | `docker logout <server>` | N/A (no hay endpoint, es local) |

#### 9.5 `DockerEngine::Resources::Builder`

| Método | Parámetros | CLI | HTTP API |
|--------|-----------|-----|----------|
| `create` | `name:, driver: nil, platform: nil, append: false, driver_opts: [], context: nil` | `docker buildx create --name ... [--driver ...] [--append] [context]` | N/A |
| `remove` | `name:` | `docker buildx rm <name>` | N/A |
| `list` | — | `docker buildx ls` | N/A |
| `inspect` | `name:` | `docker buildx inspect <name>` | N/A |

> **Nota:** Buildx no tiene API HTTP. El adaptador HttpApi lanzará
> `DockerEngine::UnsupportedOperationError` para estas operaciones.

#### 9.6 `DockerEngine::Resources::Context`

| Método | Parámetros | CLI | HTTP API |
|--------|-----------|-----|----------|
| `create` | `name:, description: nil, docker_host:` | `docker context create <name> --docker 'host=...'` | N/A |
| `list` | — | `docker context ls` | N/A |
| `inspect` | `name:, format: nil` | `docker context inspect <name> [--format ...]` | N/A |
| `remove` | `name:` | `docker context rm <name>` | N/A |

> **Nota:** Contextos Docker son locales al cliente. Solo CLI y SSH los soportan.

#### 9.7 `DockerEngine::Resources::System`

| Método | Parámetros | CLI | HTTP API |
|--------|-----------|-----|----------|
| `version` | — | `docker version` | `GET /version` |
| `client_version` | — | `docker -v` | N/A |
| `info` | `format: nil` | `docker info [--format ...]` | `GET /system/info` |

### 10. Adaptadores — Implementación

#### 10.1 `DockerEngine::Adapters::Cli` (síncrono)

Ejecuta comandos mediante `Open3.capture3` en la máquina local.

```ruby
class DockerEngine::Adapters::Cli < DockerEngine::Adapter
  def execute(operation)
    cmd = CommandBuilder.build(operation)
    stdout, stderr, status = Open3.capture3(*cmd)
    Result.new(output: stdout, error: stderr, exit_code: status.exitstatus)
  end

  def stream(operation, &block)
    cmd = CommandBuilder.build(operation)
    Open3.popen3(*cmd) do |_stdin, stdout, stderr, wait_thr|
      stdout.each_line { |line| block.call(line) }
      wait_thr.value
    end
  end
end
```

#### 10.2 `DockerEngine::Adapters::Ssh` (síncrono + pool de conexiones)

Ejecuta comandos en servidor remoto. Usa `ConnectionPool` para reutilizar
conexiones SSH. Varias operaciones comparten las mismas conexiones sin
necesidad de abrir/cerrar por cada comando.

```ruby
class DockerEngine::Adapters::Ssh < DockerEngine::Adapter
  def initialize(host:, user: "root", port: 22, keys: [],
                 proxy: nil, pool_size: 5, idle_timeout: 300)
    @pool = DockerEngine::ConnectionPool.new(
      size: pool_size,
      idle_timeout: idle_timeout
    ) do
      # Factory: crear conexión SSH nueva
      Net::SSH.start(host, user, port: port, keys: keys, proxy: proxy,
                     non_interactive: true, verify_host_key: :accept_new)
    end
  end

  def execute(operation)
    cmd = CommandBuilder.build(operation)
    cmd_string = cmd.shelljoin

    @pool.with do |ssh|
      output = ""
      error = ""
      exit_code = nil

      channel = ssh.open_channel do |ch|
        ch.exec(cmd_string) do |_, success|
          raise ConnectionError, "SSH exec failed" unless success

          ch.on_data { |_, data| output << data }
          ch.on_extended_data { |_, _, data| error << data }
          ch.on_request("exit-status") { |_, buf| exit_code = buf.read_long }
        end
      end
      channel.wait

      Result.new(output: output, error: error, exit_code: exit_code || 1)
    end
  end

  def stream(operation, &block)
    cmd = CommandBuilder.build(operation)
    cmd_string = cmd.shelljoin

    @pool.with do |ssh|
      channel = ssh.open_channel do |ch|
        ch.exec(cmd_string) do |_, success|
          raise ConnectionError, "SSH exec failed" unless success
          ch.on_data { |_, data| block.call(data) }
        end
      end
      ssh.loop { channel.active? }
    end
  end

  def close
    @pool.shutdown
  end
end
```

**Cómo funciona el pool SSH:**

```
Thread 1: client.containers.stop("web-1")  ──→ pool.with { |ssh| ... } ──→ usa conexión A
Thread 2: client.containers.stop("web-2")  ──→ pool.with { |ssh| ... } ──→ usa conexión B
Thread 3: client.containers.stop("web-3")  ──→ pool.with { |ssh| ... } ──→ usa conexión C
                                                                              (o espera si pool_size=2)

# Cuando Thread 1 termina, conexión A vuelve al pool.
# Thread 3 la reutiliza sin hacer nuevo SSH handshake.

# Tras idle_timeout segundos sin uso, la conexión se cierra automáticamente.
# Si se pide una conexión y todas están en uso + se alcanzó pool_size,
# el thread espera hasta @timeout segundos o lanza ConnectionError.
```

#### 10.3 `DockerEngine::Adapters::HttpApi` (síncrono)

Conecta al Docker Engine API vía Unix socket o TCP con TLS opcional.

```ruby
class DockerEngine::Adapters::HttpApi < DockerEngine::Adapter
  API_VERSION = "v1.45"

  def initialize(url: "unix:///var/run/docker.sock", tls: nil)
    @url = url
    @tls = tls
    @translator = HttpTranslator.new(API_VERSION)
  end

  def execute(operation)
    request = @translator.translate(operation)
    # request = { method: :post, path: "/containers/create", query: {}, body: {} }

    response = http_request(request)
    Result.new(
      output: response.body,
      exit_code: response.code.to_i < 400 ? 0 : 1,
      error: response.code.to_i >= 400 ? response.body : nil
    )
  end

  def stream(operation, &block)
    request = @translator.translate(operation)
    http_stream(request) do |chunk|
      block.call(chunk)
    end
  end
end
```

#### 10.4 `DockerEngine::Adapters::WebSocket` (asíncrono nativo)

Un adaptador cuyo transporte es inherentemente asíncrono. No puede devolver
`Result` directamente porque la respuesta llega en otro momento.

Usa `PendingStore` para almacenar operaciones pendientes y `CallbackExecutor`
para despachar callbacks sin bloquear el listener.

```ruby
class DockerEngine::Adapters::WebSocket < DockerEngine::Adapter
  def initialize(url:, headers: {},
                 pending_store: :memory,
                 callback_pool_size: 5, **store_options)
    @url = url
    @headers = headers
    @pending_store = resolve_store(pending_store, **store_options)
    @callback_executor = CallbackExecutor.new(pool_size: callback_pool_size)
    @on_stream = Concurrent::Map.new
    @connection = nil
  end

  def async?
    true
  end

  def execute(operation)
    ensure_connected!
    request_id = SecureRandom.uuid

    # Crear Future con executor para que los callbacks no bloqueen el listener
    future = DockerEngine::Future.new(executor: @callback_executor)

    # Registrar en el store (memoria o DB)
    @pending_store.register(request_id, future)

    # Enviar operación serializada por WebSocket
    message = {
      id: request_id,
      resource: operation.resource,
      action: operation.action,
      params: operation.params
    }.to_json

    @connection.send(message)

    future
  end

  def stream(operation, &block)
    ensure_connected!
    request_id = SecureRandom.uuid

    message = {
      id: request_id,
      resource: operation.resource,
      action: operation.action,
      params: operation.params,
      stream: true
    }.to_json

    @on_stream[request_id] = block
    @connection.send(message)
  end

  def close
    @connection&.close
    @pending_store.reject_all(ConnectionError.new("closed"))
    @callback_executor.shutdown
  end

  private

  def resolve_store(type, **options)
    case type
    when :memory        then PendingStores::Memory.new
    when :active_record then PendingStores::ActiveRecord.new(**options)
    when Class          then type.new(**options)
    else raise ArgumentError, "Unknown pending store: #{type}"
    end
  end

  def ensure_connected!
    return if @connection&.open?

    @connection = WebSocketClient.connect(@url, headers: @headers)

    # Listener: recibe respuestas y resuelve vía el pending_store.
    # Este bloque corre en el hilo del WS listener.
    # Los callbacks se despachan al CallbackExecutor (otro thread pool),
    # así que este hilo nunca se bloquea.
    @connection.on(:message) do |event|
      data = JSON.parse(event.data)
      request_id = data["id"]

      if (stream_handler = @on_stream[request_id])
        if data["done"]
          @on_stream.delete(request_id)
        else
          @callback_executor.dispatch { stream_handler.call(data["output"]) }
        end
      else
        result = Result.new(
          output: data["output"],
          exit_code: data["exit_code"],
          error: data["error"]
        )
        if result.success?
          @pending_store.resolve(request_id, result)
        else
          @pending_store.reject(request_id, CommandError.new(result))
        end
      end
    end

    @connection.on(:close) do
      @pending_store.reject_all(ConnectionError.new("disconnected"))
    end
  end
end
```

**Cómo funciona el flujo asíncrono:**

```
                    SÍNCRONO (CLI, SSH, HttpApi)
                    ═══════════════════════════
  caller             adapter              docker
    │                   │                    │
    ├── execute(op) ──→ │                    │
    │   (bloquea)       ├── run command ───→ │
    │                   │                    ├── procesa
    │                   │  ←── resultado ────┤
    │  ←── Result ──────┤                    │
    │                   │                    │

                    ASÍNCRONO (WebSocket)
                    ════════════════════
  caller             adapter              servidor WS
    │                   │                    │
    ├── execute(op) ──→ │                    │
    │  ←── Future ──────┤                    │
    │                   ├── send(msg) ──────→│
    │ (no bloquea,      │                    ├── procesa...
    │  sigue trabajando) │                    │   (puede tardar)
    │                   │  ←── on(:message) ─┤
    │                   ├── resolve!(result)  │
    │                   │                    │
    ├── future.value ─→ │ ← Result           │  (si necesita bloquear)
    │   ó                                    │
    ├── future.on_success { |r| ... }        │  (callback, no bloquea)
```

### 11. El Client: Unificando Sync y Async

El Client ofrece una interfaz `.async` que envuelve cualquier adaptador
(incluso los síncronos) en ejecución asíncrona:

```ruby
class DockerEngine::Client
  def initialize(adapter:, **options)
    @adapter = resolve_adapter(adapter, **options)
  end

  # Acceso a resources (síncrono por defecto)
  def containers
    @containers ||= Resources::Container.new(@adapter)
  end

  def images
    @images ||= Resources::Image.new(@adapter)
  end

  # ... demás resources

  # Wrapper asíncrono: envuelve el adaptador para que todo
  # devuelva Future<Result>, independientemente del tipo de adaptador.
  def async
    @async_client ||= AsyncProxy.new(self)
  end

  def close
    @adapter.close
  end

  private

  def resolve_adapter(type, **options)
    case type
    when :cli       then Adapters::Cli.new(**options)
    when :ssh       then Adapters::Ssh.new(**options)
    when :http_api  then Adapters::HttpApi.new(**options)
    when :websocket then Adapters::WebSocket.new(**options)
    when Class      then type.new(**options)  # adaptador custom
    else raise ArgumentError, "Unknown adapter: #{type}"
    end
  end
end

# Proxy que envuelve llamadas síncronas en Future automáticamente
class DockerEngine::AsyncProxy
  def initialize(client)
    @client = client
  end

  def containers
    @containers ||= AsyncResourceProxy.new(@client.containers)
  end

  def images
    @images ||= AsyncResourceProxy.new(@client.images)
  end

  # ... demás resources
end

class DockerEngine::AsyncResourceProxy
  def initialize(resource)
    @resource = resource
  end

  # Cualquier llamada al resource se envuelve en un Future
  def method_missing(method, *args, **kwargs, &block)
    if @resource.respond_to?(method)
      Future.new { @resource.send(method, *args, **kwargs, &block) }
    else
      super
    end
  end

  def respond_to_missing?(method, include_private = false)
    @resource.respond_to?(method, include_private) || super
  end
end
```

### 12. Manejo de Operaciones No Soportadas

| Operación | CLI | SSH | HTTP API | WebSocket |
|-----------|-----|-----|----------|-----------|
| `builder.*` (buildx) | OK | OK | `UnsupportedOperationError` | depende del servidor |
| `context.*` | OK | OK | `UnsupportedOperationError` | depende del servidor |
| `registry.logout` | OK | OK | `UnsupportedOperationError` | depende del servidor |
| `container.copy_from/to` | OK | OK | OK | depende del servidor |
| `container.logs(follow: true)` | OK (stream) | OK (stream) | OK (stream) | OK (stream) |

### 13. Jerarquía de Errores

```ruby
module DockerEngine
  class Error < StandardError; end
  class ConnectionError < Error; end              # No se puede conectar
  class ContainerNotFoundError < Error; end       # Contenedor no existe
  class ImageNotFoundError < Error; end           # Imagen no existe
  class AuthenticationError < Error; end          # Fallo de login en registro
  class CommandError < Error                      # Comando falló (exit code != 0)
    attr_reader :result
  end
  class UnsupportedOperationError < Error; end    # Operación no soportada por adaptador
  class TimeoutError < Error; end                 # Timeout en la operación
  class PoolExhaustedError < ConnectionError; end # Pool sin conexiones disponibles
end
```

### 14. Ejemplo de Uso Completo

```ruby
# ──── Adaptador CLI local ────
client = DockerEngine::Client.new(adapter: :cli)

# ──── Adaptador SSH con pool (5 conexiones, idle 5 min) ────
client = DockerEngine::Client.new(
  adapter: :ssh,
  host: "production.server.com",
  user: "deploy",
  port: 22,
  keys: ["~/.ssh/deploy_key"],
  pool_size: 5,
  idle_timeout: 300
)

# ──── Adaptador HTTP API vía socket Unix ────
client = DockerEngine::Client.new(
  adapter: :http_api,
  url: "unix:///var/run/docker.sock"
)

# ──── Adaptador WebSocket asíncrono ────
client = DockerEngine::Client.new(
  adapter: :websocket,
  url: "wss://docker-gateway.example.com/ws"
)

# ──── Adaptador custom ────
client = DockerEngine::Client.new(adapter: MyCustomAdapter, url: "...")


# === Uso síncrono (CLI, SSH, HttpApi) ===

client.registries.login(server: "ghcr.io", username: "user", password: "token")
client.networks.create(name: "kamal")
client.containers.run(
  image: "myapp:v1.0", name: "web", detach: true,
  restart: "unless-stopped", network: "kamal",
  env: { "RAILS_ENV" => "production" },
  labels: { "service" => "myapp" }
)
result = client.containers.list(filters: { label: ["service=myapp"] })
client.containers.stop(name: "web", timeout: 30)


# === Uso asíncrono (cualquier adaptador) ===

# Opción 1: callbacks (nunca bloquea)
client.async.containers.stop(name: "web-1").on_success { |r| puts "1 stopped" }
client.async.containers.stop(name: "web-2").on_success { |r| puts "2 stopped" }
client.async.containers.stop(name: "web-3").on_success { |r| puts "3 stopped" }

# Opción 2: recoger futures y esperar resultados
futures = servers.map do |server|
  client.async.containers.run(
    image: "myapp:v2.0", name: "web-#{server}", detach: true
  )
end
results = futures.map(&:value)  # bloquea hasta que todos terminen

# Opción 3: encadenar transformaciones
client.async.containers.inspect(name: "web")
  .then { |result| result.parsed.dig("State", "Health", "Status") }
  .on_success { |status| puts "Health: #{status}" }
  .on_failure { |error| puts "Error: #{error.message}" }

# Opción 4: con timeout
begin
  result = client.async.images.pull(image: "myapp:v2.0").value(timeout: 60)
rescue DockerEngine::TimeoutError
  puts "Pull took too long"
end


# === Pool SSH en acción: deploy paralelo ===

ssh_client = DockerEngine::Client.new(
  adapter: :ssh, host: "prod", user: "deploy", pool_size: 10
)

# 10 operaciones paralelas reutilizando conexiones SSH
threads = 10.times.map do |i|
  Thread.new do
    ssh_client.containers.stop(name: "worker-#{i}", timeout: 30)
    ssh_client.containers.remove(name: "worker-#{i}")
    ssh_client.containers.run(image: "myapp:v2", name: "worker-#{i}", detach: true)
  end
end
threads.each(&:join)

# Las 10 operaciones comparten máximo 10 conexiones SSH del pool.
# Al final, las conexiones quedan en el pool para futuras operaciones.
# Tras 300s sin uso, se cierran automáticamente.

ssh_client.close  # cierre explícito cuando ya no se necesita


# === WebSocket nativo (async por defecto) ===

ws_client = DockerEngine::Client.new(
  adapter: :websocket,
  url: "wss://docker-gateway.internal/ws"
)

# execute() ya devuelve Future directamente (adaptador async nativo)
future = ws_client.containers.stop(name: "web")
future.on_success { |r| puts "Done" }
# no hay bloqueo en ningún momento

ws_client.close
```

## Plan de Implementación

### Fase 1: Core
1. `DockerEngine::Operation` (estructura declarativa de comandos)
2. `DockerEngine::Result` y `DockerEngine::Errors`
3. `DockerEngine::Future` (promesas con callbacks, `then` con flat-map, bloqueo)
4. `DockerEngine::CallbackExecutor` (thread pool para callbacks)
5. `DockerEngine::ConnectionPool` (pool genérico thread-safe)
6. `DockerEngine::Pipeline` (secuencias declarativas con dependencias)
7. `DockerEngine::Adapter` (clase base abstracta)
8. `DockerEngine::Client` (con `async` proxy)

### Fase 2: Adaptador CLI + Resources
9. `DockerEngine::Adapters::Cli` (ejecución local con Open3)
10. `CommandBuilder` (Operation → array de strings para shell)
11. Todos los Resources (Container, Image, Network, Registry, System, Builder, Context)
12. Tests unitarios para cada resource con CLI
13. Tests de Pipeline (secuencial y paralelo)

### Fase 3: Adaptador SSH
14. `DockerEngine::Adapters::Ssh` (net-ssh + ConnectionPool)
15. Tests con SSH mockeado
16. Tests de pool de conexiones (concurrencia, idle timeout, reaping)

### Fase 4: Adaptador HTTP API
17. `DockerEngine::Adapters::HttpApi` (REST via socket/TCP)
18. `HttpTranslator` (Operation → HTTP method + path + body)
19. Manejo de UnsupportedOperationError para buildx/context
20. Tests con HTTP stubbed

### Fase 5: Adaptador WebSocket + Async
21. `DockerEngine::PendingStore` (interfaz) + `PendingStores::Memory`
22. `DockerEngine::PendingStores::ActiveRecord` (persistencia DB)
23. `DockerEngine::Adapters::WebSocket` (async nativo con PendingStore + CallbackExecutor)
24. `AsyncProxy` y `AsyncResourceProxy`
25. Tests con WebSocket mockeado
26. Tests de PendingStore (memory y active_record)

### Fase 6: Volumen (si necesario)
27. `DockerEngine::Resources::Volume` (create, ls, rm, prune)
