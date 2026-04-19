# Контекст соединения / Connection Context

`NetworkContext` — объект, привязанный к одному TCP-соединению. Thread-safe.  
> `NetworkContext` — object bound to one TCP connection. Thread-safe.

---

## Основные методы / Core Methods

```java
// Отправить пакет / Send packet
ctx.send(new WelcomePacket("Hello"));

// Отправить raw-байты / Send raw bytes
ctx.sendRaw(new byte[]{0x01, 0x02, 0x03});

// Разорвать соединение / Disconnect
ctx.disconnect("Banned: toxic behavior");

// Адрес клиента / Client address
InetSocketAddress addr = ctx.getRemoteAddress();  // /192.168.1.5:54321

// Активно ли соединение / Is connection active
boolean alive = ctx.isActive();

// Доступ к pipeline / Access pipeline
PacketPipeline pipeline = ctx.getPipeline();
```

---

## Атрибуты (пользовательские данные) / Attributes (Custom Per-Connection Data)

`AttributeKey<T>` — типизированный ключ. `Attribute<T>` — атомарное хранилище.  
> `AttributeKey<T>` — typed key. `Attribute<T>` — atomic storage.

### Хранить данные игрока / Store Player Data

```java
// Определить ключ (обычно static final) / Define key (usually static final)
static final AttributeKey<Player> PLAYER_KEY = AttributeKey.valueOf("player");

// Записать / Write
ctx.attr(PLAYER_KEY).set(new Player("player1", 42));

// Прочитать / Read
Player player = ctx.attr(PLAYER_KEY).get();

// CAS (compare-and-set) для атомарного обновления / atomic update
ctx.attr(PLAYER_KEY).compareAndSet(expected, newValue);
```

### Типичный паттерн: аутентификация / Typical Pattern: Authentication

```java
static final AttributeKey<Boolean> AUTH_KEY = AttributeKey.valueOf("authenticated");

@PacketHandler
public void onLogin(LoginPacket pkt, NetworkContext ctx) {
    if (validate(pkt.token())) {
        ctx.attr(AUTH_KEY).set(true);
        ctx.send(new WelcomePacket("Welcome, " + pkt.username()));
    } else {
        ctx.disconnect("Invalid token");
    }
}

@PacketHandler
public void onAction(ActionPacket pkt, NetworkContext ctx) {
    Boolean auth = ctx.attr(AUTH_KEY).get();
    if (auth == null || !auth) {
        ctx.disconnect("Not authenticated");
        return;
    }
    // обрабатываем / process
}
```

---

## AttributeKey — пул ключей / AttributeKey Key Pool

`AttributeKey.valueOf(name)` возвращает singleton по имени — безопасно вызывать несколько раз.  
> `AttributeKey.valueOf(name)` returns a singleton by name — safe to call multiple times.

```java
AttributeKey<Integer> k1 = AttributeKey.valueOf("score");
AttributeKey<Integer> k2 = AttributeKey.valueOf("score");
assert k1 == k2;  // один объект / same instance
```

---

## Pipeline из контекста / Pipeline from Context

```java
NetworkContext ctx = ...;
PacketPipeline pipeline = ctx.getPipeline();

// Добавить перехватчик на лету / Add interceptor at runtime
pipeline.addFirst("throttle", new RateLimitInterceptor(10, 20));
```

Изменения pipeline одного соединения не влияют на другие соединения.  
> Pipeline changes for one connection do not affect other connections.
