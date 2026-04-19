# Клиент / Client

## Подключение / Connecting

### Асинхронное / Async

```java
CompletableFuture<NetworkClient> future = Net.client()
        .codec(registry)
        .listener(new GameHandler())
        .connect("localhost", 25565);

future.thenAccept(client -> {
    client.send(new LoginPacket("player1", "token"));
}).exceptionally(ex -> {
    System.err.println("Connection failed: " + ex.getMessage());
    return null;
});
```

### Синхронное / Blocking

```java
NetworkClient client = Net.client()
        .codec(registry)
        .listener(new GameHandler())
        .connect("localhost", 25565)
        .get();           // блокирует / blocks

client.send(new LoginPacket("player1", "token"));
```

---

## Отправка пакетов / Sending Packets

```java
// Синхронно / Sync — блокирует до записи в буфер / blocks until written to buffer
client.send(new ChatPacket("Hello!"));

// Асинхронно / Async — возвращает Future / returns Future
client.sendAsync(new ChatPacket("Hello!"))
      .exceptionally(ex -> { log.error("Send failed", ex); return null; });
```

---

## Переподключение / Reconnection

```java
NetworkClient client = Net.client()
        .codec(registry)
        .reconnect(true)
        .reconnectDelay(Duration.ofSeconds(3))
        .connect("localhost", 25565)
        .get();
```

При потере соединения клиент автоматически повторяет попытку каждые `reconnectDelay`.  
> On connection loss, the client retries automatically every `reconnectDelay`.

---

## Шифрование / Encryption

```java
NetworkClient client = Net.client()
        .codec(registry)
        .encrypt(secretKey)   // тот же ключ что у сервера / same key as server
        .connect("localhost", 25565)
        .get();
```

---

## Состояние / State

```java
boolean connected = client.isConnected();

NetworkContext ctx = client.context();
InetSocketAddress addr = ctx.getRemoteAddress();
```

---

## Отключение / Disconnecting

```java
client.disconnect();

// или try-with-resources / or try-with-resources
try (NetworkClient client = Net.client().connect("localhost", 25565).get()) {
    client.send(new LoginPacket("player1", "tok"));
    Thread.sleep(1000);
}
```

---

## Обработка входящих пакетов / Handling Inbound Packets

Клиент обрабатывает входящие пакеты теми же `@PacketHandler`-методами что и сервер:

```java
@NetworkListener
public class ClientHandler {

    @PacketHandler
    public void onWelcome(WelcomePacket pkt, NetworkContext ctx) {
        System.out.println("Server says: " + pkt.message());
    }
}

NetworkClient client = Net.client()
        .listener(new ClientHandler())   // ← добавить обработчик
        .connect("localhost", 25565)
        .get();
```
