# API Справочник / API Reference

## Net (фасад / facade)

```java
Net.server()    // → NetworkServerBuilder
Net.client()    // → NetworkClientBuilder
Net.udp()       // → NetworkUdpBuilder
Net.registry()  // → PacketRegistry (пустой / empty)
```

---

## NetworkServerBuilder

| Метод | По умолчанию | Описание |
|-------|-------------|----------|
| `.host(String)` | `"0.0.0.0"` | Адрес бинда |
| `.port(int)` | `25565` | Порт |
| `.codec(PacketRegistry)` | empty | Реестр пакетов |
| `.bossThreads(int)` | `1` | Потоки принятия |
| `.workerThreads(int)` | `CPU * 2` | Потоки I/O |
| `.listener(Object)` | — | `@NetworkListener` инстанс |
| `.interceptor(PacketInterceptor)` | — | Добавить в pipeline |
| `.encrypt(SecretKey)` | `null` | AES-128-GCM (Netty-уровень) |
| `.compress(int threshold)` | `-1` (выкл) | zlib deflate (Netty-уровень) |
| `.option(ChannelOption, T)` | — | Netty channel option |
| `.start()` | — | **Запустить** → `NetworkServer` |

---

## NetworkServer

```java
server.onConnect(Consumer<NetworkContext>)     // → this (fluent)
server.onDisconnect(Consumer<NetworkContext>)  // → this (fluent)
server.broadcast(Packet)
server.connections()                           // → Collection<NetworkContext>
server.isRunning()                             // → boolean
server.shutdown()
server.close()                                 // alias for shutdown()
```

---

## NetworkClientBuilder

| Метод | По умолчанию | Описание |
|-------|-------------|----------|
| `.host(String)` | `"localhost"` | Хост сервера |
| `.port(int)` | `25565` | Порт |
| `.codec(PacketRegistry)` | empty | Реестр пакетов |
| `.listener(Object)` | — | `@NetworkListener` инстанс |
| `.interceptor(PacketInterceptor)` | — | Добавить в pipeline |
| `.encrypt(SecretKey)` | `null` | AES-128-GCM (Netty-уровень) |
| `.compress(int threshold)` | `-1` (выкл) | zlib deflate (Netty-уровень) |
| `.reconnect(boolean)` | `false` | Авто-переподключение |
| `.reconnectDelay(Duration)` | `5s` | Задержка переподключения |
| `.connect(String, int)` | — | **Async** → `CompletableFuture<NetworkClient>` |
| `.get()` | — | **Blocking** → `NetworkClient` |

---

## NetworkClient

```java
client.send(Packet)
client.sendAsync(Packet)     // → CompletableFuture<Void>
client.disconnect()
client.isConnected()          // → boolean
client.context()              // → NetworkContext
client.close()                // alias for disconnect()
```

---

## NetworkUdpBuilder

| Метод | По умолчанию | Описание |
|-------|-------------|----------|
| `.host(String)` | `"0.0.0.0"` | Адрес бинда |
| `.port(int)` | `0` (эфемерный) | Порт |
| `.codec(PacketRegistry)` | empty | Реестр пакетов |
| `.listener(Object)` | — | `@NetworkListener` инстанс |
| `.bind()` | — | **Серверная роль** → `UdpEndpoint` |
| `.connect(String, int)` | — | **Клиентская роль** → `ConnectedUdpClient` |

## UdpEndpoint

```java
endpoint.send(Packet, InetSocketAddress)
endpoint.broadcast(Packet, Iterable<InetSocketAddress>)
endpoint.localAddress()    // → InetSocketAddress
endpoint.isOpen()          // → boolean
endpoint.shutdown()
endpoint.close()           // alias for shutdown()
```

## ConnectedUdpClient

```java
client.send(Packet)                         // адрес зафиксирован / address fixed
client.sendAsync(Packet)                    // → CompletableFuture<Void>
client.serverAddress()                      // → InetSocketAddress
client.isOpen()                             // → boolean
client.disconnect()
client.close()
```

---

## NetworkContext

```java
ctx.send(Packet)
ctx.sendRaw(byte[])
ctx.disconnect(String reason)
ctx.getRemoteAddress()        // → InetSocketAddress
ctx.getPipeline()             // → PacketPipeline
ctx.attr(AttributeKey<T>)    // → Attribute<T>
ctx.isActive()                // → boolean
```

---

## PacketPipeline

```java
pipeline.addFirst(String name, PacketInterceptor)    // → this
pipeline.addLast(String name, PacketInterceptor)     // → this
pipeline.addBefore(String pivot, String name, PacketInterceptor) // → this
pipeline.remove(String name)                         // → this
pipeline.fireRead(Packet, NetworkContext)
pipeline.fireWrite(Packet, NetworkContext)
```

---

## PacketBuffer

```java
// Создание / Creation
PacketBuffer.allocate(int capacity)
PacketBuffer.wrap(byte[] data)

// Запись / Write
buf.writeVarInt(int)
buf.writeLong(long)
buf.writeDouble(double)
buf.writeBoolean(boolean)
buf.writeString(String)
buf.writeBytes(byte[])

// Чтение / Read
buf.readVarInt()       // → int
buf.readLong()         // → long
buf.readDouble()       // → double
buf.readBoolean()      // → boolean
buf.readString()       // → String
buf.readBytes(int len) // → byte[]

// Утилиты / Utilities
buf.toArray()          // → byte[]
buf.readableBytes()    // → int
buf.writerIndex()      // → int
buf.slice()            // → PacketBuffer (zero-copy)
buf.resetReaderIndex() // → this
```

---

## PacketRegistry

```java
PacketRegistry.create()                                    // → DefaultPacketRegistry
registry.register(int id, PacketCodec<T>)
registry.register(int id, Class<T>, PacketCodec<T>)        // DefaultPacketRegistry only
registry.lookup(int id)                                    // → Optional<PacketCodec<?>>
registry.idFor(Class<? extends Packet>)                    // → int
registry.scanPackage(String basePackage)
```

---

## Annotations

```java
@PacketId(int value)                          // на record / on record
@Codec(Class<? extends PacketCodec<?>> value) // на record / on record
@NetworkListener                               // на class / on class
@PacketHandler(int priority = 0)              // на метод / on method
@Interceptor(int order = 0)                   // на class / on class
```

---

## AttributeKey / Attribute

```java
AttributeKey<T> key = AttributeKey.valueOf(String name)  // singleton
key.name()  // → String

Attribute<T> attr = ctx.attr(key)
attr.get()                          // → T (nullable)
attr.set(T value)
attr.compareAndSet(T expected, T update)  // → boolean
```

---

## Built-in PacketInterceptors (pipeline)

```java
// Реализуют PacketInterceptor — работают через .interceptor() / Implement PacketInterceptor
new LoggingInterceptor()
new LoggingInterceptor(Level level)

new RateLimitInterceptor(double tokensPerSecond, long burstCapacity)
```

## Utility Classes (NOT PacketInterceptor)

```java
// НЕ реализуют PacketInterceptor — только утилитные методы
// Do NOT implement PacketInterceptor — utility methods only
// Use .compress(threshold) / .encrypt(key) on Builder for wire-level transforms

new CompressionInterceptor(int threshold)
// .compress(byte[])               → byte[]   (zlib deflate)
// .decompress(byte[], int hint)   → byte[]   (zlib inflate)
// .threshold()                    → int

new EncryptionInterceptor(SecretKey key)
// .encrypt(byte[])                → byte[]   (AES-128-GCM, random IV prepended)
// .decrypt(byte[])                → byte[]
```
