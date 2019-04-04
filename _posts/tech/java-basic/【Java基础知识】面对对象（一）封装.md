【Java基础知识】面对对象（一）封装

作为一个Java开发者必须要了解Java关于面对圣对象的知识。 脑子里要有继承层次的概念，关注多态的能力，内聚与松耦合要成为第二天性。组合就像你的面包与黄油。


假如你写了一个类，公司内部其他人也在使用你的类。如果后面因为一些原因你决定修改你的代码的行为，那么别人应该能在不修改他们的代码的情况下，使用新版本的类来替换老版本的类。

这个场景说明了面对对象的两个好处：灵活性和可维护性。 但这些不是自动获得的。需要你做些工作， 必须以支持灵活性和可维护性的方式来实现你的类与代码，Java的面对对象并不会为你设计代码。 如下所示你的类如果有一个 public 的实例变量， 而且别的程序直接设置了这个变量的值：

```java
public class BadOO {
    public int size;
}

public class ExploitBadOO {
    public static void main(String[] args) {
        BadOO b = new BadOO();
        b.size = -5; // 语法OK但用法不好！
    }
}

```

这样会很麻烦。 如果你想要控制对 size 的修改要怎么做？ 只能写一个方法来封装 size 的修改， 并将 size 变量保护起来（private）。 但是这时修改代码会同时破坏别人使用了这个类的代码 。

封装的一个主要好处就是避免你的代码修改破坏到别人的代码。 将实现细节隐藏在程序接口后面。 接口是一个别人可以调用的你代码的约定，API。 通过隐藏实现，可以在不修改使用你接口的代码的前提下重构自己的实现代码。

想要做到灵活性、可维护性和扩展性，设计的时候必须考虑到封装：
- 实例变量是受保护的（通常是private）
- 实现访问方法，强制别人通过这些方法来修改你的实例变量而不是直接修改。
- 访问方法要符合 JavaBeans 的命名规范： set<Property>, get<Property>

一般把这些访问方法叫做 setters, getters 。

```java
public class BadOO {
    private int size;
    public int getSize(){return size;}
    public void setSize(int size){this.size = size;}
}

```

上面的代码好像也没有什么用？ getter setter 什么额外的处理也没有做嘛。  即使是什么额外功能也没有getter setter 依然是值得的， 因为你可以在以后任务时候在不破坏API的情况下修改代码。 即使你现在不认为未来真的会修改这段代码，但良好的OO设计需要为未来做打算。 为了完全起见， 还是使用getter setter 更好，这样任务时候都可以自由的重构代码。
