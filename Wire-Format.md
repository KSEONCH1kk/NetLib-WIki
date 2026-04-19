# Формат пакетов / Wire Format

## Без шифрования / Without Encryption

```
┌──────────────────────────────────────────────────────────┐
│  4 bytes     │  4 bytes    │  N bytes                    │
│  total-len   │  packet-id  │  payload (PacketBuffer)     │
│  (big-endian)│  (big-endian)│                            │
└──────────────────────────────────────────────────────────┘
```

- `total-len` = `4 + N` (включает поле ID, не включает сам total-len)  
- `packet-id` = значение `@PacketId` (int, big-endian)  
- `payload` = байты из `PacketCodec.encode()`

> `total-len` = `4 + N` (includes the ID field, does NOT include the length field itself).

**Пример / Example** — LoginPacket("alice", "tok123"):

```
| 00 00 00 11 | 00 00 00 01 | 05 61 6C 69 63 65  07 74 6F 6B 31 32 33 |
  total=17      id=1          "alice" (len=5)     "tok123" (len=7)
```

---

## С шифрованием / With Encryption

```
┌────────────────────────────────────────────────────────────────┐
│  4 bytes      │  12 bytes   │  M bytes                        │
│  outer-len    │  random IV  │  AES-GCM ciphertext + 16B tag   │
│               │             │  (decrypts to inner frame above) │
└────────────────────────────────────────────────────────────────┘
```

- `outer-len` = `12 + M` (IV + ciphertext)
- IV — случайный `SecureRandom`, новый для каждого пакета
- Зашифровано: весь внутренний фрейм `[4B len][4B id][payload]`
- GCM-тег (16 байт) добавлен к ciphertext автоматически JCA

> IV is random per-packet. The entire inner frame is encrypted. GCM tag is 16 bytes appended by JCA.

---

## Framing в Netty / Netty Framing

### Без шифрования / Without encryption

```java
new LengthFieldBasedFrameDecoder(
    65535,  // maxFrameLength
    0,      // lengthFieldOffset
    4,      // lengthFieldLength
    -4,     // lengthAdjustment (total-len includes itself: -4 to get payload-only size)
    4       // initialBytesToStrip (skip the 4-byte length field)
)
```

После этого декодера в pipeline приходит: `[4B packet-id][N bytes payload]`

### С шифрованием / With encryption

```
wire → EncryptionDecoder → [4B inner-len][4B id][payload]
     → LengthFieldBasedFrameDecoder → [4B id][payload]
     → PacketDecoder
```

`EncryptionDecoder` читает `[4B outer-len]`, затем `outer-len` байт, дешифрует, выдаёт plain ByteBuf.

---

## VarInt в payload / VarInt in Payload

VarInt используется внутри `PacketBuffer.writeString()` (длина строки) и `writeVarInt()`.  
Это НЕ часть фрейма — только payload.  
> VarInt is used inside `PacketBuffer.writeString()` (string length) and `writeVarInt()`.  
> It is NOT part of the frame — only the payload.

```
1 байт:  0xxxxxxx                 (значения 0–127)
2 байта: 1xxxxxxx 0xxxxxxx        (значения 128–16383)
3 байта: 1xxxxxxx 1xxxxxxx 0xxx…  (и т.д.)
```

---

## Ограничения / Limits

| Параметр | Значение |
|----------|----------|
| Максимальный размер фрейма | 65535 байт |
| Максимальный payload | 65527 байт (65535 - 4B id - 4B len) |
| VarInt max | 2^31 - 1 |
| Максимальная длина строки | ограничена размером payload |

Лимит 65535 задан в `LengthFieldBasedFrameDecoder`. Увеличить через `workerThreads` и изменение параметра.  
> 65535 limit set in `LengthFieldBasedFrameDecoder`. Increase by changing `maxFrameLength`.
