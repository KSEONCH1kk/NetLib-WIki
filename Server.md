# Сервер / Server

## Запуск / Starting

```java
NetworkServer server = Net.server()
        .host("0.0.0.0")                    // по умолчанию / default
        .port(25565)
        .codec(registry)
        .bossThreads(1)                     // потоки принятия / accept threads
        .workerThreads(8)                   // потоки I/O / I/O threads
        .listener(new GameHandler())
        .interceptor(new LoggingInterceptor())
        .encrypt(secretKey)                 // опционально / optional
        .start();                           // блокирует до бинда / blocks until bound
```

`.start()` бросает `ConnectionException` если порт занят.  
> `.start()` throws `ConnectionException` if the port is already in use.

---

## События соединения / Connection Events

```java
server.onConnect(ctx -> {
    System.out.println("New player: " + ctx.getRemoteAddress());
    // сохранить ctx для broadcast / store ctx for broadcast
});

server.onDisconnect(ctx -> {
    System.out.println("Player left: " + ctx.getRemoteAddress());
});
```

Обработчики вызываются из Netty worker-потока. Не блокируй их.  
> Handlers are called from Netty worker thread. Do not block.

---

## Broadcast

```java
server.broadcast(new AnnouncementPacket("Server restart in 1 minute"));
```

Отправляет пакет всем активным соединениям. Неактивные соединения пропускаются.  
> Sends to all active connections. Inactive connections are skipped.

---

## Список соединений / Connection List

```java
Collection<NetworkContext> all = server.connections();
int count = all.size();

for (NetworkContext ctx : all) {
    if (ctx.isActive()) {
        ctx.send(new PingPacket());
    }
}
```

---

## Остановка / Shutdown

```java
server.shutdown();

// или try-with-resources / or try-with-resources
try (NetworkServer server = Net.server().port(25565).start()) {
    // ...
}
```

`shutdown()` закрывает все соединения и останавливает EventLoopGroup.  
> `shutdown()` closes all connections and shuts down EventLoopGroups.

---

## Netty Channel Options

```java
Net.server()
    .option(ChannelOption.SO_BACKLOG, 128)
    .option(ChannelOption.TCP_NODELAY, true)
    .option(ChannelOption.SO_KEEPALIVE, true)
    .start();
```

---

## Несколько слушателей / Multiple Listeners

```java
Net.server()
    .listener(new GameHandler())
    .listener(new AdminHandler())
    .listener(new MetricsHandler())
    .start();
```

Все обработчики регистрируются в `PacketHandlerRegistry` по приоритету (`@PacketHandler(priority = 10)`).

---

## Потоки / Threads

| Группа | Назначение | По умолчанию |
|--------|------------|--------------|
| bossGroup | Принятие соединений | 1 |
| workerGroup | I/O, декодирование | CPU * 2 |

Netty гарантирует, что все события одного соединения обрабатываются в одном потоке.  
> Netty guarantees all events for a single connection are handled in one thread.
