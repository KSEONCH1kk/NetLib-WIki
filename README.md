# NetLib — Документация / Documentation
https://sobaka.intave.tech:3000/bus/f7fdcb9c-7e43-48a5-bc94-235a94e769be
**NetLib** (`ru.kseonyt.net`) — аннотационно-управляемая сетевая библиотека для Java 17, построенная на Netty 4.1.  
Вдохновлена Proton / Unity Networking: ноль шаблонного кода, полный ООП, строгий SOLID.

> **[EN]** `ru.kseonyt.net` is an annotation-driven, zero-boilerplate Java 17 networking library built on Netty 4.1.  
> Inspired by Proton / Unity Networking: full OOP, strict SOLID.

---

## Навигация / Navigation

| Страница | Page | Описание |
|----------|------|----------|
| [Быстрый старт](Quick-Start.md) | Quick Start | Сервер + клиент за 5 минут |
| [Архитектура](Architecture.md) | Architecture | Общая структура, SOLID-решения |
| [Пакеты и кодеки](Packets.md) | Packets & Codecs | `@PacketId`, `@Codec`, `PacketBuffer` |
| [Реестр пакетов](Registry.md) | Packet Registry | `PacketRegistry`, авто-сканирование |
| [Конвейер](Pipeline.md) | Pipeline | `PacketInterceptor`, встроенные перехватчики |
| [Шифрование](Encryption.md) | Encryption | AES-128-GCM, прозрачное шифрование |
| [Сжатие](Compression.md) | Compression | zlib, прозрачное сжатие, `.compress(threshold)` |
| [UDP](UDP.md) | UDP | `UdpEndpoint`, `ConnectedUdpClient`, отличия от TCP |
| [Сервер](Server.md) | Server | `NetworkServer`, `NetworkServerBuilder` |
| [Клиент](Client.md) | Client | `NetworkClient`, переподключение |
| [Обработчики](Handlers.md) | Handlers | `@PacketHandler`, `@NetworkListener` |
| [Контекст соединения](Context.md) | Connection Context | `NetworkContext`, `AttributeKey` |
| [Формат пакетов](Wire-Format.md) | Wire Format | Бинарный протокол |
| [Исключения](Exceptions.md) | Exceptions | Иерархия ошибок |
| [API-справочник](API-Reference.md) | API Reference | Полный перечень публичных методов |

---

## Быстрый пример / Quick Example

```java
// Определить пакет / Define a packet
@PacketId(1)
@Codec(LoginCodec.class)
public record LoginPacket(String username, String token) implements Packet {}

// Обработать его / Handle it
@NetworkListener
public class GameHandler {
    @PacketHandler
    public void onLogin(LoginPacket pkt, NetworkContext ctx) {
        ctx.send(new WelcomePacket("hi " + pkt.username()));
    }
}

// Запустить сервер / Start server
NetworkServer server = Net.server().port(25565).start();

// Подключить клиент / Connect client
NetworkClient client = Net.client().connect("localhost", 25565).get();
client.send(new LoginPacket("player1", "tok-xyz"));
```

---

## Требования / Requirements

| | |
|--|--|
| Java | 17+ |
| Netty | 4.1.108+ |
| SLF4J | 2.0+ |
| org.jetbrains:annotations | 24+ (compile-only) |
