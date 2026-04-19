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

### CompressionInterceptor

Zlib deflate/inflate. Пакеты меньше `threshold` байт не сжимаются.  
> Zlib deflate/inflate. Packets smaller than `threshold` bytes are not compressed.

```java
new CompressionInterceptor(256)  // сжимать от 256 байт / compress if payload >= 256 bytes
```

> **Примечание:** Перехватчик работает на уровне объектов. Для сжатия на уровне байт нужен отдельный `ChannelHandler`.  
> **Note:** This interceptor operates at the packet-object level. Byte-level compression requires a separate `ChannelHandler`.

---

### EncryptionInterceptor

AES-128-GCM на уровне конвейера. Для прозрачного шифрования всего соединения используй `.encrypt(key)` в Builder — см. [Шифрование](Encryption.md).

```java
new EncryptionInterceptor(secretKey)
// encrypt(byte[]) / decrypt(byte[]) — публичные методы для ручного использования
```

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
