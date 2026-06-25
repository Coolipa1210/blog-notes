---
标题: 相对路径到底相对谁？IDE、命令行、jar 包运行目录一次讲清
创建时间: 2026-06-05
更新时间: 2026-06-05
已发布平台: CSDN
关联知识:
  - "[[013_第十二篇：IO 流（上）——File、字节流与字符流|Java IO 流基础]]"
  - "[[002_第一篇：Java 运行机制与程序结构|Java 运行机制与 classpath]]"
tags:
  - 知识库/杂项知识点
  - Java/基础
  - Java/IO
  - Java/运行机制
  - 概念/相对路径
  - 概念/工作目录
  - 概念/classpath
  - 概念/JAR
---

> [!info] 知识摘要
> **总结**
> - `new File("a.txt")`、`Path.of("a.txt")` 这类普通文件相对路径，相对的是当前进程工作目录，也就是 `System.getProperty("user.dir")`。
> - IDE、命令行、`java -jar` 的差异，本质上通常是工作目录不同；jar 所在目录、classpath 资源是另外两套问题。
>
> **相关知识点**
> - [[013_第十二篇：IO 流（上）——File、字节流与字符流|File 与 IO 流]]、[[002_第一篇：Java 运行机制与程序结构|classpath]]、`user.dir`、工作目录、JAR 所在目录、classpath 资源

---

## 一、先给结论：相对路径相对 `user.dir`

这些写法都是普通文件路径：

```java
new File("a.txt");
new FileInputStream("a.txt");
Path.of("a.txt");
Files.readString(Path.of("a.txt"));
```

它们默认不相对源码文件、不相对 `.class` 文件、不相对包目录，也不相对 jar 包所在目录。

它们相对的是当前 Java 进程的工作目录：

```java
System.getProperty("user.dir")
```

所以：

```java
Path.of("data", "a.txt")
```

真正含义是：

```text
user.dir/data/a.txt
```

如果 `user.dir` 是：

```text
D:\code\demo
```

那么 `Path.of("data", "a.txt")` 指向：

```text
D:\code\demo\data\a.txt
```

这就是整篇的核心。后面所有 IDE、命令行、jar 的差异，都只是这个核心在不同启动方式下的表现。

![Java 相对路径基准关系图](imgs/019_IO01_relative-path-bases.png)

---

## 二、IDE、命令行、jar：谁改变了工作目录

### 2.1 IDE 运行：看运行配置里的 Working directory

IDE 里相对路径经常“看起来相对项目根目录”，不是因为 Java 规定相对项目根目录，而是 IDE 的运行配置通常把工作目录设成了项目根目录或模块根目录。

如果运行配置里的工作目录是：

```text
D:\code\demo
```

那么：

```java
new File("a.txt")
```

指向：

```text
D:\code\demo\a.txt
```

如果你把 Working directory 改成：

```text
D:\tmp
```

同一段代码立刻变成：

```text
D:\tmp\a.txt
```

> [!warning] 常见误区：相对路径相对当前 Java 文件
>
> 正确理解：普通文件相对路径不关心当前代码文件在哪里，只关心当前进程工作目录。

### 2.2 命令行运行：看你从哪里执行 `java`

命令行更直接。

```text
cd /d D:\code\demo
java -cp out PathBaseDemo
```

此时 `user.dir` 通常是：

```text
D:\code\demo
```

如果你站在另一个目录启动：

```text
cd /d C:\Users\me
java -cp D:\code\demo\out PathBaseDemo
```

`.class` 仍然从 `D:\code\demo\out` 加载，但普通相对文件路径会从 `C:\Users\me` 出发。

这里要分清：

| 概念 | 负责什么 |
| --- | --- |
| `classpath` | JVM 从哪里找类和资源 |
| `user.dir` | 普通相对文件路径从哪里出发 |

它们经常同时出现，但不是一回事。

### 2.3 `java -jar`：不会自动相对 jar 所在目录

假设 jar 在：

```text
D:\apps\demo.jar
```

你站在用户目录执行：

```text
cd /d C:\Users\me
java -jar D:\apps\demo.jar
```

那么：

```java
new File("test.txt")
```

通常指向：

```text
C:\Users\me\test.txt
```

不是：

```text
D:\apps\test.txt
```

只有当你先进入 jar 所在目录：

```text
cd /d D:\apps
java -jar demo.jar
```

`test.txt` 才会落到 jar 旁边。原因不是 `java -jar` 特殊，而是此时 `user.dir` 刚好等于 jar 所在目录。

---

## 三、先打印再争论：最小排查代码

路径问题不要靠猜。先把工作目录和解析后的路径打出来。

下面这段适合快速排查，不是业务代码模板：

```java
import java.nio.file.Files;
import java.nio.file.InvalidPathException;
import java.nio.file.Path;

public class PathDebug {
    public static void main(String[] args) {
        String input = "data/a.txt";

        try {
            Path path = Path.of(input);

            System.out.println("user.dir = " + System.getProperty("user.dir"));
            System.out.println("input = " + path);
            System.out.println("absolute = " + path.toAbsolutePath().normalize());
            System.out.println("exists = " + Files.exists(path));
        } catch (InvalidPathException e) {
            System.out.println("bad path = " + input);
            System.out.println(e.getMessage());
        } catch (SecurityException e) {
            System.out.println("no permission to inspect path = " + input);
            System.out.println(e.getMessage());
        }
    }
}
```

`Files.exists(path)` 在普通本地开发里很方便，但它不是永远无风险：权限严格、SecurityManager 或特殊运行环境下可能抛 `SecurityException`。快速排查可以直接打，写成文档范例时最好把边界说清楚。

---

## 四、路径拼接：别把字符串当路径 API

最差写法：

```java
String path = "data" + "\\" + "a.txt";
```

稍微换皮但本质还是差：

```java
String path = "data" + File.separator + "a.txt";
```

`File.separator` 只是给你当前系统的分隔符，不会让字符串拼接突然变成路径建模。你仍然要自己处理多余分隔符、绝对路径片段、空路径、跨平台细节和可读性。

固定几段路径，直接写：

```java
Path path = Path.of("data", "a.txt");
```

已有基础目录，再拼子路径：

```java
Path base = Path.of("data");
Path path = base.resolve("a.txt");
```

多级子路径也不用手搓：

```java
Path path = Path.of("data", "2026", "a.txt");
```

判断规则很简单：

| 场景 | 推荐写法 |
| --- | --- |
| 几段固定路径 | `Path.of("data", "a.txt")` |
| 已有 `base`，追加子路径 | `base.resolve("a.txt")` |
| 需要父目录 | `path.getParent()` |
| 需要文件名 | `path.getFileName()` |

路径是结构化数据，不是拿 `+` 号串起来的装饰字符串。

---

## 五、绝对路径、规范路径、真实路径不是一回事

很多人打印：

```java
file.getAbsolutePath()
```

然后以为问题解决了。

其实 `getAbsolutePath()` 只是把相对路径接到当前工作目录后面，它不保证文件存在，也不处理真实文件系统里的符号链接。

几个常见方法要分清：

| 方法 | 作用 | 关键限制 |
| --- | --- | --- |
| `toAbsolutePath()` / `getAbsolutePath()` | 转成绝对形式 | 主要是路径字符串层面的解析 |
| `normalize()` | 消掉 `.`、`..` 这类片段 | 仍然不访问真实文件系统 |
| `toRealPath()` | 得到真实存在路径 | 会访问文件系统，文件不存在会抛异常，会处理符号链接 |
| `getCanonicalPath()` | `File` API 中的规范路径 | 会访问文件系统，可能抛 `IOException` |

如果只是打印日志，`toAbsolutePath().normalize()` 通常够用。

如果要做路径比较、安全检查、防止目录逃逸，别拿 `getAbsolutePath()` 糊弄自己。用 `Path.toRealPath()` 或 `File.getCanonicalPath()`，并且认真处理异常。否则 `..`、符号链接、大小写不敏感文件系统，迟早会把“看起来没问题”的路径判断打穿。

例如限制文件必须在某个根目录下，思路应该接近这样：

```java
Path root = Path.of("uploads").toRealPath();
Path target = root.resolve(userInput).normalize();
Path realTarget = target.toRealPath();

if (!realTarget.startsWith(root)) {
    throw new SecurityException("Path escapes upload directory");
}
```

真实业务里还要结合“文件是否允许不存在”“是否允许创建新文件”“是否可能被符号链接替换”等场景继续收紧。这里先记住底线：安全检查不要只看字符串形态。

---

## 六、`src/main/resources`：开发期能用，不代表运行期可依赖

这段路径在 IDE 里可能能跑：

```java
new File("src/main/resources/config.txt");
```

原因通常是 IDE 工作目录刚好是项目根目录。

但它不是可移植的运行时路径。打包发布后，用户可能只拿到：

```text
demo.jar
```

而不是你的整个源码目录。

更准确的说法是：

```text
开发期临时用源码目录直接路径可以，但不要把它当成可移植的运行时契约。
```

如果资源应该随程序发布，放进 `resources` 后用 classpath 读取：

```java
import java.io.InputStream;

try (InputStream in = App.class.getResourceAsStream("/config/default.properties")) {
    if (in == null) {
        throw new IllegalStateException("Resource not found");
    }

    // read from in
}
```

注意 `/config/default.properties` 不是文件系统绝对路径，而是从 classpath 根开始找资源。

`Class.getResource()` 的基本规则：

| 写法 | 相对谁 |
| --- | --- |
| `App.class.getResource("a.txt")` | `App` 所在包 |
| `App.class.getResource("/a.txt")` | classpath 根 |

资源读取和普通文件读取是两套规则：

| 目标 | 推荐方式 |
| --- | --- |
| 用户电脑上的外部文件 | `Path` / `Files` |
| jar 内置默认资源 | `getResourceAsStream()` |
| 运行时生成文件 | 写到外部目录，不要写进 jar 内部 |

---

## 七、如果确实要 jar 所在目录，显式获取

jar 所在目录不是普通相对路径的默认基准。如果你的需求就是“结果文件放在 jar 旁边”，要明确获取 jar 位置。

```java
import java.net.URISyntaxException;
import java.nio.file.Files;
import java.nio.file.Path;

public class App {
    static Path appDir() {
        try {
            Path codePath = Path.of(
                    App.class.getProtectionDomain()
                            .getCodeSource()
                            .getLocation()
                            .toURI()
            );

            return Files.isRegularFile(codePath) ? codePath.getParent() : codePath;
        } catch (URISyntaxException e) {
            throw new IllegalStateException("Cannot locate application directory", e);
        }
    }
}
```

注意两个坑：

- jar 运行时，`codePath` 通常是 `demo.jar` 文件本身，要取 `getParent()` 才是 jar 所在目录。
- IDE 运行时，`codePath` 可能是 `target/classes` 或 `out/production` 这类目录。

还有一个更现实的问题：jar 所在目录不一定可写。安装目录、系统目录、只读介质里都可能写失败。小工具可以这么设计，正式应用最好让输出目录可配置，或使用用户目录、配置目录、日志目录。

---

## 八、到底该写到哪里

相对路径的问题，最后经常不是语法问题，而是产品问题：这个文件应该跟着谁走？

| 需求 | 更合适的位置 |
| --- | --- |
| 命令行工具的本次输出 | 当前工作目录或参数指定目录 |
| 小型绿色工具的输出 | jar 所在目录，但要处理权限 |
| 用户配置 | 用户目录、应用配置目录或显式传入路径 |
| 程序默认模板 | classpath 资源，只读 |
| 临时文件 | 系统临时目录 |
| 日志 | 日志框架配置的目录 |

不要用一个含糊的 `"../test.txt"` 承担所有设计责任。

---

## 总结

### 速查表

| 问题 | 答案 |
| --- | --- |
| `new File("a.txt")` 相对谁 | `user.dir` |
| IDE 中为什么像是相对项目根目录 | IDE 的 Working directory 常设为项目根目录 |
| `java -jar` 是否相对 jar 所在目录 | 不会，仍然看 `user.dir` |
| `../test.txt` 相对谁往上一级 | 相对 `user.dir` |
| classpath 资源怎么读 | `getResourceAsStream()` |
| 固定几段路径怎么写 | `Path.of("data", "a.txt")` |
| 已有基础目录怎么追加 | `base.resolve("a.txt")` |
| 能不能用 `File.separator` 拼字符串 | 不推荐，还是手动拼字符串 |
| 路径安全检查看什么 | `toRealPath()` / `getCanonicalPath()`，不要只看绝对路径字符串 |

### 记忆口诀

```text
普通文件看 user.dir。
内置资源看 classpath。
jar 位置问 CodeSource。
路径拼接交给 Path。
安全判断看真实路径。
```

最后用一句话收束：

```text
Java 普通相对路径的基准不是源码、class、项目结构或 jar 文件，而是当前进程工作目录 user.dir；其他目录需求都应该显式表达。
```
