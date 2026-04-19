# Пакеты и кодеки / Packets & Codecs

## Интерфейс Packet / The Packet Interface

`Packet` — маркерный интерфейс. Никаких методов.  
Рекомендуется использовать `record` — неизменяемый по умолчанию.  
> `Packet` is a marker interface with no methods. Use `record` for immutability by default.

```java
public interface Packet {}
```

---

## Определение пакета / Defining a Packet

```java
@PacketId(42)          // уникальный ID в сети / unique network ID
@Codec(MyCodec.class)  // кодек для сериализации / serialization codec
public record MyPacket(String name, int level, boolean online) implements Packet {}
```

**Правила / Rules:**
- `@PacketId` — уникальный `int` в рамках приложения; повтор → `NetworkException` при регистрации
- `@Codec` — класс должен иметь конструктор без аргументов (используется при `scanPackage()`)

---

## PacketCodec\<T\>

```java
public interface PacketCodec<T extends Packet> {
    void encode(T packet, PacketBuffer buffer) throws PacketEncodeException;
    T    decode(PacketBuffer buffer)           throws PacketDecodeException;
}
```

### Пример / Example

```java
public final class MyCodec implements PacketCodec<MyPacket> {

    @Override
    public void encode(MyPacket pkt, PacketBuffer buf) {
        buf.writeString(pkt.name());
        buf.writeVarInt(pkt.level());
        buf.writeBoolean(pkt.online());
    }

    @Override
    public MyPacket decode(PacketBuffer buf) {
        String  name   = buf.readString();
        int     level  = buf.readVarInt();
        boolean online = buf.readBoolean();
        return new MyPacket(name, level, online);
    }
}
```

---

## PacketBuffer

Fluent-обёртка над `ByteBuffer`. Auto-grow при записи. Protobuf-совместимый VarInt.  
> Fluent wrapper over `ByteBuffer`. Auto-grows on write. Protobuf-compatible VarInt.

### Запись / Write Methods

| Метод | Байт | Описание |
|-------|------|----------|
| `writeVarInt(int)` | 1–5 | Protobuf unsigned varint |
| `writeLong(long)` | 8 | Big-endian |
| `writeDouble(double)` | 8 | IEEE 754 |
| `writeBoolean(boolean)` | 1 | 0x00 / 0x01 |
| `writeString(String)` | varint + UTF-8 bytes | Length-prefixed UTF-8 |
| `writeBytes(byte[])` | n | Raw bytes |

### Чтение / Read Methods

`readVarInt()`, `readLong()`, `readDouble()`, `readBoolean()`, `readString()`, `readBytes(int)`

### Утилиты / Utilities

```java
PacketBuffer buf = PacketBuffer.allocate(256);   // выделить / allocate
PacketBuffer buf = PacketBuffer.wrap(byteArray); // обернуть / wrap

byte[] arr = buf.toArray();          // все записанные байты / all written bytes
int    n   = buf.readableBytes();    // сколько байт осталось читать / bytes remaining
PacketBuffer slice = buf.slice();    // zero-copy sub-buffer
buf.resetReaderIndex();              // перемотать читатель / rewind reader
```

---

## VarInt — кодирование / VarInt Encoding

Значения 0–127 занимают 1 байт. Совместимо с protobuf Base 128.  
> Values 0–127 fit in 1 byte. Compatible with protobuf Base 128 encoding.

```
Value    Bytes
0        0x00
127      0x7F
128      0x80 0x01
300      0xAC 0x02
```

---

## Порядок полей / Field Order

Кодек читает поля в том же порядке, в котором они записаны. Изменение порядка ломает совместимость.  
> Codec reads fields in the same order they were written. Changing order breaks compatibility.
