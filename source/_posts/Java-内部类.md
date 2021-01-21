---
title: Java 内部类
date: 2020-12-25 18:20:52
tags: Java
categories: Java
---

## 内部类的作用

首先我们要知道什么事内部类



> 内部类（inner class）：定义在另一个类中的类



内部类有什么作用：



- 内部类方法可以访问该类定义所在作用域中的数据，包括 `private` 修饰的私有数据


- 内部类可以对同一包中的其他类隐藏起来


- 内部类可以实现 Java 单继承的缺陷




### 内部类方法可以访问该类定义所在作用域中的数据

我们看下面的代码



```java
public class TestExample {

    private List<Integer> list = new ArrayList<>();

    private class InnerClass {

        public Integer get(int poi) {
            return list.get(poi);
        }
    }

    public void init() {
        this.list.add(1);
        this.list.add(2);
    }

    public Integer get(int poi) {
        InnerClass innerClass = new InnerClass();
        return innerClass.get(poi);
    }

    public static void main(String[] args) {
        TestExample testExample = new TestExample();
        testExample.init();
        System.out.println(testExample.get(1));
    }
}
```



上面代码中，`InnerClass` 就是 `TestExample` 的一个内部类。可以看出 `InnerClass` 可以直接访问 `TestExample` 中定义的 list 私有变量。如果不将 `InnerClass` 定义为内部类，想要访问 list 私有变量，没有 public getXXX 方式是不行的。



为什么内部类可以随意访问外部类的成员呢？



> 当外部类的对象创建一个内部类的对象时，内部类对象必定会秘密捕获一个指向外部类对象的引用，然后访问外部类成员时，就是用那个引用来选择外围类的成员的。这些编译器已经帮我们处理了
>
> 
>
> 另外注意内部类只是一种编译器现象，与虚拟机无关。编译器会将内部类编译成「外部类名$内部类名」的常规文件，虚拟机对此一无所知



### 内部类可以对同一包中的其他类隐藏起来

我们知道，普通的类不能使用 `private` `protected` 访问权限符来修饰，而内部类则可以使用 `private` 和 `protected` 来修饰。当我们使用 `private` 来修饰内部类的时候，这个类就对外隐藏了。这有什么用？当内部类失信案例 某个接口的时候，再进行向上转型，对外部来说，就完全实现了接口的实现。如下：



```java
public interface InterfaceTest {

    void test();
}

public class Example {

	// 内部类实现 InterfaceTest 接口
    private class InsideClass implements InterfaceTest {

        public void test() {
            System.out.println("这是一个测试");
        }
    }

	// 返回一个 InterfaceTest 实例
    public InterfaceTest getIn() {
        return new InsideClass();
    }
}

public class TestExample {

    public static void main(String[] args) {
        Example example = new Example();
        InterfaceTest in = example.getIn();
        in.test();
    }
}
```



上面的代码我们只知道 Example 的 `getIn()` 方法能返回一个 `InterfaceTest` 实例但并不知道这个实例是怎么实现的。而且由于 `InnerClass` 是 `private` 的，所以我们如果不看代码的话根本看不到这个具体类的名字，所以说它可以很好的实现隐藏



### 内部类可以实现 Java 单继承的缺陷

Java 是不允许使用 `extends` 去继承多个类的，但内部类的引入可以很好地解决这个事情。



每个内部类都可以继承自一个（接口的）实现，所以无论外围类是否已经继承了某个（接口的）实现，对于内部类没有影响。如果没有内部类提供的，可以继承多个具体的或抽象的类的能力，一些设计与编程问题就难以解决。接口解决了部分问题，一个类可以实现多个接口，内部类允许继承多个非接口类型（类或抽象类）



Java 只能继承一个类这个大家都知道，而在有内部类之前它的多重继承方式是用接口来实现的。但使用接口有时候有很多不方便 的地方。比如我们实现一个接口就必须实现它里面的多有方法。而有了内部类就不一样了，它可以使我们的类继承多个具体类或抽象类。如下：



```java
// 抽象类 A
public abstract class ClassA {

    public String name() {
        return "liuyifei";
    }

    public abstract void doSomething();
}

// 具体类 B
public class ClassB {

    public int age() {
        return 25;
    }
}

public class MainExample {

    private class Test1 extends ClassA {
        public String name() {
            return super.name();
        }

        @Override
        public void doSomething() {
            System.out.println("测试doSomething");
        }
    }

    private class Test2 extends ClassB {
        public int age() {
            return super.age();
        }
    }

    public String name() {
        return new Test1().name();
    }

    public int age() {
        return new Test2().age();
    }

    public void doSomething() {
        new Test1().doSomething();
    }


    public static void main(String[] args) {
        MainExample mi = new MainExample();
        System.out.println("姓名" + mi.name());
        System.out.println("年龄" + mi.age());
        mi.doSomething();
    }
}
```



上面这个例子可以看出来，MainExample 类通过内部类拥有了 ClassA 和 ClassB 的两个类的继承关系。



## 内部类与外部类的关系

对于非静态内部类，内部类的创建依赖外部类的实例对象，在没有外部类实例之前是无法创建内部类的

内部类是一个相对独立的实体，与外部类不是 `is - a` 关系

创建内部类的时刻并不依赖于外部类的创建



### 创建内部类的时刻并不依赖于外部类的创建

这是《Thinking In Java》中的一句话，大部分人看到这里会断章取义的认为，内部类的创建不依赖于类的创建，这种理解是错误的，去掉「时刻」二字就会变了一个味道



事实上静态内部类「嵌套类」的确不依赖于外部类的创建，因为 `static` 并不依赖于实例，而依赖于 `Class`本身



但是对于普通的内部类，其必须依赖于外部类实例的创建。正如上面第一条关系所说：对于非静态内部类，内部类的创建依赖于外部类的实例对象，在没有外部类实例之前是没办法创建内部类的



对于普通内部类创建方法有两种：



```java
public class ClassOuter {

    public void fun() {
        System.out.println("外部类方法");
    }

    public class InnerClass {

    }
}

public class TestInnerClass {

    public static void main(String[] args) {
        // 创建方式 1
        ClassOuter.InnerClass innerClass = new ClassOuter().new InnerClass();

        // 创建方式 2
        ClassOuter outer = new ClassOuter();
        ClassOuter.InnerClass inner = outer.new InnerClass();
    }
}
```



需要注意的是，**正是由于这种依赖关系，所以普通内部类中不允许有 static 成员，包括嵌套类（内部类的静态内部类）**。因为 static 本身是针对类本身来说的，又由于非 static 内部类总是由一个外部的对象生成，既然与对象相关，就没有静态的字段和方法。当然静态内部类不依赖于外部类，所以其允许有 static 成员



### 内部类是一个相对独立的实体，与外部类不是 is - a 关系

首先什么是「is - a 关系」。is - a 关系是指继承关系。知道什么是 is - a 关系后，内部类与外部类不是 is - a 关系就很容易理解了



对于内部类是一个相对独立的实体，我们可以从两个方面来理解这句话：



- 一个外部类可以拥有多个内部类对象，而它们之间没有任何关系，是独立的个体

- 从编译结果来看，内部类被表现为「外部类$内部类.class」，所以对于虚拟机来说它与一个单独的类来说没什么区别。但是我们知道它们是有关系的，因为内部类默认持有一个外部类的引用



## 内部类的分类

内部类可以分为：静态内部类（嵌套类）和非静态内部类。非静态内部类有可以分为：成员内部类、方法内部类匿名内部类。



静态内部类和非静态内部类的区别：



1. 静态内部类可以有静态成员，而非静态内部类不能有静态成员
2. 静态内部类可以访问外部类的静态变量，而不可访问外部类的非静态变量
3. 非静态内部类的非静态成员可以访问外部类的非静态变量
4. 静态内部类的创建不依赖于外部类，而非静态内部类必须依赖于外部类的创建而创建



我们通过一个例子来加深一下理解



```java
public class ClassOuter {

    private int noStaticInt = 1;
    private static int STATIC_INT = 2;

    public void fun() {
        System.out.println("外部类方法");
    }

    public class InnerClass {
        // 下面那句代码编译器会报错，非静态内部类不能有静态成员
        // static int num  = 1;

        public void fun() {
            // 非静态内部类的非静态成员可以访问外部类的非静态变量
            System.out.println(STATIC_INT);
            System.out.println(noStaticInt);
        }
    }

    public static class StaticInnerClass {

        // 静态内部类可以有静态成员
        static int NUM = 1;

        public void fun() {
            System.out.println(STATIC_INT);
            // 下面那句代码会报错，不可访问外部类的非静态变量
            // System.out.println(noStaticInt);
        }
    }
}

public class TestInnerClass {

    public static void main(String[] args) {

        // 非静态内部类 创建方式 1
        ClassOuter.InnerClass innerClass = new ClassOuter().new InnerClass();

        // 非静态内部类 创建方式 2
        ClassOuter outer = new ClassOuter();
        ClassOuter.InnerClass inner = outer.new InnerClass();

        // 静态内部类的创建方式
        ClassOuter.StaticInnerClass staticInnerClass = new ClassOuter.StaticInnerClass();
    }
}
```



### 局部内部类

> 如果一个内部类只在一个方法中使用到了，那么我们可以将这个类定义在方法内部，这种内部类成为局部内部类。其作用域仅限于该方法



局部内部类有几点值得我们注意的地方：



- 局部内部类不允许使用访问权限修饰符。`private` `public` `protected` 都不允许

- 局部内部类对外完全隐藏，除了创建这个类的方法可以访问它，其它的地方都是不允许访问的

- 局部内部类与成员内部类不同之处就是它可以引用方法的局部变量，但是该局部变量必须声明为 `final`，而且内部不允许修改该变量的值。如果不加 `final`，编译器会自动加上 `final`



```
public class ClassOuter {

    private int noStaticInt = 1;
    private static int STATIC_INT = 2;

    private Integer params;

    public void fun() {
        System.out.println("外部类方法");
    }

    public void testFunctionClass() {

        Integer params = 0;

        class FunctionClass {
            private void fun() {
                System.out.println("局部内部类的输出");
                System.out.println(STATIC_INT);
                System.out.println(noStaticInt);
                System.out.println(params);
                // params 不可变所以这句话编译错误
                // params++;
            }
        }

        FunctionClass functionClass = new FunctionClass();
        functionClass.fun();
    }
}
```



### 匿名内部类

- 匿名内部类是没有访问修饰符的

- 匿名内部类必须继承一个抽象类或者接口

- 匿名内部类中不能存在任何静态成员或方法

- 匿名内部类是没有构造方法的，因为它没有类名

- 与局部内部类相同，匿名内部类也可以引用局部变量，此变量也必须声明为 final



```java
public class Button {

    public void click(final int params) {
        new ActionListener() {
            @Override
            public void onAction() {
                System.out.println("click action..." + params);
            }
        }.onAction();
    }

    public interface ActionListener {
        void onAction();
    }

    public static void main(String[] args) {
        Button button = new Button();
        button.click(2);
    }
}
```



### 为什么局部变量需要 final 修饰

因为局部变量和匿名内部类的生命周期不同



匿名内部类是创建后存储在堆中的，而方法中的局部变量是存储在 Java 栈中。当方法执行完毕后，就进行退栈，同时局部变量也会消失。那么此时匿名内部类还有可能在堆中存储着，那么匿名内部类要到哪里去找这个局部变量呢？



为了解决这个问题，编译器自动帮我们在匿名内部类中创建了一个局部变量的备份，就是说即使方法执行结束，匿名内部类中还有一个备份，自然就不怕找不到了



但是，如果局部变量中的 a 不停的在变化，那么岂不是也要让备份 a 的变量无时无刻地变化。为了保持局部变量与匿名内部类中备份域保持一致，编译器不得不规定死这些局部域必须是常量，一旦赋值不能再发生变化。这就是为什么匿名内部类应用外部方法的域必须是常量域的原因了



特别注意：**在 Java 8 中已经去掉对 final 的修饰限制，但其实只要在匿名内部类使用了，该变量还是会自动变成 final 类型（只能使用，不能赋值）**



## 实际开发中内部类有可能会引起的问题

### 内部类会造成程序的内存泄露

要想了解为啥内部类会造成内存泄露，我们需要了解 Java 虚拟机的回收机制。Java 的内存回收机制通过「可达性分析」来实现。即 Java 虚拟机会通过内存回收机制来判定引用是否可达，如果不可达就会在某些时刻去回收这些引用



那么内部类在什么情况下会造成内存泄露的可能呢



> 1. 如果一个匿名内部类没有被任何引用持有，那么匿名内部类对象用完就有机会被回收
> 2. 如果内部类仅仅只是在外部类中被引用，当外部类不再被引用时，外部类和内部类就可以被 GC 回收
> 3. 如果当内部类的引用被外部类以外的其他类引用时，就会造成内部类和外部类无法被 GC 回收的情况，即使外部类没有被引用，因为内部类持有指向外部类的引用

‘

## 参考资料

[搞懂 Java 内部类](https://juejin.cn/post/6844903566293860366)

