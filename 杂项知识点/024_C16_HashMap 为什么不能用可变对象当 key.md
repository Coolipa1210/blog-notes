---
标题: "【Java杂项】HashMap 为什么不能用可变对象当 key？从 get 返回 null 到哈希查找机制"
创建时间: 2026-06-09
更新时间: 2026-06-10
已发布平台: CSDN
关联知识:
  - "[[017_第十六篇：集合框架（下）——Map、HashMap 与 TreeMap|Java 集合框架：Map、HashMap 与 TreeMap]]"
  - "[[016_M14_equals 与 hashCode 为什么必须一起重写|equals 与 hashCode 为什么必须一起重写]]"
  - "[[015_M15_final 修饰变量、方法、类|final 修饰变量、方法、类]]"
tags:
  - 知识库/杂项知识点
  - Java/基础
  - Java/集合
  - Java/HashMap
  - 概念/hashCode
  - 概念/equals
  - 概念/可变对象
  - 概念/不可变对象
---

> [!info] 知识摘要
> **总结**
> - `HashMap` 查找 key 时，会先用 key 当前的 `hashCode()` 计算桶位置，再在桶内用 `==` 或 `equals()` 确认具体 key。
> - 如果 key 放入 `HashMap` 后，参与 `equals()` / `hashCode()` 的字段被修改，新的哈希值可能指向另一个桶，`get()` 就可能返回 `null`。
> - 更危险的问题不是“同一个引用 get 不到”，而是 key 的相等性语义被污染：不同对象可能在修改后变成相等，但仍分布在不同桶里。
> - 如果 Map 已经被污染，`remove(key)` 不一定能清理旧节点；可靠处理方式是遍历 `entrySet()` 按引用或稳定业务标识清理，或者重建一个使用稳定 key 的新 Map。
> - 工程上更稳的做法是使用 `String`、`Integer`、`UUID`、`record`、Lombok `@Value` 这类不可变值对象，或把自定义 key 设计成 `final class` + 关键字段 `final`。
>
> **相关知识点**
> - [[017_第十六篇：集合框架（下）——Map、HashMap 与 TreeMap|HashMap]]、[[016_M14_equals 与 hashCode 为什么必须一起重写|equals 与 hashCode]]、哈希表、桶定位、不可变对象、`final`

---

## 一、先看现象：同一个 key，为什么突然 get 不到了

先用一段最小代码复现问题：

```java
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;

public class Demo {
    public static void main(String[] args) {
        Student s = new Student("张三", 18);

        Map<Student, String> map = new HashMap<>();
        map.put(s, "北京");

        System.out.println(map.get(s)); // 北京

        s.setAge(20);

        System.out.println(map.get(s)); // 可能是 null
    }
}

class Student {
    private final String name;
    private int age;

    Student(String name, int age) {
        this.name = name;
        this.age = age;
    }

    void setAge(int age) {
        this.age = age;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (!(o instanceof Student)) {
            return false;
        }
        Student other = (Student) o;
        return age == other.age && Objects.equals(name, other.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
}
```

直觉上很多人会觉得：`s` 是同一个对象引用，`HashMap` 里面存的 key 不也是这个对象吗？既然外面的 `s.age` 改了，Map 里面那个 key 的 `age` 也跟着改了，为什么 `map.get(s)` 还会找不到？

关键点在这里：

```text
HashMap 不是先把所有 key 都拿出来逐个 equals。
HashMap 会先根据 hashCode 定位桶，再只在这个桶里找 key。
```

所以问题不是“这个对象还在不在 Map 里”，而是：

```text
修改后的 key 还能不能带着 HashMap 找回原来那个桶。
```

如果找错桶，后面的 `==` 和 `equals()` 都没有机会执行到原来的节点。

---

## 二、HashMap 查找 key 的顺序

可以把 `HashMap` 的查找过程简化成三步：

```text
key.hashCode()
    -> 计算桶下标
    -> 在桶内用 == 或 equals() 找具体节点
```

更贴近 JDK 8 以后实现的过程是：

```java
int h = key.hashCode();
int hash = h ^ (h >>> 16);
int index = (table.length - 1) & hash;
```

这里不需要死背源码，只要抓住两个结论：

| 步骤 | 作用 |
| --- | --- |
| `hashCode()` | 提供 key 的哈希值基础 |
| 扰动计算 | 让高位信息也参与桶分布 |
| `(n - 1) & hash` | 根据数组长度算出桶下标 |
| 桶内比较 | 先看哈希值，再用 `==` 或 `equals()` 确认 key |

也就是说，`equals()` 并不是第一步。

`HashMap` 不会这样找：

```text
遍历整个 Map -> 对每个 key 调 equals()
```

而是这样找：

```text
先算桶 -> 只查这个桶 -> 桶内再比较 key
```

这正是可变 key 出问题的根源。

---

## 三、为什么修改 key 后可能返回 null

继续看前面的例子。

插入时，`Student("张三", 18)` 的 `hashCode()` 由 `name` 和 `age` 共同决定。

```java
Student s = new Student("张三", 18);
map.put(s, "北京");
```

假设插入时算出的桶位置是 5：

```text
Bucket 5
  -> key: Student{name="张三", age=18}, value: "北京"
```

注意，`HashMap` 节点里保存的是 key 引用，不是复制一个新的 `Student` 对象。所以当你执行：

```java
s.setAge(20);
```

Map 里面那个 key 对象看到的年龄也变成了 20：

```text
Bucket 5
  -> key: Student{name="张三", age=20}, value: "北京"
```

但节点还停留在原来的桶 5 里。

接着再查：

```java
map.get(s);
```

这次 `hashCode()` 按 `Student("张三", 20)` 重新计算，可能得到另一个桶，比如 12：

```text
HashMap 去 Bucket 12 找

Bucket 12
  -> 空
```

于是返回 `null`。

这里最容易误解的是：对象确实还在 `HashMap` 里，但 `HashMap` 是通过“当前哈希值”导航的。你改了参与哈希计算的字段，就相当于把导航地址改了，而数据还留在旧位置。

---

## 四、同一个对象引用为什么也救不了

`HashMap` 在桶内确认 key 时，确实会做引用比较。

可以把桶内比较理解成：

```java
if (node.hash == hash && (node.key == key || key.equals(node.key))) {
    return node.value;
}
```

但这个判断有一个前提：已经进入了正确的桶。

如果 `hashCode()` 改变后，`HashMap` 算出来的是另一个桶，那么它只会去新桶里找。旧桶里的节点不会被扫描到，`node.key == key` 也就没有执行机会。

所以这句话很重要：

```text
引用相同，只能在桶内比较阶段生效，不能跳过桶定位阶段。
```

这也是为什么下面两件事可以同时成立：

| 判断 | 结果 |
| --- | --- |
| `s` 和 Map 里保存的 key 是同一个对象 | 是 |
| `map.get(s)` 一定能找到 value | 否 |

`HashMap` 的高效查找依赖桶定位。它为了避免全表扫描，必须先相信 `hashCode()` 给出的方向。

---

## 五、更严重的问题：相等性语义已经被污染

可变 key 不只会导致 `get()` 返回 `null`。更工程化、也更隐蔽的问题是：Map 里的 key 状态已经被污染，`HashMap` 的内部桶分布和 key 当前的相等性语义不再一致。

先看同一个对象引用被重复插入的情况。

看这段代码：

```java
Student s = new Student("张三", 18);

Map<Student, String> map = new HashMap<>();
map.put(s, "北京");

s.setAge(20);

map.put(s, "上海");
```

第二次 `put()` 时，`HashMap` 会按修改后的 `hashCode()` 计算新桶。如果新桶里找不到相同 key，它就会创建一个新节点。

内部状态可能变成：

```text
Bucket 5
  -> key: Student{name="张三", age=20}, value: "北京"

Bucket 12
  -> key: Student{name="张三", age=20}, value: "上海"
```

这就很危险了：同一个对象引用，可能在 `HashMap` 的不同桶里出现两次。

表现出来的症状包括：

- `size()` 和业务直觉不一致。
- `get()` 查到的值取决于当前 key 状态。
- `remove()` 可能删不掉旧桶里的节点。
- 缓存、去重、权限映射这类逻辑出现难排查的数据不一致。

这类问题通常不是立刻报错，而是让集合进入一种“还能运行但语义已经坏了”的状态。

更麻烦的是，问题不要求你复用同一个对象引用。只要 key 的状态变化会改变跨对象相等性，不同对象也会把 Map 搞坏。

```java
Student s = new Student("张三", 18);
Student t = new Student("张三", 20);

Map<Student, String> map = new HashMap<>();
map.put(s, "北京");
map.put(t, "上海");

s.setAge(20);

System.out.println(s.equals(t)); // true
System.out.println(map.size());  // 仍然可能是 2
```

从业务语义看，`s` 和 `t` 现在已经是同一个 key 了；但从 `HashMap` 的内部结构看，它们可能仍在两个不同桶里：

```text
Bucket 5
  -> key: Student{name="张三", age=20}, value: "北京"

Bucket 12
  -> key: Student{name="张三", age=20}, value: "上海"
```

这时 `map.size()` 仍然是 2，因为 `HashMap` 不会在你修改对象字段时自动重新分布节点，也不会全表扫描去修复“现在变得相等”的 key。

后续查询也会变得很难推理：

- 用 `s` 查，可能按 `s` 当前哈希值进入某个桶。
- 用 `t` 查，可能进入另一个桶。
- 两个对象当前 `equals()` 为 `true`，但 Map 内部已经存在两个逻辑相等的 key。
- 后续 `put()`、`remove()`、`containsKey()` 都依赖当前哈希定位，行为会和业务直觉脱节。

所以这个坑的本质不是“同一个引用为什么找不到”，而是：

```text
HashMap 要求 key 的哈希身份和相等身份在集合生命周期内稳定。
```

一旦 key 的身份变化，集合内部结构不会跟着重排，Map 的语义就已经坏了。

---

## 六、是不是所有可变对象都不能当 key

严格说，不是“对象可变”本身一定会破坏 `HashMap`，而是：

```text
参与 equals() / hashCode() 的字段，在放入 HashMap 后不能变。
```

比如下面这个类，`remark` 可以变，但 `id` 不变，并且 `equals()` / `hashCode()` 只依赖 `id`：

```java
import java.util.Objects;

final class UserKey {
    private final long id;
    private String remark;

    UserKey(long id, String remark) {
        this.id = id;
        this.remark = remark;
    }

    void setRemark(String remark) {
        this.remark = remark;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (!(o instanceof UserKey)) {
            return false;
        }
        UserKey other = (UserKey) o;
        return id == other.id;
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}
```

这个 key 的 `remark` 变化不会影响桶定位，因为 `remark` 不参与相等判断和哈希计算。

但工程上仍然建议把 key 设计得更清楚：

| 写法 | 风险 |
| --- | --- |
| 关键字段可变，参与 `hashCode()` | 高风险，放入 Map 后修改会破坏查找 |
| 非关键字段可变，不参与 `hashCode()` | 可以工作，但要保证业务语义清楚 |
| 关键字段 `final`，对象整体接近不可变 | 最稳妥 |

如果一个类名字就叫 `UserKey`、`OrderKey`、`CacheKey`，读代码的人天然会期待它作为 key 是稳定的。

---

## 七、自定义 key 的推荐写法：让类型本身防呆

如果确实要用自定义对象作为 `HashMap` 的 key，建议遵守四条规则。

第一，`equals()` 和 `hashCode()` 必须一起重写。

```text
如果 a.equals(b) == true，那么 a.hashCode() 必须等于 b.hashCode()。
```

第二，key 类型本身尽量禁止继承。

如果类允许被继承，子类可以重写 `equals()` / `hashCode()`，也可以引入新的可变字段参与哈希计算。这样父类里看起来稳定的 key 语义，到了子类里仍然可能被破坏。

所以真正用来做 key 的值对象，优先写成：

```text
final class + private final 字段 + 不提供修改关键字段的方法
```

第三，参与 `equals()` / `hashCode()` 的字段尽量用 `final`。

```java
import java.util.Objects;

final class EmployeeKey {
    private final long id;
    private final String companyCode;

    EmployeeKey(long id, String companyCode) {
        this.id = id;
        this.companyCode = companyCode;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (!(o instanceof EmployeeKey)) {
            return false;
        }
        EmployeeKey other = (EmployeeKey) o;
        return id == other.id && Objects.equals(companyCode, other.companyCode);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, companyCode);
    }
}
```

第四，不要给关键字段提供 setter。

这不是为了语法好看，而是为了保证对象一旦作为 key 放进 `HashMap`，后续不会因为状态变化导致桶位置变化。

Java 16 以后，`record` 很适合表达这种 key：

```java
record EmployeeKey(long id, String companyCode) {
}
```

`record` 的字段天然是 `private final`，生成的 `equals()` / `hashCode()` 基于所有组件，类也不能被普通类继承。它非常适合“多个字段共同组成一个稳定 key”的场景。

如果不想让所有字段都参与相等判断，就不要把它们放进 key 的 `record` 里。比如显示名称、备注、状态这类会变化的信息，应该放在 value 对象里，而不是混进 key 对象。

常见安全 key：

| key 类型 | 为什么安全 |
| --- | --- |
| `String` | 不可变，哈希语义稳定 |
| `Integer` / `Long` | 包装类不可变 |
| `UUID` | 值对象语义稳定 |
| `record` key | 组件不可变，自动生成稳定的 `equals()` / `hashCode()` |
| 自定义不可变 key | `final class`，关键字段 `final`，不提供修改入口 |

---

## 八、Lombok 的 @Data 为什么容易埋坑

如果项目里用 Lombok，要特别小心下面这种写法：

```java
import lombok.Data;

@Data
class Employee {
    private long id;
    private String name;
}
```

`@Data` 通常会生成 getter、setter、`equals()`、`hashCode()`、`toString()` 等方法。问题在于：默认生成的 `equals()` / `hashCode()` 可能把多个字段都纳入计算，同时 setter 又允许这些字段被修改。

于是就容易出现：

```java
Employee e = new Employee();
e.setId(1L);
e.setName("Alice");

Map<Employee, String> map = new HashMap<>();
map.put(e, "admin");

e.setName("Bob");

System.out.println(map.get(e)); // 可能是 null
```

更稳的做法不是只把 `@Data` 换成 `@Getter`，而是把 key 明确建模成不可变值对象。

如果所有字段都属于 key 的身份，优先用 Lombok 的 `@Value`：

```java
import lombok.Value;

@Value
class EmployeeKey {
    long id;
    String companyCode;
}
```

`@Value` 会把类做成不可变值对象的形态：类默认是 `final`，字段默认是 `private final`，不会生成 setter，并会生成基于字段的 `equals()` / `hashCode()`。

如果只有部分字段属于 key 身份，可以明确指定字段，但仍要避免把一个可继续演化的业务实体直接当 key：

```java
import lombok.EqualsAndHashCode;
import lombok.Getter;

@Getter
@EqualsAndHashCode(of = "id")
final class EmployeeKey {
    private final long id;

    EmployeeKey(long id) {
        this.id = id;
    }
}
```

注意这里我没有把 `name` 放进 key 类型。因为一个类如果叫 `EmployeeKey`，最好只承载身份字段。展示名、备注、部门名称这类会变化的信息，应该放在 value 或业务实体里。

Lombok 可以减少样板代码，但不能替你决定对象相等语义。用于 key 的类，重点不是“少写代码”，而是让后续开发者很难把它改坏。

---

## 九、StringBuilder 适合当 HashMap key 吗

`StringBuilder` 也是一个常见反例。

它是可变对象，而且没有像 `String` 那样按内容重写 `equals()` / `hashCode()`，默认继承的是 `Object` 的引用相等语义。

这意味着：

```java
StringBuilder a = new StringBuilder("abc");
StringBuilder b = new StringBuilder("abc");

System.out.println(a.equals(b)); // false
```

如果把 `StringBuilder` 当 key，它和前面 `Student` 的问题不是同一种机制。

`StringBuilder` 默认继承 `Object` 的 `equals()` / `hashCode()`，所以默认哈希身份通常跟对象标识绑定，而不是跟内部字符内容绑定。追加内容后，默认 `hashCode()` 不会因为字符内容改变而重新按内容计算。

因此下面这段代码通常还能查到：

```java
Map<StringBuilder, String> map = new HashMap<>();

StringBuilder key = new StringBuilder("user:1");
map.put(key, "Alice");

key.append(":profile");

System.out.println(map.get(key)); // 通常还能查到，但 key 的业务含义已经变了
```

这里的坑不是“桶定位一定失效”，而是“key 的业务含义已经变了”。它属于逻辑语义错误，不是典型的哈希定位失效。

更具体地说：

| key 类型 | 变化后主要问题 |
| --- | --- |
| 自定义 `Student`，字段参与 `hashCode()` | 哈希定位可能失效，`get()` / `remove()` 找错桶 |
| `StringBuilder` 默认作为 key | 哈希定位通常不因内容变化失效，但业务语义会漂移 |

如果业务上想用字符串内容当 key，应该转成不可变的 `String`：

```java
String key = builder.toString();
map.put(key, value);
```

不要把一个会继续变化的构造器对象直接丢进 `HashMap` 当 key。

---

## 十、Map 已经被污染了怎么办

如果 key 已经被错误修改，不要指望再随便 `remove(key)` 就一定能清理掉旧节点。因为 `remove()` 也要先按当前 key 的哈希值定位桶，定位错了同样删不掉。

可靠的临时清理方式是遍历 `entrySet()`。遍历会沿着 `HashMap` 内部已有节点走，不依赖你传入 key 的当前哈希值重新定位桶。

如果你要按对象引用清理某个已经变坏的 key：

```java
Iterator<Map.Entry<Student, String>> it = map.entrySet().iterator();

while (it.hasNext()) {
    Map.Entry<Student, String> entry = it.next();
    if (entry.getKey() == target) {
        it.remove();
    }
}
```

如果你要按稳定业务标识清理，例如学生 ID，下面假设 `Student` 有一个不会变化的 `getId()`：

```java
Iterator<Map.Entry<Student, String>> it = map.entrySet().iterator();

while (it.hasNext()) {
    Map.Entry<Student, String> entry = it.next();
    if (entry.getKey().getId() == targetId) {
        it.remove();
    }
}
```

但这只能算事故处理，不是长期方案。

更彻底的处理方式是重建一个干净 Map。下面假设 `StudentKey` 是只基于稳定 ID 的不可变 key：

```java
Map<StudentKey, String> fixed = new HashMap<>();

for (Map.Entry<Student, String> entry : broken.entrySet()) {
    Student student = entry.getKey();
    StudentKey key = new StudentKey(student.getId());
    fixed.put(key, entry.getValue());
}
```

重建时要注意两件事：

- 新 key 必须使用稳定身份字段。
- 如果旧 Map 里已经存在逻辑重复 key，要明确保留规则，比如保留最新值、保留最早值、记录冲突后人工处理。

线上缓存如果已经被污染，通常不要只做单点 `remove()`。更稳的是暂停继续写入、切换到稳定 key、重建缓存或重启加载数据。

---

## 十一、工程里怎么避坑

可以按下面这个清单判断：

| 问题 | 建议 |
| --- | --- |
| key 放进 Map 后还会修改吗 | 如果会，不要让修改字段参与 `equals()` / `hashCode()` |
| 这个类只是为了当 key 吗 | 设计成 `final class`、`record` 或 Lombok `@Value` |
| 能不能直接用已有值类型 | 优先用 `String`、`Long`、`UUID` 等不可变类型 |
| Lombok 是否生成了过宽的 `equals()` / `hashCode()` | 避免实体类 `@Data` 直接当 key；优先单独建 `@Value` key |
| `get()` 返回 `null` 是否一定代表 key 不存在 | 不一定，也可能 value 本身就是 `null`，或 key 状态已经变坏；可配合 `containsKey()` 判断 |

如果你已经发现线上出现这种问题，通常需要：

- 停止继续修改 key 的关键字段。
- 新建稳定 key 类型。
- 重建相关 `HashMap` 或缓存数据。
- 排查是否有逻辑重复 key 和无法删除的旧节点。

---

## 十二、面试里怎么答

如果面试问：`HashMap` 为什么不能用可变对象当 key？

可以这样答：

```text
不是所有可变对象绝对不能当 key，真正危险的是：对象放入 HashMap 后，参与 equals 和 hashCode 的字段发生变化。

HashMap 查找 key 时，会先根据 key 当前的 hashCode 计算桶位置，再在桶内通过 == 或 equals 确认具体 key。插入时按旧 hash 放在旧桶，修改字段后 hashCode 变了，get 时就可能去新桶查找。即使传入的是同一个对象引用，也可能因为第一步桶定位错了，根本比较不到旧桶里的节点，所以返回 null。

工程上应该优先使用 String、Integer、UUID、record 或不可变值对象，或把自定义 key 设计成 final class，关键字段 final，并且不要提供会修改这些字段的 setter。
```

如果想补充得更完整，可以再加一句：

```text
可变 key 还可能导致 containsKey 失败、remove 失败，甚至同一个对象引用被 put 到多个桶中，形成逻辑重复 key。
```

如果面试官继续追问“已经坏了怎么办”，可以接着说：

```text
remove(key) 也依赖当前 hashCode 定位桶，所以不一定能删掉旧节点。临时处理可以遍历 entrySet，用迭代器 remove 按引用或稳定业务 ID 清理；更彻底的办法是用稳定 key 重建一个新的 Map，并显式处理旧数据里的逻辑重复 key。
```

这个补充能把回答从“背规则”推进到“知道怎么处理事故”。

---

## 十三、总结

`HashMap` 的 key 必须保持稳定，本质上是因为 `HashMap` 的高效查找依赖哈希定位。

| 误区 | 正确理解 |
| --- | --- |
| 同一个对象引用一定能 get 到 | 不一定，必须先定位到正确桶 |
| `equals()` 能解决所有 key 查找问题 | `equals()` 只在桶内比较阶段生效 |
| 可变对象一定不能当 key | 关键是参与 `equals()` / `hashCode()` 的字段不能变 |
| 同一个引用找不到才是最大问题 | 更大的问题是跨对象相等性污染和逻辑重复 key |
| `@Data` 生成方法就安全 | 实体类 `@Data` 不适合直接当 key，优先建不可变 key 类型 |
| `remove(key)` 一定能修复坏数据 | 不一定，必要时遍历清理或重建 Map |
| `get()` 返回 `null` 就说明没有这个 key | 还要考虑 value 为 `null`、key 状态变化等情况 |

最后记住一句话：

```text
对象一旦作为 HashMap 的 key，它的哈希身份就应该稳定。
```

最稳妥的设计不是“放进去以后提醒大家别改”，而是让关键字段从类型设计上就改不了。
