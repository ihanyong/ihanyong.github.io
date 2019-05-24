【TODO】Java_NIO_Buffe


Buffer

- capacity
- limit
- position

除了boolean 之外，每一个Java原始类型都对应一个Buffer的子类。

### 数据传输
每个子类都有两种类型的get/put操作
- 相对位置的get/put
-绝对位置的get/put


### Marking & resetting

### Invariants
0 <= mark <= position <= limit <= capacity

### Clearing, flipping, rewinding

- clear： 将 postion 置0， limit 置为capacity, mark 为-1。 相当于清除buffer中的所有数据。 其实只是修改了position, limit，mark的位置， 真正的数据没有动。
- flip： 将 limit 置为position,  postion 置0，, mark 为-1。 可以用于将写模式转换为读取模式。
- rewind： 将 postion 置0，, mark 为-1。 可用于从头再次读取buffer。


直接看代码，
```java
    public final Buffer clear() {
        position = 0;
        limit = capacity;
        mark = -1;
        return this;
    }
    public final Buffer flip() {
        limit = position;
        position = 0;
        mark = -1;
        return this;
    }
    public final Buffer rewind() {
        position = 0;
        mark = -1;
        return this;
    }
```

### thread safety
not thread safe

### Invocation chaining

```
b.flip();
b.positin(23);
b.limit(42);
```
b.flip().position(23).limit(42);
```
```