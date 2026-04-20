# Конвейер перехватчиков / Interceptor Pipeline

`PacketPipeline` — упорядоченная цепочка `PacketInterceptor`, через которую проходит каждый пакет  
при чтении (onRead) и записи (onWrite).  
> `PacketPipeline` is an ordered chain of `PacketInterceptor`s. Every packet passes through on read and write.

---

## Интерфейс / Interface

```java
public interface PacketInterceptor<T extends Packet> {
    Class<T> packetType();    // какие пакеты перехватывать / which packets to intercept
    void onRead (T pkt, NetworkContext ctx, InterceptorChain chain);
    void onWrite(T pkt, NetworkContext ctx, InterceptorChain chain);
}
```

`chain.proceed()` — передать пакет следующему перехватчику.  
Без вызова `proceed()` пакет не идёт дальше.  
> `chain.proceed()` — pass to next interceptor. Without it, the packet stops here.

---

## Добавление перехватчиков / Adding Interceptors

### Через Builder (рекомендуется) / Via Builder (recommended)

```java
Net.server()
    .interceptor(new LoggingInterceptor())
    .interceptor(new RateLimitInterceptor(100, 200))
    .start();
```

### Через Pipeline вручную / Via Pipeline manually

```java
NetworkContext ctx = ...;
PacketPipeline pipeline = ctx.getPipeline();

pipeline.addFirst("logging",   new LoggingInterceptor());
pipeline.addLast ("ratelimit", new RateLimitInterceptor(50, 100));
pipeline.addBefore("ratelimit", "auth", new AuthInterceptor());
pipeline.remove("logging");
```

---

## Встроенные перехватчики / Built-in Interceptors

### LoggingInterceptor

Логирует каждый пакет через SLF4J.

```java
new LoggingInterceptor()               // уровень DEBUG / DEBUG level
new LoggingInterceptor(Level.INFO)     // явно указать уровень / explicit level
```

Вывод: `[READ ] LoginPacket from /127.0.0.1:54321`

---

### RateLimitInterceptor

Token-bucket лимитер на соединение. Состояние хранится в `AttributeKey` контекста.  
> Per-connection token-bucket rate limiter. State stored in connection `AttributeKey`.

```java
new RateLimitInterceptor(
    100.0,   // токенов/секунду / tokens per second
    200      // ёмкость всплеска / burst capacity
)
```

При переполнении: `PipelineException("Rate limit exceeded for /ip:port")`

---

### CompressionInterceptor ⚠️ не PacketInterceptor / NOT a PacketInterceptor

`CompressionInterceptor` **не реализует** `PacketInterceptor`. Добавление через `.interceptor()` **не даёт** сжатие.  
Это утилитный класс для ручного сжатия байт.  
> `CompressionInterceptor` does **not** implement `PacketInterceptor`. Adding it via `.interceptor()` does nothing.  
> It is a byte-level utility class only.

**Для автосжатия** используй `.compress(threshold)` в Builder:
```java
Net.server().compress(256).start();    // ← правильно / correct
Net.server().interceptor(new CompressionInterceptor(256)).start();  // ← ничего не делает / no-op
```

Ручное использование / Manual use:
```java
CompressionInterceptor util = new CompressionInterceptor(256);
byte[] compressed = util.compress(rawBytes);
byte[] original   = util.decompress(compressed, 0);
```

→ Полная документация: [Compression.md](Compression.md)

---

### EncryptionInterceptor ⚠️ не PacketInterceptor / NOT a PacketInterceptor

Аналогично: **не реализует** `PacketInterceptor`. Прозрачное шифрование — только `.encrypt(key)` в Builder.  
> Same: does **not** implement `PacketInterceptor`. Transparent encryption via `.encrypt(key)` in Builder only.

```java
Net.server().encrypt(secretKey).start();  // ← правильно / correct

// Ручное использование / Manual use:
EncryptionInterceptor util = new EncryptionInterceptor(key);
byte[] encrypted  = util.encrypt(plaintext);
byte[] decrypted  = util.decrypt(encrypted);
```

→ Полная документация: [Encryption.md](Encryption.md)

---

## Порядок выполнения / Execution Order

Перехватчики выполняются в порядке добавления для `onRead` и `onWrite`.  
`packetType()` фильтрует: перехватчик `Packet.class` получает ВСЕ пакеты;  
перехватчик `LoginPacket.class` — только `LoginPacket`.  
> Interceptors execute in insertion order. `packetType()` filters by exact type or supertype.

```
onRead:  interceptor[0] → interceptor[1] → ... → handler
onWrite: interceptor[0] → interceptor[1] → ... → wire
```

---

## Свой перехватчик / Custom Interceptor

```java
@Interceptor(order = 20)
public final class AuthInterceptor implements PacketInterceptor<LoginPacket> {

    @Override
    public Class<LoginPacket> packetType() { return LoginPacket.class; }

    @Override
    public void onRead(LoginPacket pkt, NetworkContext ctx, InterceptorChain chain) {
        if (!validate(pkt.token())) {
            ctx.disconnect("Invalid token");
            return;          // не вызываем proceed() — пакет остановлен
        }
        chain.proceed();
    }

    @Override
    public void onWrite(LoginPacket pkt, NetworkContext ctx, InterceptorChain chain) {
        chain.proceed();     // запись не проверяем
    }
}
```
