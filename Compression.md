# Сжатие / Compression

NetLib поддерживает **прозрачное** zlib deflate/inflate на уровне Netty-канала.  
Работает только для TCP. UDP не поддерживается (нет стримового channel pipeline).  
> Transparent zlib deflate/inflate at Netty channel level. TCP only.

---

## Как включить / How to Enable

```java
// Сервер / Server
NetworkServer server = Net.server()
        .port(25565)
        .codec(registry)
        .compress(256)         // ← сжимать фреймы >= 256 байт / compress frames >= 256 bytes
        .listener(new GameHandler())
        .start();

// Клиент / Client
NetworkClient client = Net.client()
        .codec(registry)
        .compress(256)         // ← должен совпадать с сервером / must match server
        .connect("localhost", 25565)
        .get();
```

**Оба конца обязаны иметь одинаковый порог** (или оба включены, оба выключены).  
Сервер с `compress(256)` + клиент без `.compress()` → клиент получит мусор.  
> Both ends must have compression enabled with the same threshold, or both disabled.

---

## Алгоритм / Algorithm

| Параметр | Значение |
|----------|----------|
| Алгоритм | zlib deflate (raw, nowrap=true) |
| Уровень | `Deflater.DEFAULT_COMPRESSION` |
| Порог | задаётся пользователем (bytes) |

Фреймы **меньше** порога передаются без сжатия (flag=0).  
Фреймы **>=** порога сжимаются (flag=1).

---

## Формат фрейма / Frame Format

```
[4B outer-length][1B flag: 0=raw, 1=compressed][data]
                                                 └── compressed or raw: [4B inner-len][4B id][payload]
```

- `outer-length` = `1 + len(data)` (flag + data, не включает само поле длины)
- При flag=0: `data` = оригинальный inner frame, без изменений
- При flag=1: `data` = zlib-compressed inner frame

---

## Позиция в Netty-пайплайне / Position in Netty Pipeline

```
Входящие (inbound) ↓
  [EncryptionDecoder]          ← если .encrypt() включён
  [CompressionDecoder]         ← читает outer frame, разжимает если flag=1
  [LengthFieldBasedFrameDecoder]
  [PacketDecoder]

Исходящие (outbound) ↑
  CompressionEncoder           ← сжимает если payload >= threshold, добавляет outer frame
  [EncryptionEncoder]          ← если .encrypt() включён
```

При одновременном использовании encrypt + compress: **compress first, then encrypt** на запись; **decrypt first, then decompress** на чтение. Это оптимально — сжатый текст хуже сжимается повторно и шифровать сжатое эффективнее.

---

## CompressionInterceptor — утилитный класс / Utility Class

`CompressionInterceptor` — **не** `PacketInterceptor`. Добавление в `.interceptor()` **не даёт** автосжатие.  
Это утилита для ручного использования:

```java
CompressionInterceptor util = new CompressionInterceptor(256);

byte[] compressed   = util.compress(rawBytes);           // сжать
byte[] decompressed = util.decompress(compressed, 0);    // разжать (0 = неизвестный размер)
int    threshold    = util.threshold();                  // → 256
```

Для автосжатия — только `.compress(threshold)` в Builder.

---

## Совместно с шифрованием / Combined with Encryption

```java
NetworkServer server = Net.server()
        .port(25565)
        .codec(registry)
        .compress(256)     // compress first
        .encrypt(key)      // then encrypt
        .start();

NetworkClient client = Net.client()
        .codec(registry)
        .compress(256)
        .encrypt(key)
        .connect("localhost", 25565)
        .get();
```

Порядок вызовов в builder не важен — Netty handlers расставляются в правильном порядке автоматически.  
> Order of builder calls doesn't matter — handler insertion order is fixed internally.
