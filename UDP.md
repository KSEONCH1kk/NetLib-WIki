# UDP / UDP Support

NetLib поддерживает UDP через `NioDatagramChannel` Netty.  
UDP connectionless по природе — один класс (`UdpEndpoint`) обслуживает и серверную, и клиентскую роль.  
> NetLib supports UDP via Netty's `NioDatagramChannel`. One class handles both server and client roles.

---

## TCP vs UDP — когда что использовать / When to Use Which

| | TCP | UDP |
|-|-----|-----|
| Доставка | Гарантирована | Нет гарантий |
| Порядок | Сохраняется | Не гарантирован |
| Latency | Выше (ACK, retransmit) | Ниже |
| Применение | Логин, чат, инвентарь | Позиции игроков, стрельба, ping |
| Reconnect | ✅ | ❌ (нет соединения) |
| Шифрование (встроенное) | ✅ `.encrypt(key)` | ❌ (вручную) |

---

## Серверная сторона / Server Side

```java
PacketRegistry registry = Net.registry();
// регистрируем пакеты...

UdpEndpoint server = Net.udp()
        .host("0.0.0.0")     // по умолчанию / default
        .port(25565)
        .codec(registry)
        .listener(new GameHandler())
        .bind();
```

`server.send(packet, address)` — отправить пакет конкретному адресу.  
`server.broadcast(packet, addresses)` — разослать список адресов.

```java
server.send(new WorldStatePacket(state), playerAddr);
server.broadcast(new ChatPacket("hello"), allPlayers);

InetSocketAddress local = server.localAddress();  // /0.0.0.0:25565
boolean open = server.isOpen();

server.shutdown();  // или try-with-resources
```

---

## Клиентская сторона / Client Side

```java
ConnectedUdpClient client = Net.udp()
        .codec(registry)
        .listener(new ClientHandler())
        .connect("localhost", 25565);  // эфемерный локальный порт / ephemeral local port

// send() без адреса — адрес зафиксирован при .connect()
client.send(new PositionPacket(x, y, z));
client.sendAsync(new PingPacket()).thenRun(() -> System.out.println("sent"));

client.serverAddress();  // → /127.0.0.1:25565
client.isOpen();
client.disconnect();
```

`ConnectedUdpClient` — тонкая обёртка над `UdpEndpoint` с запомненным `InetSocketAddress`.  
> `ConnectedUdpClient` is a thin wrapper over `UdpEndpoint` with a stored `InetSocketAddress`.

---

## Обработчики / Handlers

Те же `@PacketHandler` методы работают для UDP и TCP.  
Тип контекста — `NetworkContext` (UDP-реализация):  
- `ctx.getRemoteAddress()` → адрес отправителя датаграммы
- `ctx.send(packet)` → ответ отправителю датаграммы
- `ctx.disconnect(reason)` → no-op (UDP connectionless)
- `ctx.attr(key)` → состояние по адресу (между датаграммами от одного отправителя)

```java
@NetworkListener
public class GameHandler {

    @PacketHandler
    public void onPosition(PositionPacket pkt, NetworkContext ctx) {
        // ctx.getRemoteAddress() = адрес клиента / client's address
        broadcastToOthers(pkt, ctx.getRemoteAddress());
    }
}
```

---

## Формат датаграммы / Datagram Wire Format

```
[4-byte packet-id][N bytes payload]
```

Нет length-prefix — UDP сам гарантирует границы датаграммы.  
> No length prefix — UDP guarantees datagram boundaries natively.

---

## Состояние на соединение / Per-Address State

UDP не имеет соединений, но `AttributeKey` работает per-sender-address:

```java
static final AttributeKey<PlayerState> STATE_KEY = AttributeKey.valueOf("udp-player-state");

@PacketHandler
public void onLogin(LoginPacket pkt, NetworkContext ctx) {
    ctx.attr(STATE_KEY).set(new PlayerState(pkt.username()));
}

@PacketHandler
public void onMove(MovePacket pkt, NetworkContext ctx) {
    PlayerState state = ctx.attr(STATE_KEY).get();
    if (state == null) { ctx.send(new ErrorPacket("Not logged in")); return; }
    state.update(pkt);
}
```

Состояние хранится в `ConcurrentHashMap<InetSocketAddress, ...>` внутри `NettyUdpEndpoint`.  
Утечка памяти возможна если адреса накапливаются — очищай при необходимости через `UdpEndpoint.send()` и логику приложения.

---

## Ограничения / Limitations

| Ограничение | Причина |
|-------------|---------|
| Нет встроенного шифрования | Нет channel pipeline как у TCP |
| Нет reconnect | Connectionless |
| Нет compression channel handler | Нет TCP-стрима |
| Max datagram size | ~65507 байт (IP limit) |
| Нет ordering / reliability | UDP-природа; реализуй RUDP в приложении при необходимости |

Для шифрования UDP используй `EncryptionInterceptor.encrypt()`/`decrypt()` вручную перед отправкой и после получения.
