  

## 一、项目整体架构

  

dpt-shell 是一个 Android DEX 函数抽取壳，采用"编译时抽取 + 运行时填回"的机制。

  

### 模块划分

- **dpt 模块**：Java 命令行工具，负责处理 APK/AAB，生成加壳后的安装包

- **shell 模块**：Android 应用模块（Java + C++），负责运行时保护和解密

  

---

  

## 二、核心流程

  

### 2.1 编译时处理流程（dpt 模块）

  

```

输入 APK/AAB

↓

1. 解压 APK，提取所有 DEX 文件

↓

2. 遍历每个 DEX 文件的所有类和方法

↓

3. 提取方法体（CodeItem）：

- 读取方法的 insns（指令数组）

- 保存原始字节码

- 将方法体替换为 return 语句（或随机指令混淆）

↓

4. 将所有 CodeItem 加密存储到 zip 文件

↓

5. 修改 AndroidManifest.xml：

- 备份原 Application 类名

- 替换为 ProxyApplication

↓

6. 集成 shell 模块：

- 将 shell 的 DEX 和 SO 文件添加到 APK

- 将 CodeItem 文件添加到 assets

↓

7. 生成垃圾代码 DEX（可选）

↓

8. 打包、对齐、签名

↓

输出加壳 APK

```

  

### 2.2 运行时执行流程（shell 模块）

  

```

App 启动

↓

1. ProxyApplication.attachBaseContext()

- 解压 shell SO 文件到 dataDir

- 加载 shell SO（System.load）

- 调用 JniBridge.ia() 初始化

↓

2. SO 加载时（.init_array）

- init_dpt() 执行：

* 解密 .bitcode 段（如果启用）

* dpt_hook() 进行 Hook

* createAntiRiskProcess() 创建反调试进程

↓

3. Hook 关键函数：

- hook_mmap()：使 DEX 内存可写

- hook_DefineClass()：拦截类加载

- hook_execve()：阻止 dex2oat

↓

4. JniBridge.ia() 执行：

- 从 APK 中读取 CodeItem 文件

- 解析并加载到内存（dexMap）

- 提取 DEX 文件到 dataDir

↓

5. combineDexElements()：

- 将提取的 DEX 合并到 ClassLoader

- 优先从壳的 DEX 查找类

↓

6. 类加载时（DefineClass Hook）：

- 检测到类被加载

- 遍历类的所有方法

- 从 dexMap 查找对应的 CodeItem

- 将 CodeItem 写回方法体位置

↓

7. ProxyApplication.onCreate()：

- 调用原 Application.onCreate()

- 应用正常启动

```

  

---

  

## 2.3 DEX 处理详细流程

  

在编译时处理阶段，DEX 文件会经过以下 7 个关键步骤的处理：

  

### 步骤 1：extractDexCode - 提取 DEX 代码到资源目录

  

**实现位置**：`dpt/src/main/java/com/luoye/dpt/builder/AndroidPackage.java::extractDexCode()`

  

**处理过程**：

```java

// 1. 遍历所有 DEX 文件（classes.dex, classes2.dex...）

List<File> dexFiles = getDexFiles(getDexDir(packageDir));

  

// 2. 如果需要保留部分类，先分割 DEX

if (isKeepClasses()) {

// 将 DEX 分为两部分：

// - keepDex: 保留在 APK 中的类（匹配排除规则）

// - splitDex: 需要保护的类（抽取代码）

DexUtils.splitDex(dexFile, keepDex, splitDex);

}

  

// 3. 提取所有方法的 CodeItem（字节码）

List<Instruction> ret = DexUtils.extractAllMethods(

dexFile, extractedDexFile, packageName, dumpCode, obfuscate);

  

// 4. 将提取的代码保存到 assets 目录

// 文件：assets/classes.dex.dat（MultiDexCode 格式）

MultiDexCodeUtils.writeMultiDexCode(dataOutputPath, multiDexCode);

```

  

**目的**：

- **抽取方法字节码**：将方法体替换为 return 语句或随机指令

- **保存到 assets**：运行时从 assets 读取并填回

- **隐藏真实代码**：APK 中不再包含原始字节码

  

**结果**：

- 原 DEX 文件：方法体被替换（空壳）

- `assets/classes.dex.dat`：包含所有 CodeItem 数据

  

---

  

### 步骤 2：addJunkCodeDex - 添加垃圾代码 DEX

  

**实现位置**：`dpt/src/main/java/com/luoye/dpt/builder/AndroidPackage.java::addJunkCodeDex()`

  

**处理过程**：

```java

// 将 junkcode.dex 添加到 DEX 目录

addDex(getJunkCodeDexPath(), getDexDir(packageDir));

// 例如：classes.dex, classes2.dex, junkcode.dex

```

  

**目的**：

- **反调试**：垃圾代码类包含检测逻辑

- **混淆**：增加分析难度

- **运行时检测**：在类加载时触发反调试检查

  

**结果**：

- DEX 目录中新增 `junkcode.dex`（包含垃圾代码类）

  

---

  

### 步骤 3：compressDexFiles - 压缩 DEX 文件

  

**实现位置**：`dpt/src/main/java/com/luoye/dpt/builder/AndroidPackage.java::compressDexFiles()`

  

**处理过程**：

```java

// 1. 将所有 DEX 文件压缩成 ZIP（STORE 模式，不压缩）

Map<String, CompressionMethod> rulesMap = new HashMap<>();

rulesMap.put("classes\\d*.dex", CompressionMethod.STORE);

ZipUtils.compress(getDexFiles(getDexDir(packageDir)),

unalignedFilePath, rulesMap);

  

// 2. 执行 zipalign 对齐（优化内存映射）

ZipAlign.alignZip(randomAccessFile, out);

  

// 3. 保存到 assets 目录

// 文件：assets/classes.dex.zip

```

  

**目的**：

- **隐藏 DEX**：压缩后不易被直接提取

- **对齐优化**：zipalign 提升加载性能

- **统一管理**：多个 DEX 合并为一个 ZIP

  

**结果**：

- `assets/classes.dex.zip`：包含所有 DEX 文件（压缩包）

  

---

  

### 步骤 4：deleteAllDexFiles - 删除所有 DEX 文件

  

**实现位置**：`dpt/src/main/java/com/luoye/dpt/builder/AndroidPackage.java::deleteAllDexFiles()`

  

**处理过程**：

```java

List<File> dexFiles = getDexFiles(getDexDir(packageDir));

for (File dexFile : dexFiles) {

dexFile.delete(); // 删除所有 .dex 文件

}

```

  

**目的**：

- **清理**：删除已处理的 DEX（代码已抽取）

- **防止泄露**：避免原始 DEX 残留在 APK 中

- **准备合并**：为后续合并壳 DEX 做准备

  

**结果**：

- DEX 目录清空（所有 .dex 文件已删除）

  

---

  

### 步骤 5：combineDexZipWithShellDex - 将壳 DEX 与原 DEX 合并

  

**实现位置**：`dpt/src/main/java/com/luoye/dpt/builder/AndroidPackage.java::combineDexZipWithShellDex()`

  

**处理过程**：

```java

// 1. 读取壳 DEX（ProxyApplication 等）

byte[] shellDex = readFile(getProxyDexPath());

  

// 2. 读取压缩的 DEX ZIP

byte[] zipData = readFile(originalDexZipFile); // assets/classes.dex.zip

  

// 3. 合并：壳 DEX + ZIP 数据 + ZIP 长度（4字节）

byte[] newDexBytes = new byte[shellDexLen + zipDataLen + 4];

System.arraycopy(shellDex, 0, newDexBytes, 0, shellDexLen);

System.arraycopy(zipData, 0, newDexBytes, shellDexLen, zipDataLen);

System.arraycopy(intToByte(zipDataLen), 0, newDexBytes, totalLen-4, 4);

  

// 4. 修复 DEX 头部（file_size, SHA1, CheckSum）

FileUtils.fixFileSizeHeader(newDexBytes);

FileUtils.fixSHA1Header(newDexBytes);

FileUtils.fixCheckSumHeader(newDexBytes);

  

// 5. 生成新的 classes.dex

writeFile(targetDexFile, newDexBytes);

```

  

**目的**：

- **隐藏 ZIP**：将 ZIP 数据嵌入 DEX 尾部

- **伪装**：看起来是普通 DEX，实际包含压缩包

- **运行时提取**：从 DEX 尾部读取 ZIP 长度，提取压缩包

  

**文件结构**：

```

新的 classes.dex:

┌─────────────────┐

│ 壳 DEX 代码 │ ProxyApplication, JniBridge 等

├─────────────────┤

│ ZIP 数据 │ classes.dex.zip（原 DEX 文件）

├─────────────────┤

│ ZIP 长度(4B) │ 用于运行时定位 ZIP 起始位置

└─────────────────┘

```

  

**运行时提取**（`shell/src/main/cpp/dpt_util.cpp`）：

```cpp

// shell 运行时从 DEX 文件末尾读取 ZIP 长度

uint32_t zipLen = *(uint32_t*)(dexEnd - 4);

uint8_t* zipData = dexBegin + (dexSize - zipLen - 4);

// 解压 ZIP 得到原始 DEX 文件

```

  

---

  

### 步骤 6：addKeepDexes - 添加保留 DEX 文件

  

**实现位置**：`dpt/src/main/java/com/luoye/dpt/builder/AndroidPackage.java::addKeepDexes()`

  

**处理过程**：

```java

File keepDexTempDir = getKeepDexTempDir(packageDir);

File[] files = keepDexTempDir.listFiles();

  

// 将保留的 DEX 文件添加到 DEX 目录

for (File file : files) {

if (file.getName().endsWith(".dex")) {

addDex(file.getAbsolutePath(), getDexDir(packageDir));

}

}

```

  

**目的**：

- **保留部分类**：匹配排除规则的类保留在原始 DEX 中

- **兼容性**：某些类需要直接可见（如系统组件）

- **性能**：减少运行时填回的工作量

  

**结果**：

- 如果启用了 `keepClasses`，会添加 `classes2.dex`、`classes3.dex` 等保留的 DEX

  

---

  

### 步骤 7：删除临时保留 DEX 目录

  

**处理过程**：

```java

FileUtils.deleteRecurse(apk.getKeepDexTempDir(apkMainProcessPath));

```

  

**目的**：

- **清理临时文件**：删除处理过程中的临时目录

  

---

  

### DEX 处理流程总结

  

```

原始 APK:

├── classes.dex (原始代码)

├── classes2.dex (原始代码)

└── ...

  

步骤 1: extractDexCode

├── classes.dex (代码已抽取，方法体为空)

├── classes2.dex (代码已抽取，方法体为空)

└── assets/

└── classes.dex.dat (CodeItem 数据)

  

步骤 2: addJunkCodeDex

├── classes.dex (代码已抽取)

├── classes2.dex (代码已抽取)

└── junkcode.dex (垃圾代码)

  

步骤 3: compressDexFiles

├── classes.dex (代码已抽取)

├── classes2.dex (代码已抽取)

├── junkcode.dex (垃圾代码)

└── assets/

├── classes.dex.dat (CodeItem 数据)

└── classes.dex.zip (压缩的 DEX 文件)

  

步骤 4: deleteAllDexFiles

└── assets/

├── classes.dex.dat (CodeItem 数据)

└── classes.dex.zip (压缩的 DEX 文件)

  

步骤 5: combineDexZipWithShellDex

├── classes.dex (壳 DEX + ZIP 数据)

└── assets/

├── classes.dex.dat (CodeItem 数据)

└── classes.dex.zip (已删除)

  

步骤 6: addKeepDexes (如果启用)

├── classes.dex (壳 DEX + ZIP 数据)

├── classes2.dex (保留的类，如果启用 keepClasses)

└── assets/

└── classes.dex.dat (CodeItem 数据)

```

  

### 为什么这样做？

  

1. **代码隐藏**：原始字节码不在 APK 的 DEX 中，运行时动态填回，静态分析无法直接看到

2. **混淆**：方法体被替换，增加逆向难度；垃圾代码增加干扰

3. **保护**：ZIP 嵌入 DEX，隐藏压缩包；运行时从内存中提取，不落盘

4. **兼容性**：保留部分类，确保系统组件正常工作；支持 MultiDex

5. **性能**：zipalign 对齐优化加载；按需填回，减少内存占用

  

---

  

## 三、已实现功能的实现方式

  

### 3.1 三代代码分离保护（核心功能）

  

**实现位置**：

- `dpt/src/main/java/com/luoye/dpt/util/DexUtils.java`

- `shell/src/main/cpp/dpt_hook.cpp`

  

**实现原理**：

  

1. **编译时抽取**（DexUtils.extractMethod）：

```java

// 1. 读取方法的 CodeItem

Code code = dex.readCode(method);

int insnsOffset = method.getCodeOffset() + 16; // CodeItem 头部 16 字节

  

// 2. 保存原始指令

byte[] byteCode = new byte[insnsCapacity * 2];

for (int i = 0; i < insnsCapacity; i++) {

byteCode[i * 2] = outRandomAccessFile.readByte();

byteCode[i * 2 + 1] = outRandomAccessFile.readByte();

}

  

// 3. 替换为 return 语句或随机指令

if(obfuscateIns) {

outRandomAccessFile.writeShort(insRandom.nextInt()); // 随机指令

} else {

outRandomAccessFile.writeShort(0x0e); // nop 指令

}

outRandomAccessFile.write(returnByteCodes); // 写入 return

```

  

2. **运行时填回**（dpt_hook.cpp::patchMethod）：

```cpp

// 1. Hook DefineClass，拦截类加载

void* DefineClassV22(...) {

patchClass(descriptor, dex_file, dex_class_def);

return g_originDefineClassV22(...);

}

  

// 2. 遍历类的所有方法

for (uint64_t i = 0; i < direct_methods_size; i++) {

auto method = directMethods[i];

patchMethod(begin, location.c_str(), dexSize, dexIndex,

method.method_idx_delta_, method.code_off_);

}

  

// 3. 从内存中查找 CodeItem 并写回

void patchMethod(...) {

// 计算 method_idx

uint32_t methodIdx = ...;

// 从 dexMap 查找

auto codeItemVec = dexMap[dexIndex];

data::CodeItem *codeItem = codeItemVec->at(methodIdx);

// 计算 insns 位置

uint8_t *insns = begin + code_off + 16;

// 写回字节码

memcpy(insns, codeItem->getInsns(), codeItem->getInsnsSize());

}

```

  

### 3.2 指令混淆

  

**实现位置**：`DexUtils.extractMethod()`

  

**实现原理**：

- 默认模式：用 `0x0e` (nop) 填充

- 混淆模式：用随机数填充（`insRandom.nextInt()`）

- 增加逆向分析难度

  

### 3.3 垃圾代码生成与保护

  

**实现位置**：

- `dpt/src/main/java/com/luoye/dpt/dex/JunkCodeGenerator.java`

- `shell/src/main/cpp/dpt_hook.cpp::patchClass()`

  

**实现原理**：

  

1. **生成垃圾类**：

```java

// 生成 50-100 个垃圾类

for(int i = 0; i < generateClassCount; i++) {

// 每个类包含：

// - <clinit>：调用 System.exit(0)

// - <init>：调用 System.exit(0)

// - 随机方法：System.exit(0) 或抛出 NullPointerException

}

```

  

2. **运行时检测**：

```cpp

// 检测垃圾类是否被非法调用

if(descriptor 包含 junkClassName) {

char ch = descriptor[descriptorLength - 2];

if(isdigit(ch)) { // 如果类名包含数字（说明被调用）

dpt_crash(); // 崩溃

}

}

```

  

### 3.4 SO 文件加密

  

**实现位置**：

- `dpt/src/main/java/com/luoye/dpt/builder/AndroidPackage.java::encryptSoFile()`

- `shell/src/main/cpp/dpt.cpp::decrypt_bitcode()`

  

**实现原理**：

  

1. **编译时加密**：

```java

// 读取 .bitcode 段

ReadElf readElf = new ReadElf(soFile);

List<SectionHeader> sections = readElf.getSectionHeaders();

for (SectionHeader section : sections) {

if (".bitcode".equals(section.getName())) {

byte[] bitcode = IoUtils.readFile(soFile, section.getOffset(), section.getSize());

byte[] enc = CryptoUtils.rc4Crypt(rc4Key, bitcode); // RC4 加密

IoUtils.writeFile(soFile, enc, section.getOffset());

}

}

```

  

2. **运行时解密**：

```cpp

// 在 .init_array 中解密

void decrypt_bitcode() {

// 1. 获取 SO 文件路径

Dl_info info;

dladdr((void*)decrypt_bitcode, &info);

// 2. 读取 .bitcode 段

Elf_Shdr shdr;

get_elf_section(&shdr, so_path, ".bitcode");

// 3. 修改内存权限为可写

dpt_mprotect(target, target + size, PROT_READ | PROT_WRITE | PROT_EXEC);

// 4. RC4 解密

rc4_init(&dec_state, key, 16);

rc4_crypt(&dec_state, target, bitcode, size);

memcpy(target, bitcode, size);

// 5. 恢复内存权限

dpt_mprotect(target, target + size, PROT_READ | PROT_EXEC);

}

```

  

### 3.5 Frida 检测

  

**实现位置**：`shell/src/main/cpp/dpt_risk.cpp`

  

**实现原理**：

```cpp

void* detectFridaOnThread(void* args) {

while (true) {

// 1. 检测 Frida SO 文件

int frida_so_count = find_in_maps(1, "frida-agent");

if(frida_so_count > 0) {

dpt_crash();

}

// 2. 检测 Frida 线程

int frida_thread_count = find_in_threads_list(4,

"pool-frida", "gmain", "gbus", "gum-js-loop");

if(frida_thread_count >= 2) {

dpt_crash();

}

sleep(10);

}

}

```

  

### 3.6 子进程反调试

  

**实现位置**：`shell/src/main/cpp/dpt_risk.cpp`

  

**实现原理**：

```cpp

void createAntiRiskProcess() {

pid_t child = fork();

if(child == 0) {

// 子进程：检测 Frida + ptrace

detectFrida();

doPtrace(); // PTRACE_TRACEME

} else {

// 主进程：监控子进程

protectChildProcess(child);

detectFrida();

}

}

  

void* protectProcessOnThread(void* args) {

pid_t child = *((pid_t*)args);

int pid = waitpid(child, nullptr, 0);

if(pid > 0) { // 子进程被调试器 attach 会退出

dpt_crash();

}

}

```

  

### 3.7 execve Hook（阻止 dex2oat）

  

**实现位置**：`shell/src/main/cpp/dpt_hook.cpp`

  

**实现原理**：

```cpp

int fake_execve(const char *pathname, char *const argv[], char *const envp[]) {

if (strstr(pathname, "dex2oat") != nullptr) {

errno = EACCES;

return -1; // 阻止 dex2oat 执行

}

return BYTEHOOK_CALL_PREV(fake_execve, pathname, argv, envp);

}

```

  

### 3.8 字符串混淆

  

**实现位置**：`shell/src/main/cpp/common/obfuscate.h`

  

**实现原理**：

- 编译时使用 XOR 加密字符串

- 运行时自动解密

- 防止字符串在二进制文件中可见

  

### 3.9 mmap Hook（使 DEX 可写）

  

**实现位置**：`shell/src/main/cpp/dpt_hook.cpp`

  

**实现原理**：

```cpp

void* fake_mmap(void* __addr, size_t __size, int __prot, int __flags, int __fd, off_t __offset) {

int hasRead = (__prot & PROT_READ) == PROT_READ;

int hasWrite = (__prot & PROT_WRITE) == PROT_WRITE;

if(hasRead && !hasWrite) {

prot = prot | PROT_WRITE; // 添加写权限

}

return BYTEHOOK_CALL_PREV(fake_mmap, __addr, __size, prot, __flags, __fd, __offset);

}

```

  

---

  

## 四、未实现功能的实现思路

  

### 4.1 二代 DEX 整体加密保护

  

**实现思路**：

1. 编译时：使用 AES 加密整个 DEX 文件

2. 运行时：在 ClassLoader 加载 DEX 前解密

3. 参考：https://github.com/luoyesiqiu/dpt-shell/issues

  

**关键技术点**：

- Hook `DexFile.openDexFile()` 或 `DexFile.loadDex()`

- 在内存中解密 DEX 后再传递给系统

  

### 4.2 文件完整性校验

  

**实现思路**：

1. 编译时：计算 DEX/SO/资源文件的 SHA256 哈希值

2. 运行时：定期校验文件哈希

3. 检测到篡改：崩溃或退出

  

**实现代码示例**：

```cpp

// 编译时计算哈希

std::string calculateHash(const char* filePath) {

SHA256_CTX ctx;

SHA256_Init(&ctx);

// 读取文件并计算哈希

return hash;

}

  

// 运行时校验

void verifyIntegrity() {

std::string currentHash = calculateHash(dexPath);

if(currentHash != expectedHash) {

dpt_crash();

}

}

```

  

**参考资料**：

- Android 文件完整性校验：https://developer.android.com/training/articles/security-config

  

### 4.3 ROOT 检测

  

#### 4.3.1 实现思路

  

ROOT 检测应该采用**多层次、多方式**的综合检测策略，参考现有的 `detectFrida()` 实现方式，在 `dpt_risk.cpp` 中实现。

  

**检测方式**：

1. **文件系统检测**：检测 su 文件、Magisk 文件、root 管理应用

2. **系统属性检测**：检测 `ro.build.tags`、`ro.debuggable` 等属性

3. **环境变量检测**：检测 PATH 中的 su

4. **进程检测**：检测可疑的 root 相关进程

5. **SELinux 状态检测**：检测 SELinux 是否被禁用

  

#### 4.3.2 代码结构设计

  

**文件位置**：

- `shell/src/main/cpp/dpt_risk.h`：添加函数声明

- `shell/src/main/cpp/dpt_risk.cpp`：实现检测逻辑

- `shell/src/main/cpp/dpt.cpp`：集成到 `createAntiRiskProcess()`

  

**函数设计**：

```cpp

// dpt_risk.h

void detectRoot(); // 启动 ROOT 检测

  

// dpt_risk.cpp

bool isRooted(); // 检测设备是否已 ROOT

void* detectRootOnThread(void* args); // 后台检测线程

```

  

#### 4.3.3 检测方式详解

  

**方式 1：检测 su 文件路径**

```cpp

const char *su_paths[] = {

"/system/bin/su",

"/system/xbin/su",

"/sbin/su",

"/vendor/bin/su",

"/data/local/su",

"/data/local/bin/su",

"/data/local/xbin/su",

"/system/sbin/su",

"/system/bin/failsafe/su",

"/system/xbin/daemonsu",

"/system/etc/init.d/99SuperSUDaemon",

"/dev/com.koushikdutta.superuser.daemon/",

"/system/app/Superuser.apk",

"/system/app/SuperSU.apk",

nullptr

};

  

for (int i = 0; su_paths[i] != nullptr; i++) {

if (access(su_paths[i], F_OK) == 0) {

return true;

}

}

```

  

**方式 2：检测 Magisk 相关文件**

```cpp

const char *magisk_paths[] = {

"/sbin/magisk",

"/system/bin/magisk",

"/system/xbin/magisk",

"/data/adb/magisk",

"/cache/magisk.log",

"/data/adb/magisk.db",

"/data/adb/modules",

nullptr

};

```

  

**方式 3：检测系统属性**

```cpp

char prop_value[256] = {0};

  

// 检测 ro.build.tags（test-keys 表示测试版本）

if (__system_property_get("ro.build.tags", prop_value) > 0) {

if (strstr(prop_value, "test-keys") != nullptr) {

return true;

}

}

  

// 检测 ro.debuggable（调试版本，可能更容易 ROOT）

prop_value[0] = '\0';

if (__system_property_get("ro.debuggable", prop_value) > 0) {

if (strcmp(prop_value, "1") == 0) {

// 注意：仅 ro.debuggable=1 不一定表示 ROOT，可作为参考

}

}

  

// 检测 ro.secure（安全模式）

prop_value[0] = '\0';

if (__system_property_get("ro.secure", prop_value) > 0) {

if (strcmp(prop_value, "0") == 0) {

return true; // ro.secure=0 通常表示已 ROOT

}

}

```

  

**方式 4：检测 root 管理应用数据目录**

```cpp

const char *root_app_paths[] = {

"/data/data/com.noshufou.android.su",

"/data/data/com.thirdparty.superuser",

"/data/data/eu.chainfire.supersu",

"/data/data/com.topjohnwu.magisk",

"/data/data/com.kingroot.kinguser",

"/data/data/com.kingo.root",

"/data/data/com.smedialink.oneclickroot",

"/data/data/com.zhiqupk.root.global",

"/data/data/com.alephzain.framaroot",

"/data/data/com.devadvance.rootcloak",

"/data/data/com.devadvance.rootcloakplus",

nullptr

};

```

  

**方式 5：检测 PATH 环境变量中的 su**

```cpp

FILE *fp = popen("which su", "r");

if (fp != nullptr) {

char result[256] = {0};

if (fgets(result, sizeof(result), fp) != nullptr) {

if (strlen(result) > 0 && result[0] != '\n') {

pclose(fp);

return true;

}

}

pclose(fp);

}

```

  

**方式 6：检测 SELinux 状态**

```cpp

// 读取 /proc/filesystems 或检查 SELinux 状态

FILE *fp = fopen("/proc/filesystems", "r");

if (fp != nullptr) {

char line[256];

while (fgets(line, sizeof(line), fp)) {

// 检查 SELinux 相关文件系统

}

fclose(fp);

}

  

// 或者通过 getenforce 命令检测

FILE *fp = popen("getenforce", "r");

if (fp != nullptr) {

char result[16] = {0};

if (fgets(result, sizeof(result), fp) != nullptr) {

if (strstr(result, "Disabled") != nullptr) {

pclose(fp);

return true; // SELinux 被禁用，可能已 ROOT

}

}

pclose(fp);

}

```

  

**方式 7：检测 /proc 文件系统中的可疑进程**

```cpp

// 读取 /proc 目录，检测可疑的 root 相关进程

DIR *dir = opendir("/proc");

if (dir != nullptr) {

struct dirent *entry;

while ((entry = readdir(dir)) != nullptr) {

if (entry->d_type == DT_DIR && isdigit(entry->d_name[0])) {

char cmdline_path[256];

snprintf(cmdline_path, sizeof(cmdline_path),

"/proc/%s/cmdline", entry->d_name);

FILE *fp = fopen(cmdline_path, "r");

if (fp != nullptr) {

char cmdline[256] = {0};

fgets(cmdline, sizeof(cmdline), fp);

fclose(fp);

// 检测可疑进程名

if (strstr(cmdline, "su") != nullptr ||

strstr(cmdline, "magisk") != nullptr ||

strstr(cmdline, "superuser") != nullptr) {

closedir(dir);

return true;

}

}

}

}

closedir(dir);

}

```

  

#### 4.3.4 实现代码结构

  

**核心检测函数**：

```cpp

/**

* 检测设备是否已 ROOT

* 综合多种检测方式，提高准确性

* @return true 如果检测到 ROOT，false 否则

*/

DPT_ENCRYPT bool isRooted() {

// 方式 1：检测 su 文件路径

// 方式 2：检测 Magisk 相关文件

// 方式 3：检测系统属性

// 方式 4：检测 root 管理应用数据目录

// 方式 5：检测 PATH 环境变量中的 su

// 方式 6：检测 SELinux 状态

// 方式 7：检测 /proc 文件系统中的可疑进程

// 如果任一方式检测到 ROOT，返回 true

return false;

}

```

  

**后台检测线程**：

```cpp

/**

* ROOT 检测线程（持续运行）

* 定期检测设备 ROOT 状态

*/

[[noreturn]] DPT_ENCRYPT void *detectRootOnThread(__unused void *args) {

while (true) {

if (isRooted()) {

DLOGD("detected root, crashing...");

dpt_crash();

}

sleep(10); // 每 10 秒检测一次

}

}

```

  

**启动检测函数**：

```cpp

/**

* 启动 ROOT 检测

* 1. 立即检测一次

* 2. 启动后台检测线程

*/

DPT_ENCRYPT void detectRoot() {

// 立即检测一次

if (isRooted()) {

DLOGD("detected root immediately, crashing...");

dpt_crash();

}

// 启动后台检测线程

pthread_t t;

pthread_create(&t, nullptr, detectRootOnThread, nullptr);

}

```

  

#### 4.3.5 集成位置

  

**集成到 `createAntiRiskProcess()`**：

```cpp

// dpt.cpp::createAntiRiskProcess()

DPT_ENCRYPT void createAntiRiskProcess() {

// 首先检测 ROOT（在主进程和子进程都要检测）

detectRoot();

pid_t child = fork();

if(child < 0) {

// fork 失败

DLOGW("fork fail!");

detectFrida();

detectRoot(); // 添加 ROOT 检测

}

else if(child == 0) {

// 子进程：检测 Frida、ROOT 和 ptrace

DLOGD("in child process");

detectFrida();

detectRoot(); // 添加 ROOT 检测

doPtrace();

}

else {

// 主进程：监控子进程 + 检测 Frida 和 ROOT

DLOGD("in main process, child pid: %d", child);

protectChildProcess(child);

detectFrida();

detectRoot(); // 添加 ROOT 检测

}

}

```

  

#### 4.3.6 执行流程

  

```

应用启动

↓

init_dpt() (SO 加载时，.init_array)

↓

createAntiRiskProcess()

↓

detectRoot() (立即检测一次)

├─→ isRooted() 综合检测

│ ├─→ 检测 su 文件路径

│ ├─→ 检测 Magisk 文件

│ ├─→ 检测系统属性

│ ├─→ 检测 root 管理应用

│ ├─→ 检测 PATH 中的 su

│ ├─→ 检测 SELinux 状态

│ └─→ 检测可疑进程

│

└─→ 如果检测到 ROOT → dpt_crash()

│

└─→ 启动 detectRootOnThread() 后台线程

└─→ 每 10 秒检测一次

└─→ 如果检测到 ROOT → dpt_crash()

```

  

#### 4.3.7 字符串混淆

  

**使用 AY_OBFUSCATE 宏**：

所有检测路径都应该使用 `AY_OBFUSCATE()` 宏进行字符串混淆，防止静态分析：

  

```cpp

const char *su_path = AY_OBFUSCATE("/system/bin/su");

const char *magisk_path = AY_OBFUSCATE("/sbin/magisk");

const char *prop_name = AY_OBFUSCATE("ro.build.tags");

```

  

#### 4.3.8 注意事项

  

1. **误报风险**：

- `ro.debuggable=1` 不一定表示 ROOT（可能是开发版本）

- `ro.build.tags=test-keys` 可能是官方测试版本

- 建议综合多种检测方式，避免单一检测的误报

  

2. **绕过可能**：

- Magisk Hide 可能会隐藏 ROOT 痕迹

- 某些高级 ROOT 工具可能会 Hook 系统调用

- 建议结合其他检测方式（如 Frida 检测）

  

3. **性能影响**：

- 文件系统检测（access）性能开销很小

- 系统属性检测性能开销很小

- `/proc` 目录遍历可能有一定开销，建议限制检测频率

- 后台线程每 10 秒检测一次，对性能影响很小

  

4. **兼容性**：

- `popen()` 在某些 Android 版本可能不可用，需要添加错误处理

- `/proc` 目录访问需要权限，可能在某些设备上失败

- 建议添加异常处理，避免检测失败导致应用崩溃

  

5. **检测时机**：

- 应用启动时立即检测一次

- 后台线程持续检测（每 10 秒）

- 主进程和子进程都要检测，提高可靠性

  

6. **日志输出**：

- 使用 `DLOGD()` 输出调试日志

- 检测到 ROOT 时输出警告日志

- 避免输出敏感信息（如检测路径）

  

#### 4.3.9 头文件依赖

  

**需要在 `dpt_risk.h` 中添加**：

```cpp

#include <sys/system_properties.h> // 系统属性检测

#include <fcntl.h> // access() 函数

#include <stdio.h> // popen(), fgets()

#include <dirent.h> // /proc 目录遍历

```

  

#### 4.3.10 测试建议

  

1. **正常设备测试**：

- 在未 ROOT 的设备上运行，应用应正常启动

- 检查 logcat，确认没有误报

  

2. **ROOT 设备测试**：

- 在已 ROOT 的设备上运行，应用应立即崩溃

- 测试不同的 ROOT 工具（SuperSU、Magisk、KingRoot 等）

  

3. **绕过测试**：

- 测试 Magisk Hide 是否能绕过检测

- 测试删除 su 文件后是否能绕过检测

  

4. **性能测试**：

- 监控检测线程的 CPU 占用

- 确认检测不影响应用正常使用

  

#### 4.3.11 参考资源

  

- **RootBeer**：https://github.com/scottyab/rootbeer

- **Android 系统属性**：https://source.android.com/devices/tech/config

- **Magisk 文档**：https://topjohnwu.github.io/Magisk/

- **SELinux 文档**：https://source.android.com/security/selinux

  

### 4.4 模拟器检测

  

**实现思路**：

1. 检测系统属性：`ro.kernel.qemu`, `ro.hardware`

2. 检测硬件特征：IMEI、手机号、传感器

3. 检测文件特征：`/dev/socket/qemud`

  

**实现代码示例**：

```cpp

bool isEmulator() {

char prop[256];

__system_property_get("ro.kernel.qemu", prop);

if(strcmp(prop, "1") == 0) return true;

__system_property_get("ro.hardware", prop);

if(strstr(prop, "goldfish")) return true;

if(access("/dev/socket/qemud", F_OK) == 0) return true;

return false;

}

```

  

**参考资料**：

- Android Emulator Detection：https://github.com/Fuzion24/AndroidEmulatorDetection

  

### 4.5 Xposed/Substrate 检测

  

**实现思路**：

1. 检测 Xposed：`/system/framework/XposedBridge.jar`

2. 检测 Substrate：`/data/data/com.saurik.substrate`

3. Hook 检测：尝试 Hook 系统函数，检测是否被 Hook

  

**实现代码示例**：

```cpp

bool isXposedInstalled() {

if(access("/system/framework/XposedBridge.jar", F_OK) == 0) return true;

// 检测 Xposed 类

void* handle = dlopen("libxposed_art.so", RTLD_NOW);

if(handle != nullptr) {

dlclose(handle);

return true;

}

return false;

}

```

  

**参考资料**：

- Xposed 检测：https://github.com/rovo89/XposedBridge

  

### 4.6 字符串加密（DEX 中）

  

**实现思路**：

1. 编译时：使用 ASM 或 Javassist 修改 DEX，加密字符串常量

2. 运行时：Hook `String` 类的初始化，解密字符串

  

**实现方案**：

- 使用 ASM 在编译时修改字节码

- 将所有字符串替换为加密后的字节数组

- 运行时通过 JNI 解密

  

**参考资料**：

- ASM 框架：https://asm.ow2.io/

- 字符串加密示例：https://github.com/obfuscator-llvm/obfuscator

  

### 4.7 资源文件加密

  

**实现思路**：

1. 编译时：加密 res/ 和 assets/ 中的资源文件

2. 运行时：Hook `AssetManager.open()` 和 `Resources.openRawResource()`

3. 在读取时解密

  

**实现代码示例**：

```cpp

// Hook AssetManager.open()

int fake_AAssetManager_open(AAssetManager* mgr, const char* filename, int mode) {

AAsset* asset = AAssetManager_open(mgr, filename, mode);

if(asset != nullptr && isEncrypted(filename)) {

// 读取加密数据

// 解密

// 返回解密后的 Asset

}

return asset;

}

```

  

**参考资料**：

- Android 资源保护：https://developer.android.com/guide/topics/resources/providing-resources

  

### 4.8 VMP 虚拟化保护

  

**实现思路**：

1. 将 DEX 指令转换为自定义虚拟指令

2. 实现虚拟解释器执行虚拟指令

3. 虚拟指令与真实指令一一对应

  

**实现方案**：

- 使用 LLVM Obfuscator 或自定义编译器

- 将 Dalvik 指令映射到虚拟指令集

- 实现虚拟解释器（类似 JVM）

  

**参考资料**：

- LLVM Obfuscator：https://github.com/obfuscator-llvm/obfuscator

- VMP 原理：https://bbs.pediy.com/thread-216095.htm

  

### 4.9 Java2CPP 转换

  

**实现思路**：

1. 使用 J2C 工具将 Java 代码转换为 C++

2. 编译为 SO 文件

3. 通过 JNI 调用

  

**实现方案**：

- 使用 JNI 生成工具（如 SWIG）

- 或使用自定义编译器

  

**参考资料**：

- JNI 开发指南：https://docs.oracle.com/javase/8/docs/technotes/guides/jni/

  

### 4.10 防内存 DUMP

  

**实现思路**：

1. Hook `memcpy`, `fwrite` 等内存操作函数

2. 检测内存读取操作

3. 混淆内存布局

  

**实现代码示例**：

```cpp

void* fake_memcpy(void* dest, const void* src, size_t n) {

// 检测是否在读取 DEX 内存

if(isDexMemory(src)) {

// 返回假数据或崩溃

dpt_crash();

}

return BYTEHOOK_CALL_PREV(fake_memcpy, dest, src, n);

}

```

  

### 4.11 APK 签名校验

  

**实现思路**：

1. 在 Application 启动时校验 APK 签名

2. 与预期签名对比

3. 不匹配则退出

  

**实现代码示例**：

```java

public static boolean verifySignature(Context context) {

PackageManager pm = context.getPackageManager();

PackageInfo packageInfo = pm.getPackageInfo(

context.getPackageName(),

PackageManager.GET_SIGNATURES

);

Signature[] signatures = packageInfo.signatures;

String signature = signatures[0].toCharsString();

// 与预期签名对比

return signature.equals(EXPECTED_SIGNATURE);

}

```

  

**参考资料**：

- Android 签名机制：https://source.android.com/security/apksigning

  

### 4.12 防截屏

  

**实现思路**：

1. 在 Activity 中设置 `FLAG_SECURE`

2. 检测截屏事件（MediaProjection API）

  

**实现代码示例**：

```java

@Override

protected void onCreate(Bundle savedInstanceState) {

super.onCreate(savedInstanceState);

getWindow().setFlags(

WindowManager.LayoutParams.FLAG_SECURE,

WindowManager.LayoutParams.FLAG_SECURE

);

}

```

  

### 4.13 SharePreferences/SQLite 加密

  

**实现思路**：

1. 自定义 SharedPreferences 和 SQLiteOpenHelper

2. 在读写时自动加密/解密

  

**实现方案**：

- 使用 SQLCipher 加密数据库

- 使用 AES 加密 SharedPreferences

  

**参考资料**：

- SQLCipher：https://www.zetetic.net/sqlcipher/

- Android 数据加密：https://developer.android.com/training/articles/keystore

  

---

  

## 五、关键技术点总结

  

### 5.1 Hook 框架

- **Dobby**：用于 Hook ART 内部函数（DefineClass）

- **bhook**：用于 Hook 系统库函数（mmap, execve）

  

### 5.2 DEX 文件格式

- CodeItem 结构：16 字节头部 + insns 数组

- ClassData 结构：存储类的字段和方法信息

- 需要理解 DEX 文件格式才能正确提取和填回

  

### 5.3 内存管理

- 使用 `mmap` Hook 使 DEX 内存可写

- 使用 `mprotect` 修改内存权限

  

### 5.4 多版本兼容

- 支持 Android 5.0 - 14.0

- 不同版本 ART 结构体不同，需要适配

  

---

  

## 六、参考资料

  

1. **DEX 文件格式**：

- https://source.android.com/devices/tech/dalvik/dex-format

  

2. **ART 虚拟机**：

- https://source.android.com/devices/tech/dalvik

  

3. **Hook 技术**：

- Dobby：https://github.com/jmpews/Dobby

- bhook：https://github.com/bytedance/bhook

  

4. **Android 安全**：

- https://developer.android.com/training/articles/security-tips

  

5. **加固技术**：

- 看雪论坛：https://bbs.kanxue.com/

- 安全客：https://www.anquanke.com/