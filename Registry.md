# Реестр пакетов / Packet Registry

`PacketRegistry` хранит маппинг `packetId ↔ PacketCodec` и `packetType ↔ packetId`.  
Thread-safe для чтения после фазы сборки.  
> `PacketRegistry` stores `packetId ↔ PacketCodec` and `packetType ↔ packetId` mappings.  
> Thread-safe for reads after the build phase.

---

## Создание / Creation

```java
PacketRegistry registry = PacketRegistry.create(); // возвращает DefaultPacketRegistry
// или через фасад / or via facade
PacketRegistry registry = Net.registry();
```

---

## Ручная регистрация / Manual Registration

```java
DefaultPacketRegistry registry = (DefaultPacketRegistry) Net.registry();

registry.register(1, LoginPacket.class,   new LoginCodec());
registry.register(2, WelcomePacket.class, new WelcomeCodec());
registry.register(3, ChatPacket.class,    new ChatCodec());
```

Повторная регистрация одного ID бросает `NetworkException`.  
> Registering a duplicate ID throws `NetworkException`.

---

## Автосканирование / Auto-Scan

Если пакеты аннотированы `@PacketId` + `@Codec`, можно сканировать весь пакет:  
> If packets have `@PacketId` + `@Codec`, scan the whole package:

```java
registry.scanPackage("com.mygame.packet");
```

`scanPackage` находит все `.class`-файлы в пакете, проверяет обе аннотации,  
инстанциирует кодек через `getDeclaredConstructor().newInstance()`, регистрирует.

**Требование:** кодек должен иметь публичный конструктор без аргументов.  
> **Requirement:** codec must have a public no-args constructor.

---

## Передача в сервер / Passing to Server

```java
NetworkServer server = Net.server()
        .codec(registry)   // ← передать реестр / pass registry
        .port(25565)
        .start();
```

Если `.codec()` не вызван, создаётся пустой реестр — все входящие пакеты вызовут `PacketDecodeException`.  
> If `.codec()` is omitted, an empty registry is used — all inbound packets throw `PacketDecodeException`.

---

## Lookup во время выполнения / Runtime Lookup

```java
Optional<PacketCodec<?>> codec = registry.lookup(packetId);
int id = registry.idFor(LoginPacket.class);
```
