title: Java 8新特性(一)
author: Loux

cover: /img/post/24.jpg

tags:

  - 后端知识
categories:
  - 后端知识
date: 2019-11-07 14:38:00
---
说来惭愧，JDK1.8 2014年就发布了，现在都2019年了对其新特性用的还是不多，只使用过Lambda表达式。最近这段时间项目进度不忙，平时上班也有比较多的空闲时间，这才将Java 8的新特性系统的整理一遍。

总的来说，大概新增了Lambda表达式、Stream流、Optional类、新的时间和日期ApI

# Lambda表达式

 Lambda表达式（lambda expression）是一个匿名函数，由数学中的λ演算而得名。在Java 8中可以把Lambda表达式理解为匿名函数，它没有名称，但是有参数列表、函数主体、返回类型等。就我个人来说，感觉比较像ES6中的箭头函数。

Lambda表达式的语法如下：

> ( parameters ) -> { statements;  }

而使用Lambda表达式的目的就是让代码变得更加的简洁、易于理解

下面举两个例子来说明

案例一：集合排序

```java
// Comparator排序
List<Integer> list = Arrays.asList(3, 1, 4, 5, 2);
list.sort(new Comparator<Integer>() {
    @Override
    public int compare(Integer o1, Integer o2) {
        return o1.compareTo(o2);
    }
});

// 使用Lambda表达式简化
list.sort((o1, o2) -> o1.compareTo(o2));
```

案例二：创建线程

```java
// Runnable代码块
Thread thread = new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello Man!");
    }
});

// 使用Lambda表达式简化
Thread thread = new Thread(() -> System.out.println("Hello Man!"));
```

可以看出，只要是内部类的代码块，就可以使用Lambda表达式。 `Comparator`排序的Lambda表达式还可以进一步简化： 

> list.sort(Integer::comparator);

 这种写法被称为 **方法引用**，方法引用是Lambda表达式的简便写法。如果你的Lambda表达式只是调用这个方法，最好使用名称调用，而不是描述如何调用，这样可以进一步提高代码的可读性。 

同样的在遍历集合输出时也可以使用这种写法。

# Stream流

 新增的Stream API与`InputStream`和`OutputStream`是完全不同的概念，Stream API是对Java中集合操作的增强，可以利用它进行各种过滤、排序、分组、聚合等操作。Stream API配合Lambda表达式可以加大的提高代码可读性和编码效率，Stream API也支持并行操作，我们不用再花费很多精力来编写容易出错的多线程代码了，Stream API已经替我们做好了，并且充分利用多核CPU的优势。借助Stream API和Lambda，我们可以很容易的编写出高性能的并发处理程序。 

例如要在Java程序中将User集合筛选获取所有年龄大于20岁的用户的名称，并按照用户的创建时间进行排序 ，如果使用普通写法就会写出冗长的代码，效率不高，还容易出Bug。而用Stream的方式可以快速写出来，既简单可读性又强。

```java
List<User> list = getUserList();
//查询User集合中年龄大于20的按生日排序的集合，并重新组合为用户姓名集合
List<String> userNames = list.stream()
      .filter(user -> user.getAge() > 20)
      .sorted(Comparator.comparing(User::getBirthday))
      .map(User::getName)
      .collect(toList());
```

 在Java中，集合是一种数据结构，或者说是一种容器，用于存放数据，流不是容器，它不关心数据的存放，只关注如何处理。可以把流当做是Java中的`Iterator`，不过它比`Iterator`的功能更加丰富、高效

创建流的方式有很多，在这里介绍以下几种

## 由值创建流

 使用静态方法`Stream.of()`创建流，该方法接收一个变长参数： 

`Stream<Stream> stream = Stream.of("A", "B", "C", "D"); `

 也可以使用静态方法`Stream.empty()`创建一个空的流： 

` Stream<Stream> stream = Stream.empty(); `

## 通过函数创建流

 `java.util.stream.Stream`中有两个静态方法用于从函数生成流，他们分别是`Stream.generate()`和`Stream.iterate()`： 

```java
// iteartor
Stream.iterate(1, n -> n + 1).limit(50).forEach(System.out::println);

// generate
Stream.generate(() -> "welcome!").limit(10).forEach(System.out::println);
```

第一个方法会打印出1~50的整数，而第二个方法会打印十次`welcome!`

我们需要注意的是这两个方法创建的都是无限流，即若不加以限制则永远不会停止。所以我们使用 limit() 函数来限制它避免打印无数个值。

## 使用流

Stream接口中包含着许多对流操作的方法，这些方法分别为：

- `filter()`：对流的元素过滤
- `map()`：将流的元素映射成另一个类型
- `distinct()`：去除流中重复的元素
- `sorted()`：对流的元素排序
- `forEach()`：对流中的每个元素执行某个操作
- `peek()`：与`forEach()`方法效果类似，不同的是，该方法会返回一个新的流，而`forEach()`无返回
- `limit()`：截取流中前面几个元素
- `skip()`：跳过流中前面几个元素
- `toArray()`：将流转换为数组
- `reduce()`：对流中的元素归约操作，将每个元素合起来形成一个新的值
- `collect()`：对流的汇总操作，比如输出成`List`集合
- `anyMatch()`：匹配流中的元素，类似的操作还有`allMatch()`和`noneMatch()`方法
- `findFirst()`：查找第一个元素，类似的还有`findAny()`方法
- `max()`：求最大值
- `min()`：求最小值
- `count()`：求总数





