# Обработчики пакетов / Packet Handlers

## Аннотации / Annotations

### @NetworkListener

Помечает класс как контейнер обработчиков. Обязателен для авто-сканирования.  
> Marks a class as a handler container. Required for auto-scan.

```java
@NetworkListener
public class GameHandler { ... }
```

### @PacketHandler

Помечает метод как обработчик конкретного типа пакета. Тип определяется по первому параметру.  
> Marks a method as a handler for a specific packet type. Type is determined by the first parameter.

```java
@PacketHandler
public void onLogin(LoginPacket pkt, NetworkContext ctx) { ... }

// без контекста тоже работает / without context also works
@PacketHandler
public void onChat(ChatPacket pkt) { ... }

// с приоритетом / with priority (higher = first)
@PacketHandler(priority = 10)
public void onLoginHighPriority(LoginPacket pkt, NetworkContext ctx) { ... }
```

---

## Сигнатуры методов / Method Signatures

| Параметры | Поддерживается |
|-----------|----------------|
| `(PacketType pkt, NetworkContext ctx)` | ✅ |
| `(PacketType pkt)` | ✅ |
| `(NetworkContext ctx, PacketType pkt)` | ❌ первый параметр — пакет |

Метод может быть `public`, `protected` или package-private. `private` не поддерживается.  
> Method can be `public`, `protected`, or package-private. `private` is not supported.

---

## Несколько обработчиков одного типа / Multiple Handlers for Same Type

```java
@NetworkListener
public class GameHandler {

    @PacketHandler(priority = 10)   // выполняется первым / executes first
    public void onLoginAuth(LoginPacket pkt, NetworkContext ctx) {
        // валидация токена / validate token
    }

    @PacketHandler(priority = 0)    // выполняется вторым / executes second
    public void onLoginSession(LoginPacket pkt, NetworkContext ctx) {
        // создать сессию / create session
    }
}
```

Все обработчики одного типа выполняются в порядке убывания приоритета.  
> All handlers for the same type execute in descending priority order.

---

## Перехват суперкласса / Supertype Handlers

Обработчик на `Packet.class` получает ВСЕ пакеты (если нет более специфичного обработчика):  
> A handler on `Packet.class` receives ALL packets (if no more specific handler exists):

```java
@PacketHandler
public void onAny(Packet pkt, NetworkContext ctx) {
    log.info("Received: {}", pkt.getClass().getSimpleName());
}
```

---

## Регистрация / Registration

### Через Builder (рекомендуется) / Via Builder (recommended)

```java
Net.server()
    .listener(new GameHandler())
    .listener(new AdminHandler())
    .start();
```

### Вручную / Manually

```java
PacketHandlerRegistry registry = new PacketHandlerRegistry();
registry.register(new GameHandler());

// авто-сканирование пакета / auto-scan package
registry.registerAll("com.mygame.handler");
```

---

## Исключения в обработчиках / Exceptions in Handlers

Исключение в `@PacketHandler` логируется через SLF4J и не останавливает обработку других обработчиков.  
Соединение не разрывается автоматически — обработай ошибку вручную.  
> Exceptions in `@PacketHandler` are logged via SLF4J. Other handlers continue executing.  
> The connection is NOT automatically closed — handle errors manually.

```java
@PacketHandler
public void onLogin(LoginPacket pkt, NetworkContext ctx) {
    try {
        process(pkt);
    } catch (Exception e) {
        ctx.disconnect("Internal error");
        throw e;  // залогируется / will be logged
    }
}
```
