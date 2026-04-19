# Исключения / Exceptions

Все исключения NetLib — `RuntimeException`. Не нужно объявлять в `throws`.  
> All NetLib exceptions are `RuntimeException`. No need to declare in `throws`.

---

## Иерархия / Hierarchy

```
RuntimeException
└── NetworkException                 ← базовый / base
    ├── PacketEncodeException        ← ошибка сериализации
    ├── PacketDecodeException        ← ошибка десериализации
    ├── HandlerNotFoundException     ← нет обработчика (не бросается по умолчанию)
    ├── PipelineException            ← ошибка в перехватчике
    └── ConnectionException          ← ошибка соединения / bind / connect
```

---

## Когда бросается / When Thrown

| Исключение | Причина |
|-----------|---------|
| `PacketEncodeException` | `codec.encode()` бросил исключение; нет кодека для типа пакета |
| `PacketDecodeException` | `codec.decode()` бросил исключение; неизвестный packet-id |
| `HandlerNotFoundException` | (опционально) нет `@PacketHandler` для типа пакета |
| `PipelineException` | Rate limit exceeded; ошибка в `CompressionInterceptor`; ошибка `EncryptionInterceptor` |
| `ConnectionException` | Порт занят при старте сервера; таймаут подключения клиента |
| `NetworkException` | Дублирующийся `@PacketId` при регистрации; нет ID для типа пакета |

---

## Обработка / Handling

```java
try {
    NetworkClient client = Net.client()
            .connect("localhost", 25565)
            .get();
} catch (ConnectionException e) {
    log.error("Cannot connect: {}", e.getMessage());
}

@PacketHandler
public void onLogin(LoginPacket pkt, NetworkContext ctx) {
    try {
        ctx.send(new ResponsePacket(...));
    } catch (PacketEncodeException e) {
        log.error("Failed to encode response", e);
        ctx.disconnect("Server error");
    }
}
```

---

## Конструкторы / Constructors

Все исключения поддерживают два конструктора:

```java
new PacketDecodeException("Unknown packet id: 42");
new PacketDecodeException("Codec error", cause);
```
