`org.apache.commons.cli` 是一个用于解析命令行参数的 Java 库。它使得开发者可以轻松地为自己的 Java 程序添加健壮的命令行接口。本文档将介绍其核心用法和常见模式。

## 1. 核心概念：三步曲

使用 Commons CLI 的过程可以清晰地分为三个阶段：**定义**、**解析** 和 **查询**。

1.  **定义 (Definition)**: 使用 `Options` 类来定义你的程序接收哪些参数。每个参数由一个 `Option` 对象表示。
2.  **解析 (Parsing)**: 使用 `CommandLineParser` 接口（通常是 `DefaultParser` 实现类）来解析来自 `main` 方法的 `String[] args` 参数数组。
3.  **查询 (Interrogation)**: 使用解析后得到的 `CommandLine` 对象，查询哪些参数被用户传入，并获取它们的值。

## 2. 添加依赖

在项目中使用前，需要先添加依赖。

**Maven (`pom.xml`):**

```xml
<dependency>
    <groupId>commons-cli</groupId>
    <artifactId>commons-cli</artifactId>
    <version>1.6.0</version> <!-- 建议使用最新版本 -->
</dependency>
```

**Gradle (`build.gradle`):**

```groovy
implementation 'commons-cli:commons-cli:1.6.0' // 建议使用最新版本
```

## 3. 基础用法示例

下面是一个简单的示例，演示了如何处理两个基本选项：一个布尔标志 `-h` (帮助) 和一个带值的选项 `-f <file>` (指定文件)。

```java
import org.apache.commons.cli.*;

public class CliExample {
    public static void main(String[] args) {
        // 1. 定义阶段：创建 Options 对象
        Options options = new Options();
        options.addOption("h", "help", false, "Print this message"); // 布尔选项
        options.addOption("f", "file", true, "File to process");   // 带参数的选项

        // 2. 解析阶段：解析命令行参数
        CommandLineParser parser = new DefaultParser();
        CommandLine cmd;

        try {
            cmd = parser.parse(options, args);
        } catch (ParseException e) {
            System.err.println("Error parsing command line options: " + e.getMessage());
            return;
        }

        // 3. 查询阶段：根据参数执行相应操作
        if (cmd.hasOption("h")) {
            // 如果用户传入了 -h 或 --help，打印帮助信息
            HelpFormatter formatter = new HelpFormatter();
            formatter.printHelp("my-app", options);
        } else {
            // 处理 -f 或 --file 选项
            if (cmd.hasOption("f")) {
                String filePath = cmd.getOptionValue("f");
                System.out.println("Processing file: " + filePath);
            } else {
                System.out.println("No file specified. Use -f <file> to specify a file.");
            }
        }
    }
}
```

**命令行测试：**

```bash
# 运行不带参数
java CliExample
# 输出: No file specified. Use -f <file> to specify a file.

# 运行并指定文件
java CliExample -f /path/to/my/file.txt
# 输出: Processing file: /path/to/my/file.txt

# 运行并请求帮助
java CliExample -h
# 输出:
# usage: my-app
#  -f,--file <arg>   File to process
#  -h,--help         Print this message
```

## 4. 常用功能与模式

### 4.1. 使用 `Option.builder` (推荐)

`Option.builder` 提供了一种更具可读性的方式来创建 `Option` 对象。

```java
Options options = new Options();

// 等同于 new Option("h", "help", false, "Print help")
Option help = Option.builder("h")
    .longOpt("help")
    .desc("Print this message")
    .build();

// 等同于 new Option("p", "port", true, "Port number")
Option port = Option.builder("p")
    .longOpt("port")
    .hasArg() // 表明此选项需要一个参数
    .argName("port_number") // 参数名，用于帮助文档
    .desc("The port to connect to")
    .build();

options.addOption(help);
options.addOption(port);
```
*   `.hasArg()`: 表示选项需要一个参数。
*   `.hasArgs()`: 表示选项可以有多个参数。
*   `.required()`: 将选项设置为必填。

### 4.2. 必填选项 (Required Option)

如果某个选项必须由用户提供，可以将其设置为必填。如果用户未提供，`parse` 方法会抛出 `ParseException`。

```java
Option input = Option.builder("i")
    .longOpt("input")
    .hasArg()
    .required() // 设置为必填
    .desc("Input file path (required)")
    .build();

options.addOption(input);

// 如果运行时不提供 -i 或 --input，程序会报错。
```

### 4.3. 互斥选项组 (OptionGroup)

如果你希望一组选项中只有一个能被用户选择（例如，`--encrypt` 和 `--decrypt` 不能同时出现）。

```java
Options options = new Options();
OptionGroup group = new OptionGroup();

group.addOption(new Option("e", "encrypt", false, "Encrypt the file"));
group.addOption(new Option("d", "decrypt", false, "Decrypt the file"));

// 可以设置该组是否为必填
group.setRequired(true);

options.addOptionGroup(group);

// 命令行测试
// java MyApp -e  (成功)
// java MyApp -d  (成功)
// java MyApp -e -d (失败，抛出 ParseException)
```

### 4.4. 获取参数值并提供默认值

`getOptionValue` 方法有一个重载版本，允许你提供一个默认值。

```java
// 定义
Option mode = Option.builder("m")
    .longOpt("mode")
    .hasArg()
    .desc("Set the running mode (e.g., 'fast' or 'slow')")
    .build();
options.addOption(mode);

// 查询
String runningMode = cmd.getOptionValue("m", "normal"); // 如果用户没提供 -m, runningMode 的值会是 "normal"
System.out.println("Running in " + runningMode + " mode.");
```

### 4.5. 处理带类型的参数

`CommandLine` 返回的参数值总是 `String` 类型。你需要自己转换它。

```java
// 定义
Option port = Option.builder("p").longOpt("port").hasArg().desc("Port number").build();
options.addOption(port);

// 查询和转换
if (cmd.hasOption("p")) {
    try {
        String portStr = cmd.getOptionValue("p");
        int portNumber = Integer.parseInt(portStr);
        System.out.println("Using port: " + portNumber);
    } catch (NumberFormatException e) {
        System.err.println("Invalid port number specified.");
    }
}
```

## 5. 总结

Apache Commons CLI 通过简单的三步模型，极大地简化了 Java 命令行程序的开发。通过灵活运用 `Option.builder`、`OptionGroup` 和 `HelpFormatter`，你可以构建出功能强大且用户友好的命令行工具。

---