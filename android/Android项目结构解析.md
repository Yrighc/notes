# Android 项目结构解析

#Android #Android开发 #项目结构 #JetpackCompose #Kotlin

## 项目概述

这是一个使用 [[Jetpack Compose]] 和 [[Kotlin]] 构建的现代 Android 应用项目。项目名称：`TestAndroid`，包名：`top.atych.testandroid`。

**技术栈：**
- [[Kotlin]] 2.0.21
- [[Android Gradle Plugin]] 8.13.2
- [[Jetpack Compose]] (Material 3)
- 最低支持 Android 7.0 (API 24)
- 目标 Android 14 (API 36)

---

## 项目目录结构

### 根目录结构

```
TestAndroid/
├── app/                    # 应用模块（主模块）
├── build.gradle.kts        # 项目级构建配置
├── settings.gradle.kts     # 项目设置和模块声明
├── gradle.properties       # Gradle 全局配置
├── gradle/                 # Gradle 相关文件
│   ├── libs.versions.toml  # 依赖版本管理
│   └── wrapper/           # Gradle Wrapper
├── gradlew                 # Gradle Wrapper 脚本 (Unix)
└── gradlew.bat            # Gradle Wrapper 脚本 (Windows)
```

---

## 核心文件详解

### 1. `settings.gradle.kts` - 项目设置文件

**作用：** 定义项目名称和包含的模块

```kotlin
rootProject.name = "TestAndroid"
include(":app")
```

**关键配置：**
- `pluginManagement`: 插件仓库配置（Google、Maven Central）
- `dependencyResolutionManagement`: 依赖仓库配置
- `include(":app")`: 声明包含 `app` 模块

---

### 2. `build.gradle.kts` (根目录) - 项目级构建配置

**作用：** 配置所有子模块共享的插件

```kotlin
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.kotlin.android) apply false
    alias(libs.plugins.kotlin.compose) apply false
}
```

**说明：**
- `apply false`: 不在根项目应用，只在子模块中应用
- 使用版本目录 (`libs.versions.toml`) 统一管理插件版本

---

### 3. `app/build.gradle.kts` - 应用模块构建配置

**作用：** 配置应用的具体构建参数和依赖

#### 关键配置解析

**命名空间和 SDK 版本：**
```kotlin
namespace = "top.atych.testandroid"
compileSdk { version = release(36) }
minSdk = 24
targetSdk = 36
```

**应用信息：**
```kotlin
applicationId = "top.atych.testandroid"  // 应用唯一标识
versionCode = 1                          // 内部版本号
versionName = "1.0"                      // 用户可见版本号
```

**构建类型：**
```kotlin
buildTypes {
    release {
        isMinifyEnabled = false  // 是否启用代码混淆
        proguardFiles(...)       // ProGuard 规则文件
    }
}
```

**编译选项：**
```kotlin
compileOptions {
    sourceCompatibility = JavaVersion.VERSION_11
    targetCompatibility = JavaVersion.VERSION_11
}
kotlinOptions {
    jvmTarget = "11"
}
```

**Compose 支持：**
```kotlin
buildFeatures {
    compose = true  // 启用 Jetpack Compose
}
```

**主要依赖：**
- `androidx.core:core-ktx`: Kotlin 扩展库
- `androidx.lifecycle:lifecycle-runtime-ktx`: 生命周期管理
- `androidx.activity:activity-compose`: Compose Activity 支持
- `androidx.compose.*`: Compose UI 库
- `androidx.compose.material3`: Material 3 设计系统

---

### 4. `gradle/libs.versions.toml` - 版本目录

**作用：** 统一管理所有依赖和插件的版本

**结构：**
- `[versions]`: 定义版本号
- `[libraries]`: 定义库依赖
- `[plugins]`: 定义插件

**优势：**
- 版本集中管理，避免冲突
- 便于维护和升级
- 类型安全（IDE 自动补全）

**示例：**
```toml
[versions]
kotlin = "2.0.21"
composeBom = "2024.09.00"

[libraries]
androidx-compose-ui = { group = "androidx.compose.ui", name = "ui" }

[plugins]
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
```

---

### 5. `gradle.properties` - Gradle 全局配置

**关键配置：**

```properties
# JVM 参数
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8

# AndroidX 支持
android.useAndroidX=true

# Kotlin 代码风格
kotlin.code.style=official

# R 类命名空间
android.nonTransitiveRClass=true
```

**说明：**
- `android.useAndroidX=true`: 使用 AndroidX 库（而非旧的 Support Library）
- `android.nonTransitiveRClass=true`: 启用 R 类命名空间，减少 APK 大小

---

### 6. `AndroidManifest.xml` - 应用清单文件

**位置：** `app/src/main/AndroidManifest.xml`

**作用：** 声明应用的组件、权限和配置

**关键内容：**

```xml
<application
    android:allowBackup="true"              # 允许备份
    android:icon="@mipmap/ic_launcher"      # 应用图标
    android:label="@string/app_name"        # 应用名称
    android:theme="@style/Theme.TestAndroid"> # 应用主题
    
    <activity
        android:name=".MainActivity"        # 主 Activity
        android:exported="true"             # 是否可被其他应用启动
        android:label="@string/app_name">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />
            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>
</application>
```

**说明：**
- `intent-filter` 中的 `MAIN` 和 `LAUNCHER` 表示这是应用的入口 Activity
- `android:exported="true"` 表示该 Activity 可以被其他应用启动

---

## 源代码结构

### `app/src/` 目录结构

```
src/
├── main/              # 主源代码
│   ├── java/         # Kotlin/Java 源代码
│   ├── res/          # 资源文件
│   └── AndroidManifest.xml
├── test/             # 单元测试
└── androidTest/      # 仪器测试（UI 测试）
```

---

### 源代码文件解析

#### 1. `MainActivity.kt` - 主 Activity

**作用：** 应用的入口 Activity，使用 [[Jetpack Compose]] 构建 UI

**关键代码：**

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()  // 启用边到边显示
        setContent {         // 设置 Compose 内容
            TestAndroidTheme {
                Scaffold(modifier = Modifier.fillMaxSize()) { innerPadding ->
                    Greeting(
                        name = "Android",
                        modifier = Modifier.padding(innerPadding)
                    )
                }
            }
        }
    }
}
```

**说明：**
- `ComponentActivity`: 支持 Compose 的 Activity
- `enableEdgeToEdge()`: 启用沉浸式状态栏和导航栏
- `setContent {}`: 设置 Compose UI 内容
- `Scaffold`: Material 3 的基础布局组件

**Composable 函数：**
```kotlin
@Composable
fun Greeting(name: String, modifier: Modifier = Modifier) {
    Text(
        text = "Hello $name!",
        modifier = modifier
    )
}
```

---

#### 2. `ui/theme/Color.kt` - 颜色定义

**作用：** 定义应用的主题颜色

```kotlin
val Purple80 = Color(0xFFD0BCFF)    // 深色主题主色
val PurpleGrey80 = Color(0xFFCCC2DC)
val Pink80 = Color(0xFFEFB8C8)

val Purple40 = Color(0xFF6650a4)   // 浅色主题主色
val PurpleGrey40 = Color(0xFF625b71)
val Pink40 = Color(0xFF7D5260)
```

**说明：**
- 使用十六进制颜色值（ARGB 格式）
- 80 后缀通常用于深色主题，40 用于浅色主题

---

#### 3. `ui/theme/Theme.kt` - 主题配置

**作用：** 配置 Material 3 主题，支持深色模式和动态颜色

**关键功能：**
- 自动检测系统深色模式
- 支持 Android 12+ 的动态颜色（Material You）
- 定义浅色和深色颜色方案

```kotlin
@Composable
fun TestAndroidTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,  // 是否启用动态颜色
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            // Android 12+ 使用动态颜色
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context) 
            else dynamicLightColorScheme(context)
        }
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography,
        content = content
    )
}
```

---

#### 4. `ui/theme/Type.kt` - 字体排版

**作用：** 定义应用的字体样式

```kotlin
val Typography = Typography(
    bodyLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 16.sp,
        lineHeight = 24.sp,
        letterSpacing = 0.5.sp
    )
)
```

---

## 资源文件结构

### `app/src/main/res/` 目录

```
res/
├── drawable/          # 可绘制资源（图标、背景等）
├── mipmap-*/         # 应用图标（不同密度）
├── values/           # 值资源
│   ├── colors.xml    # 颜色定义
│   ├── strings.xml   # 字符串资源
│   └── themes.xml    # 主题定义
└── xml/              # XML 配置文件
    ├── backup_rules.xml
    └── data_extraction_rules.xml
```

---

### 资源文件解析

#### `values/strings.xml` - 字符串资源

```xml
<resources>
    <string name="app_name">TestAndroid</string>
</resources>
```

**使用方式：**
- 在代码中：`R.string.app_name`
- 在 XML 中：`@string/app_name`

**优势：**
- 支持多语言国际化
- 统一管理字符串，便于维护

---

#### `values/themes.xml` - 主题定义

```xml
<style name="Theme.TestAndroid" 
       parent="android:Theme.Material.Light.NoActionBar" />
```

**说明：**
- 继承 Material 设计系统的浅色主题
- `NoActionBar` 表示不使用传统的 ActionBar（因为使用 Compose）

---

#### `mipmap-*/` - 应用图标

**不同密度目录：**
- `mipmap-mdpi/`: 中等密度（160 dpi）
- `mipmap-hdpi/`: 高密度（240 dpi）
- `mipmap-xhdpi/`: 超高密度（320 dpi）
- `mipmap-xxhdpi/`: 超超高密度（480 dpi）
- `mipmap-xxxhdpi/`: 超超超高密度（640 dpi）
- `mipmap-anydpi-v26/`: Android 8.0+ 自适应图标

**说明：**
- 系统会根据设备屏幕密度自动选择合适的图标
- 提供不同尺寸的图标可以确保在所有设备上显示清晰

---

## 构建输出目录

### `app/build/` 目录

**作用：** 存储构建过程中生成的中间文件和最终输出

**重要子目录：**
- `generated/`: 自动生成的资源文件
- `intermediates/`: 编译中间文件
- `outputs/`: 最终输出（APK、AAB 等）
- `tmp/`: 临时文件

**说明：**
- 此目录由 Gradle 自动生成，不应手动编辑
- 可以安全地删除，下次构建时会重新生成

---

## 测试目录

### `app/src/test/` - 单元测试

**作用：** 存放本地单元测试（运行在 JVM 上，不需要 Android 设备）

### `app/src/androidTest/` - 仪器测试

**作用：** 存放 Android 仪器测试（需要 Android 设备或模拟器）

**测试框架：**
- JUnit 4: 单元测试框架
- Espresso: UI 测试框架
- Compose UI Test: Compose UI 测试

---

## 配置文件

### `app/proguard-rules.pro` - ProGuard 规则

**作用：** 定义代码混淆和优化规则

**使用场景：**
- Release 构建时启用代码混淆
- 保护代码不被反编译
- 减小 APK 大小

**当前状态：** 仅包含注释示例，未启用混淆（`isMinifyEnabled = false`）

---

## 关键概念

### [[Jetpack Compose]]

现代 Android UI 工具包，使用声明式编程方式构建 UI。

**特点：**
- 声明式 UI
- 函数式编程
- 实时预览
- 性能优化

### [[Material 3]]

Google 最新的设计系统，提供现代化的 UI 组件和主题。

**特性：**
- 动态颜色（Material You）
- 深色模式支持
- 可访问性增强

### [[Kotlin]]

Android 官方推荐的编程语言。

**优势：**
- 简洁的语法
- 空安全
- 协程支持
- 与 Java 互操作

---

## 开发工作流

### 1. 添加新功能

1. 在 `app/src/main/java/` 中创建新的 Kotlin 文件
2. 使用 Compose 构建 UI
3. 在 `AndroidManifest.xml` 中注册新的 Activity（如需要）

### 2. 添加依赖

1. 在 `gradle/libs.versions.toml` 中定义版本
2. 在 `app/build.gradle.kts` 的 `dependencies` 中添加依赖

### 3. 添加资源

1. 在 `app/src/main/res/` 相应目录中添加资源
2. 在代码中使用 `R.资源类型.资源名` 引用

### 4. 构建和运行

```bash
# 构建 Debug APK
./gradlew assembleDebug

# 安装到设备
./gradlew installDebug

# 运行测试
./gradlew test
```

---

## 常见问题

### Q: 为什么使用 Compose 而不是传统的 XML 布局？

**A:** Compose 提供更简洁的代码、更好的性能和开发体验。

### Q: `libs.versions.toml` 的作用是什么？

**A:** 统一管理依赖版本，避免版本冲突，便于维护。

### Q: `namespace` 和 `applicationId` 的区别？

**A:** 
- `namespace`: 用于生成 R 类和构建配置
- `applicationId`: 应用的唯一标识，用于应用商店分发

### Q: 如何添加新的 Activity？

**A:** 
1. 创建新的 Activity 类
2. 在 `AndroidManifest.xml` 中注册
3. 如需作为启动 Activity，添加相应的 `intent-filter`

---

## 相关链接

- [[Android 开发文档]]
- [[Jetpack Compose 官方文档]]
- [[Material 3 设计指南]]
- [[Kotlin 官方文档]]

---

## 总结

这是一个标准的现代 Android 项目结构，使用了：
- ✅ Kotlin 作为主要语言
- ✅ Jetpack Compose 构建 UI
- ✅ Material 3 设计系统
- ✅ Gradle 版本目录管理依赖
- ✅ 模块化项目结构

**下一步学习建议：**
1. 深入学习 [[Jetpack Compose]] 的组件和布局
2. 了解 Android 生命周期和状态管理
3. 学习 Material 3 设计规范
4. 掌握 Android 导航和路由

---

*最后更新：2024年*
