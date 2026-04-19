# Шифрование / Encryption

NetLib поддерживает **прозрачное** AES-128-GCM шифрование на уровне Netty-канала.  
«Прозрачное» означает: код приложения не меняется — `ctx.send(packet)` работает как обычно,  
шифрование происходит автоматически до передачи в сеть.  
> NetLib supports **transparent** AES-128-GCM encryption at the Netty channel level.  
> "Transparent" means application code never changes — `ctx.send(packet)` works as usual,  
> encryption happens automatically before hitting the wire.

---

## Как включить / How to Enable

```java
// Генерация ключа / Generate key (128-bit AES)
KeyGenerator gen = KeyGenerator.getInstance("AES");
gen.init(128);
SecretKey key = gen.generateKey();

// Сервер / Server
NetworkServer server = Net.server()
        .port(25565)
        .codec(registry)
        .encrypt(key)          // ← одна строка / one line
        .listener(new GameHandler())
        .start();

// Клиент / Client
NetworkClient client = Net.client()
        .codec(registry)
        .encrypt(key)          // ← тот же ключ / same key
        .connect("localhost", 25565)
        .get();
```

Оба конца **обязаны** использовать один и тот же `SecretKey`.  
> Both ends **must** use the same `SecretKey`.

---

## Алгоритм / Algorithm

| Параметр | Значение |
|----------|----------|
| Алгоритм | AES/GCM/NoPadding |
| Размер ключа | 128 бит |
| Размер IV | 12 байт (случайный для каждого пакета) |
| Тег аутентификации | 128 бит (GCM tag) |

GCM обеспечивает одновременно **конфиденциальность** и **целостность** (AEAD).  
Если зашифрованные байты изменены в транзите, decrypt бросает `PipelineException`.  
> GCM provides both **confidentiality** and **integrity** (AEAD).  
> If encrypted bytes are tampered in transit, decryption throws `PipelineException`.

---

## Формат зашифрованного фрейма / Encrypted Frame Format

Без шифрования / Without encryption:
```
[4B inner-length][4B packet-id][N bytes payload]
```

С шифрованием / With encryption:
```
[4B outer-length][12B random IV][AES-GCM ciphertext + 16B tag]
                  └─ encrypted: [4B inner-length][4B packet-id][N bytes payload] ─┘
```

- `outer-length` = `12 + len(ciphertext)` — длина IV + шифротекст
- IV генерируется заново для каждого пакета (случайный `SecureRandom`)
- Весь внутренний фрейм зашифрован, включая packet-id

> `outer-length` = `12 + ciphertext.length`. IV is regenerated per-packet via `SecureRandom`.  
> The entire inner frame — including packet-id — is encrypted.

---

## Позиция в Netty-пайплайне / Position in Netty Pipeline

```
Входящие байты (inbound) ↓
  [EncryptionDecoder]          ← ByteToMessageDecoder, читает outer frame, дешифрует
  [LengthFieldBasedFrameDecoder]   ← видит уже дешифрованные байты
  [PacketDecoder]              ← читает id, вызывает codec
  [ConnectionLifecycleHandler]

Исходящие байты (outbound) ↑
  EncryptionEncoder            ← MessageToByteEncoder<ByteBuf>, шифрует перед отправкой
```

`EncryptionChannelHandler` — внутренний класс `ru.kseonyt.net.pipeline`.  
Вызывается автоматически через `.encrypt(key)`.  
> `EncryptionChannelHandler` is in `ru.kseonyt.net.pipeline`. Called automatically via `.encrypt(key)`.

---

## Управление ключами / Key Management

NetLib **не** управляет жизненным циклом ключей — это задача приложения.  
Рекомендуется:

1. **Статический pre-shared key** — для тестов и приложений с закрытым кругом клиентов
2. **ECDH handshake** — реализуй через начальные пакеты (до включения шифрования)
3. **TLS поверх NetLib** — добавь `SslHandler` в Netty pipeline до `EncryptionDecoder`

> NetLib does **not** manage key lifecycle — that's the application's responsibility.

---

## Прямое использование EncryptionInterceptor / Direct Use

Если нужно шифровать байты вне pipeline:

```java
EncryptionInterceptor enc = new EncryptionInterceptor(key);

byte[] ciphertext = enc.encrypt(plaintext);
byte[] plaintext  = enc.decrypt(ciphertext);
```
