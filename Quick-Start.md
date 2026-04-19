# Быстрый старт / Quick Start

## 1. Зависимость Maven / Maven Dependency

```xml
<dependency>
    <groupId>ru.kseonyt</groupId>
    <artifactId>netlib</artifactId>
    <version>1.0.0</version>
</dependency>
```

---

## 2. Определить пакеты / Define Packets

Каждый пакет — это `record`, реализующий `Packet`. Две аннотации: ID и кодек.  
> Each packet is a `record` implementing `Packet`. Two annotations: ID and codec.

```java
// LoginPacket.java
@PacketId(1)
@Codec(LoginCodec.class)
public record LoginPacket(String username, String token) implements Packet {}

// WelcomePacket.java
@PacketId(2)
@Codec(WelcomeCodec.class)
public record WelcomePacket(String message) implements Packet {}
```

---

## 3. Написать кодеки / Write Codecs

```java
public final class LoginCodec implements PacketCodec<LoginPacket> {

    @Override
    public void encode(LoginPacket pkt, PacketBuffer buf) {
        buf.writeString(pkt.username());
        buf.writeString(pkt.token());
    }

    @Override
    public LoginPacket decode(PacketBuffer buf) {
        return new LoginPacket(buf.readString(), buf.readString());
    }
}
```

---

## 4. Обработчик пакетов / Packet Handler

```java
@NetworkListener
public class GameHandler {

    @PacketHandler
    public void onLogin(LoginPacket pkt, NetworkContext ctx) {
        System.out.println("Login from " + pkt.username());
        ctx.send(new WelcomePacket("Welcome, " + pkt.username()));
    }

    @PacketHandler
    public void onWelcome(WelcomePacket pkt, NetworkContext ctx) {
        System.out.println("Server says: " + pkt.message());
    }
}
```

---

## 5. Запуск сервера / Start Server

```java
PacketRegistry registry = Net.registry();
// регистрируем вручную или через scanPackage()
((DefaultPacketRegistry) registry).register(1, LoginPacket.class, new LoginCodec());
((DefaultPacketRegistry) registry).register(2, WelcomePacket.class, new WelcomeCodec());

NetworkServer server = Net.server()
        .port(25565)
        .codec(registry)
        .listener(new GameHandler())
        .start();

server.onConnect(ctx -> System.out.println("Connected: " + ctx.getRemoteAddress()));
```

---

## 6. Подключение клиента / Connect Client

```java
NetworkClient client = Net.client()
        .codec(registry)
        .listener(new GameHandler())
        .connect("localhost", 25565)
        .get();                           // блокирует до подключения / blocks until connected

client.send(new LoginPacket("player1", "secret-token"));
```

---

## 7. С шифрованием / With Encryption

```java
SecretKey key = KeyGenerator.getInstance("AES").generateKey(); // 128-bit

NetworkServer server = Net.server()
        .port(25565)
        .codec(registry)
        .encrypt(key)                     // AES-128-GCM прозрачно / transparently
        .listener(new GameHandler())
        .start();

NetworkClient client = Net.client()
        .codec(registry)
        .encrypt(key)                     // тот же ключ / same key
        .connect("localhost", 25565)
        .get();
```

> [EN] Both sides must use the same `SecretKey`. The library handles IV generation, encryption, and decryption transparently — application code never changes.

---

## 8. Остановка / Shutdown

```java
server.shutdown();   // или try-with-resources / or try-with-resources
client.disconnect();
```
