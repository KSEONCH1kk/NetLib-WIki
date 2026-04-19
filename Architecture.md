# Архитектура / Architecture

## Обзор / Overview

NetLib разделён на 8 независимых слоёв. Каждый слой зависит только от нижележащих.  
> NetLib is split into 8 independent layers. Each layer depends only on layers below it.

```
┌─────────────────────────────────────────────┐
│            Net  (статический фасад)          │  ← точка входа
├────────────────────┬────────────────────────┤
│  NetworkServer     │  NetworkClient          │  ← публичный API
│  NetworkServerBuilder  NetworkClientBuilder  │
├────────────────────┴────────────────────────┤
│       handler  (PacketHandlerRegistry        │  ← диспетчеризация
│                 PacketDispatcher)            │
├─────────────────────────────────────────────┤
│       pipeline  (PacketPipeline              │  ← перехватчики
│                  built-in interceptors)      │
├─────────────────────────────────────────────┤
│       context  (NetworkContext               │  ← состояние соединения
│                 AttributeKey / Attribute)    │
├─────────────────────────────────────────────┤
│       packet  (Packet / PacketBuffer         │  ← сериализация
│                PacketCodec / PacketRegistry) │
├─────────────────────────────────────────────┤
│       annotation  (5 аннотаций)              │  ← метаданные
├─────────────────────────────────────────────┤
│       exception  (иерархия RuntimeException) │  ← ошибки
└─────────────────────────────────────────────┘
         ↓ транспорт / transport
┌─────────────────────────────────────────────┐
│       Netty 4.1  (NIO EventLoopGroup)        │
└─────────────────────────────────────────────┘
```

---

## SOLID-решения / SOLID Decisions

### S — Единственная ответственность / Single Responsibility

| Класс | Одна задача |
|-------|-------------|
| `PacketBuffer` | Только чтение/запись байт |
| `PacketRegistry` | Только маппинг ID ↔ кодек |
| `PacketDispatcher` | Только поиск и вызов обработчика |
| `NettyNetworkServer` | Только управление Netty-сервером |
| `DefaultPacketPipeline` | Только упорядоченный список перехватчиков |

### O — Открытость/Закрытость / Open/Closed

Новые возможности добавляются через `PacketInterceptor` или новые `PacketCodec`.  
Ядро (`NettyNetworkServer`, `DefaultPacketPipeline`) не меняется.  
> New capabilities are added via `PacketInterceptor` or new `PacketCodec`. Core classes never modified.

### L — Подстановка Лисков / Liskov Substitution

`NettyNetworkServer` и `NettyNetworkClient` полностью заменяемы через `NetworkServer` и `NetworkClient`.  
> Full substitutability: any `NetworkServer` impl can replace `NettyNetworkServer`.

### I — Разделение интерфейсов / Interface Segregation

```
PacketCodec<T>        — только encode/decode
PacketInterceptor<T>  — только onRead/onWrite  
NetworkContext        — только send/disconnect/attr
```
Клиент никогда не видит Netty-internal API.

### D — Инверсия зависимостей / Dependency Inversion

`NettyNetworkServer` принимает `PacketRegistry` и `PacketDispatcher` через конструктор.  
`PacketDispatcher` принимает `PacketHandlerRegistry` через конструктор.  
Нет `new` внутри бизнес-логики. Все зависимости явные.  
> No hidden `new`. All dependencies injected via constructors.

---

## Поток данных / Data Flow

### Входящий пакет / Inbound Packet

```
TCP-байты
  → [EncryptionDecoder]  (если включено / if enabled)
  → LengthFieldBasedFrameDecoder  (4-byte framing)
  → PacketDecoder (читает id, вызывает codec.decode())
  → PacketPipeline.fireRead()  (перехватчики / interceptors)
  → PacketDispatcher.dispatch()
  → @PacketHandler-метод
```

### Исходящий пакет / Outbound Packet

```
ctx.send(packet)
  → PacketPipeline.fireWrite()  (перехватчики / interceptors)
  → codec.encode()  →  ByteBuf [4B len][4B id][payload]
  → [EncryptionEncoder]  (если включено / if enabled)
  → TCP-байты
```

---

## Потокобезопасность / Thread Safety

| Компонент | Гарантия |
|-----------|----------|
| `DefaultPacketRegistry` | `ConcurrentHashMap`, read-safe after build |
| `PacketHandlerRegistry` | `ConcurrentHashMap` + synchronized list, read-safe after build |
| `DefaultPacketPipeline` | `ReentrantReadWriteLock` — конкурентные чтения / concurrent reads |
| `Attribute<T>` | `AtomicReference` |
| `NettyNetworkContext.send()` | Netty channel thread-safe |
| `RateLimitInterceptor` | `AtomicLong` token bucket |
