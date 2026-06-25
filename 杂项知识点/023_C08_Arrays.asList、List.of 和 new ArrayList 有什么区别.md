---
标题: "【Java杂项】Arrays.asList、List.of 和 new ArrayList 有什么区别？集合可变性避坑"
创建时间: 2026-06-09
更新时间: 2026-06-09
已发布平台: CSDN
关联知识:
  - "[[015_第十四篇：集合框架（上）——Collection、泛型与 List|Java 集合框架：Collection、泛型与 List]]"
tags:
  - 知识库/杂项知识点
  - Java/基础
  - Java/集合
  - Java/List
  - 概念/Arrays.asList
  - 概念/List.of
  - 概念/ArrayList
  - 概念/不可变集合
---

> [!info] 知识摘要
> **总结**
> - `Arrays.asList()` 返回的是数组的固定长度 List 视图，可以 `set()`，不能 `add()` / `remove()`，并且会和原数组互相影响。
> - `List.of()` 返回不可修改 List，不能 `set()` / `add()` / `remove()`，也不允许 `null` 元素。
> - `new ArrayList<>(...)` 会创建一个新的可变列表副本，适合后续需要增删改的场景。
>
> **相关知识点**
> - [[015_第十四篇：集合框架（上）——Collection、泛型与 List|List 接口]]、数组、集合工厂方法、固定长度视图、不可修改集合、集合拷贝

---

## 一、先给结论：这三个写法不是同一种 List

下面三段代码看起来都能得到一个 `List`：

```java
List<String> a = Arrays.asList("A", "B", "C");
List<String> b = List.of("A", "B", "C");
List<String> c = new ArrayList<>(List.of("A", "B", "C"));
```

但它们的可变性完全不同：

| 写法 | 能 `set()` 改元素吗 | 能 `add()` / `remove()` 改长度吗 | 是否允许 `null` | 是否和原数组联动 |
| --- | --- | --- | --- | --- |
| `Arrays.asList(array)` | 能 | 不能 | 能 | 是 |
| `List.of(...)` | 不能 | 不能 | 不能 | 否 |
| `new ArrayList<>(...)` | 能 | 能 | 取决于传入元素，`ArrayList` 本身允许 | 否 |

一句话记：

```text
Arrays.asList()：固定长度的数组包装。
List.of()：不可修改的 List。
new ArrayList<>()：真正可增删的 ArrayList 副本。
```

很多坑都来自把“不能改长度”“不能改内容”“拷贝了一份数据”混成了一件事。

---

## 二、`Arrays.asList()`：固定长度，不是完全不可变

`Arrays.asList()` 最容易让人误会，因为它返回的类型也是 `List`。

看这段代码：

```java
import java.util.Arrays;
import java.util.List;

public class Demo {
    public static void main(String[] args) {
        String[] arr = {"A", "B", "C"};
        List<String> list = Arrays.asList(arr);

        list.set(1, "X");
        System.out.println(list);          // [A, X, C]
        System.out.println(arr[1]);        // X

        list.add("D");                     // UnsupportedOperationException
    }
}
```

这里有两个关键点。

第一，`set()` 可以成功，因为它只是替换已有位置上的元素，没有改变列表长度。

第二，`add()` 会抛出 `UnsupportedOperationException`，因为 `Arrays.asList()` 返回的是一个固定长度的 List。它背后包着原数组，数组长度不能变，所以这个 List 的长度也不能变。

这也是为什么它和原数组会联动：

```java
String[] arr = {"A", "B", "C"};
List<String> list = Arrays.asList(arr);

arr[0] = "Z";
System.out.println(list); // [Z, B, C]

list.set(1, "Y");
System.out.println(Arrays.toString(arr)); // [Z, Y, C]
```

`Arrays.asList()` 没有复制一份新数据，它更像是在数组外面套了一层 List 接口。

所以，不要把它理解成：

```text
数组转换成了一个独立 ArrayList。
```

更准确的理解是：

```text
数组被包装成了一个固定长度的 List 视图。
```

---

## 三、`List.of()`：不可修改，不只是不能增删

`List.of()` 是 Java 9 引入的集合工厂方法，适合快速创建一组固定内容的列表。

它和 `Arrays.asList()` 的最大区别是：`List.of()` 返回的是不可修改 List。

```java
import java.util.List;

public class Demo {
    public static void main(String[] args) {
        List<String> list = List.of("A", "B", "C");

        list.set(0, "X");  // UnsupportedOperationException
        list.add("D");     // UnsupportedOperationException
        list.remove("A");  // UnsupportedOperationException
    }
}
```

注意这里不只是不能改变长度，连替换已有元素也不行。

它还有一个常被忽略的边界：`List.of()` 不允许 `null` 元素。

```java
List<String> list = List.of("A", null, "C"); // NullPointerException
```

这点和 `Arrays.asList()`、`ArrayList` 不一样：

```java
List<String> a = Arrays.asList("A", null, "C");  // 可以

List<String> b = new ArrayList<>();
b.add(null);                                     // 可以
```

所以 `List.of()` 更适合表达“这组数据创建后不应该被修改”，比如常量配置、测试数据、方法返回的只读结果。

如果后续代码需要增删改，不要直接用 `List.of()` 的返回值继续操作。

---

## 四、`new ArrayList<>(...)`：创建可变副本

如果你想要一个真正可以增删改的列表，应该显式创建 `ArrayList`：

```java
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

public class Demo {
    public static void main(String[] args) {
        String[] arr = {"A", "B", "C"};

        List<String> list = new ArrayList<>(Arrays.asList(arr));

        list.add("D");
        list.remove("B");
        list.set(0, "X");

        System.out.println(list);              // [X, C, D]
        System.out.println(Arrays.toString(arr)); // [A, B, C]
    }
}
```

这时 `list` 和 `arr` 已经不是同一个底层存储。

`new ArrayList<>(collection)` 会把传入集合里的元素复制到新的 `ArrayList` 中。复制之后：

| 操作 | 影响 |
| --- | --- |
| 修改新 `ArrayList` 的结构 | 不影响原数组或原集合 |
| 修改原数组元素 | 不影响新 `ArrayList` 中对应位置 |
| 对新 `ArrayList` 调用 `add()` / `remove()` | 正常支持 |

这也是为什么下面这个写法很常见：

```java
List<String> list = new ArrayList<>(List.of("A", "B", "C"));
```

含义很清楚：

```text
先用 List.of() 写出初始数据，再复制成一个可变 ArrayList。
```

---

## 五、常见坑：基本类型数组不能按预期展开

`Arrays.asList()` 还有一个经典坑：基本类型数组不会被拆成多个元素。

```java
int[] nums = {1, 2, 3};

List<int[]> list = Arrays.asList(nums);

System.out.println(list.size()); // 1
```

为什么不是 3？

因为泛型不能使用基本类型，`Arrays.asList(T... a)` 看到的是一个 `int[]` 对象，而不是三个 `Integer`。所以最终得到的是一个只有一个元素的 `List<int[]>`。

如果想得到 `[1, 2, 3]` 这样的 `List<Integer>`，可以这样写：

```java
List<Integer> list = Arrays.asList(1, 2, 3);
```

或者使用流：

```java
List<Integer> list = Arrays.stream(nums)
        .boxed()
        .toList();
```

不过要注意，`Stream.toList()` 返回的 List 也不应该当成可增删列表来用。如果后续要修改，仍然建议包一层新的 `ArrayList`：

```java
List<Integer> list = new ArrayList<>(
        Arrays.stream(nums).boxed().toList()
);
```

---

## 六、到底该用哪个

按使用场景选择就很简单：

| 场景 | 推荐写法 | 原因 |
| --- | --- | --- |
| 只是把已有数组临时当成 `List` 传给方法 | `Arrays.asList(array)` | 不复制数据，但要接受固定长度和数组联动 |
| 创建一组不希望被修改的固定数据 | `List.of(...)` | 意图清楚，不允许修改，也不允许 `null` |
| 后续要增删改 | `new ArrayList<>(...)` | 得到独立可变副本 |
| 从数组生成可变列表 | `new ArrayList<>(Arrays.asList(array))` | 先包装，再复制 |
| 初始值很少且需要后续修改 | `new ArrayList<>(List.of(...))` | 写法简洁，语义明确 |

工程里最稳的判断方式是先问自己一句：

```text
这个 List 后面还会不会增删改？
```

如果会，创建 `ArrayList`。

如果不会，并且不允许 `null`，用 `List.of()`。

如果只是为了适配一个需要 `List` 参数的方法，而且数据本来就在数组里，可以用 `Arrays.asList()`。

---

## 七、面试里怎么答更稳

如果面试问：`Arrays.asList()`、`List.of()` 和 `new ArrayList<>()` 有什么区别？

不要只背一段完美定义。更稳的答法是先抓住一个主轴：

```text
这三个写法的核心差别是：固定长度视图、不可修改集合、可变副本。
```

然后按行为展开：

- `Arrays.asList()` 是基于原数组的固定长度 List，不能 `add` 和 `remove`，但可以 `set`，修改元素会影响原数组。
- `List.of()` 是 Java 9 引入的不可修改 List，`set`、`add`、`remove` 都不允许，而且不允许 `null`。
- `new ArrayList<>(...)` 是创建新的 `ArrayList` 副本，列表结构可以变化，后续可以正常增删改。

最后再补一句工程判断：

```text
需要可变列表时，不要直接操作 Arrays.asList() 或 List.of() 的返回值，要显式 new ArrayList<>(...)。
```

如果想回答得更像真正写过代码，可以顺手补一个版本演进点：`List.of()` 是 Java 9 才有的。Java 8 以及更早版本里，如果想返回一个不希望外部修改的 List，常见写法是：

```java
List<String> list = Collections.unmodifiableList(
        new ArrayList<>(Arrays.asList("A", "B", "C"))
);
```

这也能顺便避开一个追问：`unmodifiableList()` 包的是一个不可修改视图，不是天然不可变对象；如果底层那个原始 List 还被别的引用改了，这个视图看到的内容也会跟着变。

---

## 八、总结

这三个 API 的区别，不是“哪个更高级”，而是它们表达的意图不同。

| API | 核心语义 | 最容易踩的坑 |
| --- | --- | --- |
| `Arrays.asList()` | 把数组包装成固定长度 List | 不能增删；和原数组联动；基本类型数组不会展开 |
| `List.of()` | 创建不可修改 List | 不能 `set()`；不能有 `null` |
| `new ArrayList<>(...)` | 创建可变副本 | 只是浅拷贝元素，不会深拷贝对象本身 |

最后一个细节也要记住：`new ArrayList<>(...)` 复制的是元素引用，不是把每个对象都克隆一份。

```java
List<Student> a = List.of(new Student("张三"));
List<Student> b = new ArrayList<>(a);
```

这里 `b` 是新的列表，但里面的 `Student` 对象仍然是同一个对象引用。如果元素对象本身可变，修改对象内容仍然可能被多个列表同时看到。
