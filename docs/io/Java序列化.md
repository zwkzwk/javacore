---
title: 深入理解 Java 序列化
categories: ['java', 'javacore']
tags: ['java', 'javacore', 'basics', 'serialize']
date: 2018-06-04 17:59
---

# 深入理解 Java 序列化

> :notebook: 本文已归档到：「[javacore](https://github.com/dunwu/javacore)」
>
> :keyboard: 本文中的示例代码已归档到：「[javacore](https://github.com/dunwu/javacore/tree/master/codes/basics/src/main/java/io/github/dunwu/javacore/serialize)」

<!-- TOC depthFrom:2 depthTo:3 -->

- [简介](#简介)
- [序列化和反序列化](#序列化和反序列化)
- [Serializable 接口](#serializable-接口)
    - [serialVersionUID](#serialversionuid)
- [默认序列化机制](#默认序列化机制)
- [非默认序列化机制](#非默认序列化机制)
    - [transient 关键字](#transient-关键字)
    - [Externalizable 接口](#externalizable-接口)
    - [Externalizable 接口的替代方法](#externalizable-接口的替代方法)
    - [readResolve() 方法](#readresolve-方法)
- [序列化工具](#序列化工具)
- [小结](#小结)
- [参考资料](#参考资料)

<!-- /TOC -->

## 简介

<div align="center"><img src="http://dunwu.test.upcdn.net/snap/1553224129484.png!zp"/></div>

- **序列化（serialize）** - 序列化是将对象转换为字节流。
- **反序列化（deserialize）** - 反序列化是将字节流转换为对象。
- **序列化用途**
  - 序列化可以将对象的字节序列持久化——保存在内存、文件、数据库中。
  - 在网络上传送对象的字节序列。
  - RMI(远程方法调用)

> :bell: 注意：使用 Java 对象序列化，在保存对象时，会把其状态保存为一组字节，在未来，再将这些字节组装成对象。必须注意地是，对象序列化保存的是对象的”状态”，即它的成员变量。由此可知，**对象序列化不会关注类中的静态变量**。

## 序列化和反序列化

Java 通过对象输入输出流来实现序列化和反序列化：

- `java.io.ObjectOutputStream` 类的 `writeObject()` 方法可以实现序列化；
- `java.io.ObjectInputStream` 类的 `readObject()` 方法用于实现反序列化。

序列化和反序列化示例：

```java
public class SerializeDemo01 {
    enum Sex {
        MALE,
        FEMALE
    }


    static class Person implements Serializable {
        private static final long serialVersionUID = 1L;
        private String name = null;
        private Integer age = null;
        private Sex sex;

        public Person() { }

        public Person(String name, Integer age, Sex sex) {
            this.name = name;
            this.age = age;
            this.sex = sex;
        }

        @Override
        public String toString() {
            return "Person{" + "name='" + name + '\'' + ", age=" + age + ", sex=" + sex + '}';
        }
    }

    /**
     * 序列化
     */
    private static void serialize(String filename) throws IOException {
        File f = new File(filename); // 定义保存路径
        OutputStream out = new FileOutputStream(f); // 文件输出流
        ObjectOutputStream oos = new ObjectOutputStream(out); // 对象输出流
        oos.writeObject(new Person("Jack", 30, Sex.MALE)); // 保存对象
        oos.close();
        out.close();
    }

    /**
     * 反序列化
     */
    private static void deserialize(String filename) throws IOException, ClassNotFoundException {
        File f = new File(filename); // 定义保存路径
        InputStream in = new FileInputStream(f); // 文件输入流
        ObjectInputStream ois = new ObjectInputStream(in); // 对象输入流
        Object obj = ois.readObject(); // 读取对象
        ois.close();
        in.close();
        System.out.println(obj);
    }

    public static void main(String[] args) throws IOException, ClassNotFoundException {
        final String filename = "d:/text.dat";
        serialize(filename);
        deserialize(filename);
    }
}
// Output:
// Person{name='Jack', age=30, sex=MALE}
```

## Serializable 接口

**被序列化的类必须属于 Enum、Array 和 Serializable 类型其中的任何一种**。

**如果不是 Enum、Array 的类，如果需要序列化，必须实现 `java.io.Serializable` 接口，否则将抛出 `NotSerializableException` 异常**。这是因为：在序列化操作过程中会对类型进行检查，如果不满足序列化类型要求，就会抛出异常。

我们不妨做一个小尝试：将 SerializeDemo01 示例中 Person 类改为如下实现，然后看看运行结果。

```java
public class UnSerializeDemo {
    static class Person { // 其他内容略 }
    // 其他内容略
}
```

输出：结果就是出现如下异常信息。

```
Exception in thread "main" java.io.NotSerializableException:
...
```

### serialVersionUID

请注意 `serialVersionUID` 字段，你可以在 Java 世界的无数类中看到这个字段。

`serialVersionUID` 有什么作用，如何使用 `serialVersionUID`？

**`serialVersionUID` 是 Java 为每个序列化类产生的版本标识**。它可以用来保证在反序列时，发送方发送的和接受方接收的是可兼容的对象。如果接收方接收的类的 `serialVersionUID` 与发送方发送的 `serialVersionUID` 不一致，会抛出 `InvalidClassException`。

如果可序列化类没有显式声明 `serialVersionUID`，则序列化运行时将基于该类的各个方面计算该类的默认 `serialVersionUID` 值。尽管这样，还是**建议在每一个序列化的类中显式指定 `serialVersionUID` 的值**。因为不同的 jdk 编译很可能会生成不同的 `serialVersionUID` 默认值，从而导致在反序列化时抛出 `InvalidClassExceptions` 异常。

**`serialVersionUID` 字段必须是 `static final long` 类型**。

我们来举个例子：

（1）有一个可序列化类 Person

```java
public class Person implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private Integer age;
    private String address;
    // 构造方法、get、set 方法略
}
```

（2）开发过程中，对 Person 做了修改，增加了一个字段 email，如下：

```java
public class Person implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private Integer age;
    private String address;
    private String email;
    // 构造方法、get、set 方法略
}
```

由于这个类和老版本不兼容，我们需要修改版本号：

```java
private static final long serialVersionUID = 2L;
```

再次进行反序列化，则会抛出 `InvalidClassException` 异常。

综上所述，我们大概可以清楚：**`serialVersionUID` 用于控制序列化版本是否兼容**。若我们认为修改的可序列化类是向后兼容的，则不修改 `serialVersionUID`。

## 默认序列化机制

如果仅仅只是让某个类实现 `Serializable` 接口，而没有其它任何处理的话，那么就会使用默认序列化机制。

使用默认机制，在序列化对象时，不仅会序列化当前对象本身，还会对其父类的字段以及该对象引用的其它对象也进行序列化。同样地，这些其它对象引用的另外对象也将被序列化，以此类推。所以，如果一个对象包含的成员变量是容器类对象，而这些容器所含有的元素也是容器类对象，那么这个序列化的过程就会较复杂，开销也较大。

> 注意：这里的父类和引用对象既然要进行序列化，那么它们当然也要满足序列化要求：**被序列化的类必须属于 Enum、Array 和 Serializable 类型其中的任何一种**。

## 非默认序列化机制

在现实应用中，有些时候不能使用默认序列化机制。比如，希望在序列化过程中忽略掉敏感数据，或者简化序列化过程。下面将介绍若干影响序列化的方法。

### transient 关键字

**当某个字段被声明为 transient 后，默认序列化机制就会忽略该字段**。

我们将 SerializeDemo01 示例中的内部类 Person 的 age 字段声明为 `transient`，如下所示：

```java
public class SerializeDemo02 {
    static class Person implements Serializable {
        transient private Integer age = null;
        // 其他内容略
    }
    // 其他内容略
}
// Output:
// name: Jack, age: null, sex: MALE
```

从输出结果可以看出，age 字段没有被序列化。

### Externalizable 接口

无论是使用 `transient` 关键字，还是使用 `writeObject()` 和 `readObject()` 方法，其实都是基于 `Serializable` 接口的序列化。

JDK 中提供了另一个序列化接口--`Externalizable`。

**可序列化类实现 `Externalizable` 接口之后，基于 `Serializable` 接口的默认序列化机制就会失效**。

我们来基于 SerializeDemo02 再次做一些改动，代码如下：

```java
public class ExternalizeDemo01 {
    static class Person implements Externalizable {
        transient private Integer age = null;
        // 其他内容略

        private void writeObject(ObjectOutputStream out) throws IOException {
            out.defaultWriteObject();
            out.writeInt(age);
        }

        private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
            in.defaultReadObject();
            age = in.readInt();
        }

        @Override
        public void writeExternal(ObjectOutput out) throws IOException { }

        @Override
        public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException { }
    }
     // 其他内容略
}
// Output:
// call Person()
// name: null, age: null, sex: null
```

从该结果，一方面可以看出 Person 对象中任何一个字段都没有被序列化。另一方面，如果细心的话，还可以发现这此次序列化过程调用了 Person 类的无参构造方法。

- **`Externalizable` 继承于 `Serializable`，它增添了两个方法：`writeExternal()` 与 `readExternal()`。这两个方法在序列化和反序列化过程中会被自动调用，以便执行一些特殊操作**。当使用该接口时，序列化的细节需要由程序员去完成。如上所示的代码，由于 `writeExternal()` 与 `readExternal()` 方法未作任何处理，那么该序列化行为将不会保存/读取任何一个字段。这也就是为什么输出结果中所有字段的值均为空。
- 另外，**若使用 Externalizable 进行序列化，当读取对象时，会调用被序列化类的无参构造方法去创建一个新的对象；然后再将被保存对象的字段的值分别填充到新对象中**。这就是为什么在此次序列化过程中 Person 类的无参构造方法会被调用。由于这个原因，实现 `Externalizable` 接口的类必须要提供一个无参的构造方法，且它的访问权限为 `public`。

对上述 Person 类作进一步的修改，使其能够对 name 与 age 字段进行序列化，但要忽略掉 gender 字段，如下代码所示：

```java
public class ExternalizeDemo02 {
    static class Person implements Externalizable {
        transient private Integer age = null;
        // 其他内容略

        private void writeObject(ObjectOutputStream out) throws IOException {
            out.defaultWriteObject();
            out.writeInt(age);
        }

        private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
            in.defaultReadObject();
            age = in.readInt();
        }

        @Override
        public void writeExternal(ObjectOutput out) throws IOException {
            out.writeObject(name);
            out.writeInt(age);
        }

        @Override
        public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
            name = (String) in.readObject();
            age = in.readInt();
        }
    }
     // 其他内容略
}
// Output:
// call Person()
// name: Jack, age: 30, sex: null
```

### Externalizable 接口的替代方法

实现 `Externalizable` 接口可以控制序列化和反序列化的细节。它有一个替代方法：实现 `Serializable` 接口，并添加 `writeObject(ObjectOutputStream out)` 与 `readObject(ObjectInputStream in)` 方法。序列化和反序列化过程中会自动回调这两个方法。

示例如下所示：

```java
public class SerializeDemo03 {
    static class Person implements Serializable {
        transient private Integer age = null;
        // 其他内容略

        private void writeObject(ObjectOutputStream out) throws IOException {
            out.defaultWriteObject();
            out.writeInt(age);
        }

        private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
            in.defaultReadObject();
            age = in.readInt();
        }
        // 其他内容略
    }
    // 其他内容略
}
// Output:
// name: Jack, age: 30, sex: MALE
```

在 `writeObject()` 方法中会先调用 `ObjectOutputStream` 中的 `defaultWriteObject()` 方法，该方法会执行默认的序列化机制，如上节所述，此时会忽略掉 age 字段。然后再调用 writeInt() 方法显示地将 age 字段写入到 `ObjectOutputStream` 中。readObject() 的作用则是针对对象的读取，其原理与 writeObject() 方法相同。

> 注意：`writeObject()` 与 `readObject()` 都是 `private` 方法，那么它们是如何被调用的呢？毫无疑问，是使用反射。详情可见 `ObjectOutputStream` 中的 `writeSerialData` 方法，以及 `ObjectInputStream` 中的 `readSerialData` 方法。

### readResolve() 方法

当我们使用 Singleton 模式时，应该是期望某个类的实例应该是唯一的，但如果该类是可序列化的，那么情况可能会略有不同。此时对第 2 节使用的 Person 类进行修改，使其实现 Singleton 模式，如下所示：

```java
public class SerializeDemo04 {

    enum Sex {
        MALE, FEMALE
    }

    static class Person implements Serializable {
        private static final long serialVersionUID = 1L;
        private String name = null;
        transient private Integer age = null;
        private Sex sex;
        static final Person instatnce = new Person("Tom", 31, Sex.MALE);

        private Person() {
            System.out.println("call Person()");
        }

        private Person(String name, Integer age, Sex sex) {
            this.name = name;
            this.age = age;
            this.sex = sex;
        }

        public static Person getInstance() {
            return instatnce;
        }

        private void writeObject(ObjectOutputStream out) throws IOException {
            out.defaultWriteObject();
            out.writeInt(age);
        }

        private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
            in.defaultReadObject();
            age = in.readInt();
        }

        public String toString() {
            return "name: " + this.name + ", age: " + this.age + ", sex: " + this.sex;
        }
    }

    /**
     * 序列化
     */
    private static void serialize(String filename) throws IOException {
        File f = new File(filename); // 定义保存路径
        OutputStream out = new FileOutputStream(f); // 文件输出流
        ObjectOutputStream oos = new ObjectOutputStream(out); // 对象输出流
        oos.writeObject(new Person("Jack", 30, Sex.MALE)); // 保存对象
        oos.close();
        out.close();
    }

    /**
     * 反序列化
     */
    private static void deserialize(String filename) throws IOException, ClassNotFoundException {
        File f = new File(filename); // 定义保存路径
        InputStream in = new FileInputStream(f); // 文件输入流
        ObjectInputStream ois = new ObjectInputStream(in); // 对象输入流
        Object obj = ois.readObject(); // 读取对象
        ois.close();
        in.close();
        System.out.println(obj);
        System.out.println(obj == Person.getInstance());
    }

    public static void main(String[] args) throws IOException, ClassNotFoundException {
        final String filename = "d:/text.dat";
        serialize(filename);
        deserialize(filename);
    }
}
// Output:
// name: Jack, age: null, sex: MALE
// false
```

值得注意的是，从文件中获取的 Person 对象与 Person 类中的单例对象并不相等。**为了能在单例类中仍然保持序列的特性，可以使用 `readResolve()` 方法**。在该方法中直接返回 Person 的单例对象。我们在 SerializeDemo04 示例的基础上添加一个 `readObject` 方法， 如下所示：

```java
public class SerializeDemo05 {
    // 其他内容略

    static class Person implements Serializable {

        // 添加此方法
        private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
            in.defaultReadObject();
            age = in.readInt();
        }
        // 其他内容略
    }

    // 其他内容略
}
// Output:
// name: Tom, age: 31, sex: MALE
// true
```

## 序列化工具

Java 官方的序列化存在许多问题，因此，很多人更愿意使用优秀的第三方序列化工具来替代 Java 自身的序列化机制。

Java 官方的序列化主要体现在以下方面：

- Java 官方的序列化性能不高，序列化后的数据相对于一些优秀的序列化的工具，还是要大不少，这大大影响存储和传输的效率。
- Java 官方的序列化一定需要实现 Serializable 接口。
- Java 官方的序列化需要关注 serialVersionUID。
- Java 官方的序列无法跨语言使用。

当然我们还有更加优秀的一些序列化和反序列化的工具，根据不同的使用场景可以自行选择！

- [thrift](https://github.com/apache/thrift)、[protobuf](https://github.com/protocolbuffers/protobuf) - 适用于对性能敏感，对开发体验要求不高的内部系统。
- [hessian](http://hessian.caucho.com/doc/hessian-overview.xtp) - 适用于对开发体验敏感，性能有要求的内外部系统。
- [jackson](https://github.com/FasterXML/jackson)、[gson](https://github.com/google/gson)、[fastjson](https://github.com/alibaba/fastjson) - 适用于对序列化后的数据要求有良好的可读性（转为 json 、xml 形式）。

## 小结

<div align="center"><img src="http://dunwu.test.upcdn.net/snap/1553227663192.png!zp"/></div>

## 参考资料

- [Java 编程思想](https://book.douban.com/subject/2130190/)
- [JAVA 核心技术（卷 1）](https://book.douban.com/subject/3146174/)
- http://www.hollischuang.com/archives/1140
- http://www.codenuclear.com/serialization-deserialization-java/
- http://www.blogjava.net/jiangshachina/archive/2012/02/13/369898.html
- [Java 序列化的高级认识](https://www.ibm.com/developerworks/cn/java/j-lo-serial/index.html)
- https://agapple.iteye.com/blog/859052
