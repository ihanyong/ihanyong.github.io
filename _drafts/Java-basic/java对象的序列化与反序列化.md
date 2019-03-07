java对象的序列化与反序列化


 * @see java.io.ObjectOutputStream
 * @see java.io.ObjectInputStream
 * @see java.io.ObjectOutput
 * @see java.io.ObjectInput
 * @see java.io.Serializable
 * @see java.io.Externalizable


使用 ObjectOutputStream 序列化对象， ObjectInputStream 用来反序列化对象 


1. 对象的类必须实现 Serializable 或 ExternalLizable 接口
2. 最好在类中指定 serialVersionUID。 如果没有指定，编译器会自动生成一个，后续开发中类有改动的话，自动生成的 serialVersionUID 也会变化，对原有对象进行反序列化时会抛出异常:
```
Exception in thread "main" java.io.InvalidClassException: olddog.serializable.Book; local class incompatible: stream classdesc serialVersionUID = 1, local class serialVersionUID = 10
    at java.io.ObjectStreamClass.initNonProxy(ObjectStreamClass.java:699)
    at java.io.ObjectInputStream.readNonProxyDesc(ObjectInputStream.java:1885)
    at java.io.ObjectInputStream.readClassDesc(ObjectInputStream.java:1751)
    at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:2042)
    at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1573)
    at java.io.ObjectInputStream.readObject(ObjectInputStream.java:431)
    at olddog.serializable.SerializableObjectExample.deserialize(SerializableObjectExample.java:23)
    at olddog.serializable.SerializableObjectExample.readObjFile(SerializableObjectExample.java:50)
    at olddog.serializable.SerializableObjectExample.main(SerializableObjectExample.java:62)
```



```java
public class Book implements Serializable {
    //public class Book  {
    public static final long serialVersionUID = 1L;

    private String name;
    private String publisher;
    private String author;

//    private String test;

    private double price;

///////// getter setter and toString...
}


public class SerializableObjectExample {

    public static void serialize(Object obj, OutputStream outputStream) throws IOException {
        try (ObjectOutputStream out = new ObjectOutputStream(outputStream)) {
            out.writeObject(obj);
            out.flush();
        }
    }

    public static Object deserialize(InputStream inputStream) throws IOException, ClassNotFoundException {
        try (ObjectInputStream input = new ObjectInputStream(inputStream)) {
            Object obj = input.readObject();
            return obj;
        }
    }


    public static Object testInMemory(Object originObj) throws IOException, ClassNotFoundException {

        ByteArrayOutputStream byteout = new ByteArrayOutputStream(1024);
        serialize(originObj, byteout);

        byte[] bytes = byteout.toByteArray();


        return deserialize(new ByteArrayInputStream(bytes));
    }


    public static Object testWithFile(String path, Object originObj) throws IOException, ClassNotFoundException {
        writeObj2File(path, originObj);
        return readObjFile(path);
    }
    public static void writeObj2File(String path, Object originObj) throws IOException {
        FileOutputStream byteout = new FileOutputStream(path);
        serialize(originObj, byteout);
    }
    public static Object readObjFile(String path) throws IOException, ClassNotFoundException {
        return deserialize(new FileInputStream(path));
    }

    public static void main(String[] args) throws IOException, ClassNotFoundException {
//        Book book = new Book();
//        book.setAuthor("qianzhongshu");
//        book.setName("weicheng");
//        book.setPublisher("zhonghuashuju");
//        System.out.println(book);

//        Book backBook = (Book) testInMemory(book);
//        Book backBook = (Book) testWithFile("E:\\myspace\\diary\\book.obj", book);
        Book backBook = (Book) readObjFile("E:\\myspace\\diary\\book.obj");

        System.out.println(backBook);
//        System.out.println(backBook==book);
    }
}

```
