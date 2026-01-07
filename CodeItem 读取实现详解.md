## 一、DEX 文件格式中的 CodeItem 结构

  

### 1.1 CodeItem 结构定义

  

CodeItem 是 DEX 文件中存储方法字节码的数据结构，定义如下：

  

```cpp

// shell/src/main/cpp/dex/dex_file.h

struct CodeItem {

uint16_t registers_size_; // 寄存器数量（局部变量 + 参数）

uint16_t ins_size_; // 输入参数的字数

uint16_t outs_size_; // 输出参数的字数（方法调用所需）

uint16_t tries_size_; // try_items 的数量（异常处理）

uint32_t debug_info_off_; // 调试信息的文件偏移

uint32_t insns_size_in_code_units_; // insns 数组的大小（以 2 字节为单位）

uint16_t insns_[1]; // 字节码数组（实际大小可变）

};

```

  

**CodeItem 总大小 = 16 字节（头部）+ insns_size_in_code_units_ * 2 字节（字节码）**

  

### 1.2 CodeItem 在 DEX 文件中的位置

  

```

DEX 文件布局：

┌─────────────────┐

│ Header │ DEX 文件头

├─────────────────┤

│ StringIds │ 字符串 ID 表

├─────────────────┤

│ TypeIds │ 类型 ID 表

├─────────────────┤

│ ProtoIds │ 原型 ID 表

├─────────────────┤

│ FieldIds │ 字段 ID 表

├─────────────────┤

│ MethodIds │ 方法 ID 表

├─────────────────┤

│ ClassDefs │ 类定义表

├─────────────────┤

│ ClassData │ 类数据（字段和方法信息）

├─────────────────┤

│ CodeItem │ 方法代码（通过 code_off 定位）

└─────────────────┘

```

  

---

  

## 二、编译时读取 CodeItem（dpt 模块）

  

### 2.1 定位 CodeItem

  

**步骤 1：从 ClassDef 获取 ClassData**

  

```java

// DexUtils.java: extractAllMethods()

// 1. 遍历所有类定义

Iterable<ClassDef> classDefs = dex.classDefs();

for (ClassDef classDef : classDefs) {

// 2. 读取类的数据（包含字段和方法信息）

ClassData classData = dex.readClassData(classDef);

// 3. 获取所有方法

ClassData.Method[] methods = classData.allMethods();

}

```

  

**步骤 2：从 Method 获取 code_offset**

  

```java

// ClassData.Method 结构包含：

// - method_idx_delta_: 方法索引（相对于前一个方法的增量）

// - access_flags_: 访问标志

// - code_off_: CodeItem 在 DEX 文件中的偏移（关键！）

  

for (ClassData.Method method : methods) {

int codeOffset = method.getCodeOffset(); // 获取 code_off

if (codeOffset == 0) {

// code_off 为 0 表示 native 方法或抽象方法，没有代码

continue;

}

}

```

  

**步骤 3：定位到 CodeItem 的 insns 数组**

  

```java

// DexUtils.java: extractMethod()

// CodeItem 头部大小 = 16 字节

// insns 数组的偏移 = code_offset + 16

int insnsOffset = method.getCodeOffset() + 16;

```

  

### 2.2 读取 CodeItem 内容

  

**使用 Android dx 库读取 CodeItem：**

  

```java

// DexUtils.java: extractMethod()

// 1. 使用 dx 库读取 Code 结构（包含 CodeItem 的所有信息）

Code code = dex.readCode(method);

  

// 2. 获取 insns 数组的容量（指令数量）

int insnsCapacity = code.getInstructions().length;

  

// 3. 使用 RandomAccessFile 直接读取原始字节码

RandomAccessFile randomAccessFile = new RandomAccessFile(outDexFile, "rw");

byte[] byteCode = new byte[insnsCapacity * 2]; // 每个指令 2 字节

  

for (int i = 0; i < insnsCapacity; i++) {

// 定位到 insns 数组的第 i 个指令位置

randomAccessFile.seek(insnsOffset + (i * 2));

// 读取原始字节码（每个指令 2 字节）

byteCode[i * 2] = outRandomAccessFile.readByte();

byteCode[i * 2 + 1] = outRandomAccessFile.readByte();

}

```

  

**完整流程：**

  

```java

private static Instruction extractMethod(Dex dex,

RandomAccessFile outRandomAccessFile,

ClassDef classDef,

ClassData.Method method,

boolean obfuscateIns) throws Exception {

// 1. 获取 code_offset（CodeItem 在 DEX 文件中的偏移）

int codeOffset = method.getCodeOffset();

if (codeOffset == 0) {

return null; // native 方法或抽象方法

}

// 2. 计算 insns 数组的偏移

// CodeItem 头部 = registers_size(2) + ins_size(2) + outs_size(2)

// + tries_size(2) + debug_info_off(4) + insns_size(4) = 16 字节

int insnsOffset = codeOffset + 16;

// 3. 使用 dx 库读取 Code 结构

Code code = dex.readCode(method);

// 4. 获取 insns 数组大小（指令数量）

int insnsCapacity = code.getInstructions().length;

// 5. 读取原始字节码

byte[] byteCode = new byte[insnsCapacity * 2];

for (int i = 0; i < insnsCapacity; i++) {

randomAccessFile.seek(insnsOffset + (i * 2));

byteCode[i * 2] = randomAccessFile.readByte();

byteCode[i * 2 + 1] = randomAccessFile.readByte();

}

// 6. 保存到 Instruction 对象

Instruction instruction = new Instruction();

instruction.setMethodIndex(method.getMethodIndex());

instruction.setInstructionDataSize(insnsCapacity * 2);

instruction.setInstructionsData(byteCode);

return instruction;

}

```

  

---

  

## 三、运行时读取 CodeItem（shell 模块）

  

### 3.1 从内存中的 DEX 结构读取

  

**步骤 1：获取 DEX 文件在内存中的地址**

  

```cpp

// dpt_hook.cpp: patchClass()

// 根据不同 Android 版本获取 DEX 文件结构

if(g_sdkLevel >= 35) {

auto* dexFileV35 = (V35::DexFile *)dex_file;

begin = (uint8_t *)dexFileV35->begin_; // DEX 文件在内存中的起始地址

dexSize = dexFileV35->header_->file_size_;

}

else if(g_sdkLevel >= __ANDROID_API_P__){

auto* dexFileV28 = (V28::DexFile *)dex_file;

begin = (uint8_t *)dexFileV28->begin_;

dexSize = dexFileV28->size_;

}

else {

auto* dexFileV21 = (V21::DexFile *)dex_file;

begin = (uint8_t *)dexFileV21->begin_;

dexSize = dexFileV21->size_;

}

```

  

**步骤 2：从 ClassDef 读取 ClassData**

  

```cpp

// dpt_hook.cpp: patchClass()

auto* class_def = (dex::ClassDef *)dex_class_def;

  

// 计算 ClassData 在内存中的地址

auto *class_data = (uint8_t *)((uint8_t *) begin + class_def->class_data_off_);

  

// 读取 ClassData 头部（ULEB128 编码）

uint64_t static_fields_size = 0;

read += DexFileUtils::readUleb128(class_data, &static_fields_size);

  

uint64_t instance_fields_size = 0;

read += DexFileUtils::readUleb128(class_data + read, &instance_fields_size);

  

uint64_t direct_methods_size = 0;

read += DexFileUtils::readUleb128(class_data + read, &direct_methods_size);

  

uint64_t virtual_methods_size = 0;

read += DexFileUtils::readUleb128(class_data + read, &virtual_methods_size);

```

  

**步骤 3：读取方法信息（包含 code_off）**

  

```cpp

// dex_file.cpp: readMethods()

size_t dpt::DexFileUtils::readMethods(uint8_t *data,

dpt::dex::ClassDataMethod *method,

uint64_t count) {

size_t read = 0;

uint32_t methodIndexDelta = 0;

for (uint64_t i = 0; i < count; ++i) {

// 读取 method_idx_delta（ULEB128）

uint64_t methodIndex = 0;

read += readUleb128(data + read, &methodIndex);

methodIndexDelta += methodIndex;

// 读取 access_flags（ULEB128）

uint64_t accessFlags = 0;

read += readUleb128(data + read, &accessFlags);

// 读取 code_off（ULEB128）- 关键！

uint64_t codeOff = 0;

read += readUleb128(data + read, &codeOff);

method[i].method_idx_delta_ = methodIndexDelta;

method[i].access_flags_ = accessFlags;

method[i].code_off_ = codeOff; // CodeItem 的偏移

}

return read;

}

```

  

**步骤 4：定位并读取 CodeItem**

  

```cpp

// dpt_hook.cpp: patchMethod()

void patchMethod(uint8_t *begin, const char *location,

uint32_t dexSize, int dexIndex,

uint32_t methodIdx, uint32_t codeOff) {

// 1. 计算 CodeItem 结构体在内存中的地址

auto *dexCodeItem = (dex::CodeItem *)(begin + codeOff);

// 2. 计算 insns 数组在内存中的地址

// CodeItem 结构体地址 + 16 字节头部 = insns 数组地址

auto *realInsnsPtr = (uint8_t *)(dexCodeItem->insns_);

// 3. 从 dexMap 中查找对应的 CodeItem（编译时保存的）

auto codeItemVec = dexMap[dexIndex];

auto codeItem = codeItemVec->at(methodIdx);

// 4. 将保存的字节码写回 insns 数组

memcpy(realInsnsPtr, codeItem->getInsns(), codeItem->getInsnsSize());

}

```

  

---

  

## 四、CodeItem 数据格式详解

  

### 4.1 CodeItem 头部字段

  

| 字段 | 大小 | 说明 |

|------|------|------|

| `registers_size_` | 2 字节 | 寄存器数量（局部变量 + 参数） |

| `ins_size_` | 2 字节 | 输入参数的字数 |

| `outs_size_` | 2 字节 | 输出参数的字数 |

| `tries_size_` | 2 字节 | try_items 数量（异常处理） |

| `debug_info_off_` | 4 字节 | 调试信息的文件偏移 |

| `insns_size_in_code_units_` | 4 字节 | insns 数组大小（以 2 字节为单位） |

  

**头部总大小：16 字节**

  

### 4.2 insns 数组格式

  

- **每个指令占 2 字节**（16 位）

- **指令数量 = `insns_size_in_code_units_`**

- **总字节数 = `insns_size_in_code_units_ * 2`**

  

### 4.3 读取示例

  

**示例：读取一个方法的 CodeItem**

  

假设 `code_off = 0x1000`，`insns_size_in_code_units_ = 10`：

  

```

DEX 文件内存布局：

地址 内容 说明

0x1000 0x0003 registers_size_ = 3

0x1002 0x0002 ins_size_ = 2

0x1004 0x0001 outs_size_ = 1

0x1006 0x0000 tries_size_ = 0

0x1008 0x00001234 debug_info_off_ = 0x1234

0x100C 0x0000000A insns_size_in_code_units_ = 10

0x1010 0x1A00 insns[0] = const/4 v0, #0

0x1012 0x1A01 insns[1] = const/4 v1, #1

0x1014 0x0E00 insns[2] = nop

... ... ...

0x1024 0x0F00 insns[9] = return-void

```

  

**读取代码：**

  

```java

// Java 侧（编译时）

int codeOffset = 0x1000;

int insnsOffset = codeOffset + 16; // = 0x1010

int insnsSize = 10; // 从 CodeItem 头部读取

  

byte[] insns = new byte[insnsSize * 2];

randomAccessFile.seek(insnsOffset);

for (int i = 0; i < insnsSize; i++) {

insns[i * 2] = randomAccessFile.readByte();

insns[i * 2 + 1] = randomAccessFile.readByte();

}

```

  

```cpp

// C++ 侧（运行时）

uint32_t codeOff = 0x1000;

uint8_t *begin = dexFile->begin_;

  

// 读取 CodeItem 结构体

dex::CodeItem *codeItem = (dex::CodeItem *)(begin + codeOff);

  

// 读取 insns 数组

uint16_t *insns = codeItem->insns_;

uint32_t insnsSize = codeItem->insns_size_in_code_units_;

  

// 访问第 i 个指令

uint16_t instruction = insns[i];

```

  

---

  

## 五、关键数据结构

  

### 5.1 ClassData.Method（方法信息）

  

```cpp

// dex_file.h

struct ClassDataMethod {

uint32_t method_idx_delta_; // 方法索引（相对于前一个方法的增量）

uint32_t access_flags_; // 访问标志

uint32_t code_off_; // CodeItem 的文件偏移（关键！）

};

```

  

### 5.2 ClassData（类数据）

  

```cpp

// dex_file.h

struct ClassDataHeader {

uint32_t static_fields_size_; // 静态字段数量

uint32_t instance_fields_size_; // 实例字段数量

uint32_t direct_methods_size_; // 直接方法数量（构造函数、静态方法）

uint32_t virtual_methods_size_; // 虚方法数量

};

```

  

**ClassData 的存储格式（ULEB128 编码）：**

  

```

[static_fields_size] (ULEB128)

[instance_fields_size] (ULEB128)

[direct_methods_size] (ULEB128)

[virtual_methods_size] (ULEB128)

[static_fields...] (每个字段：field_idx_delta + access_flags)

[instance_fields...] (每个字段：field_idx_delta + access_flags)

[direct_methods...] (每个方法：method_idx_delta + access_flags + code_off)

[virtual_methods...] (每个方法：method_idx_delta + access_flags + code_off)

```

  

### 5.3 ULEB128 编码

  

**ULEB128（Unsigned Little-Endian Base 128）**是一种变长编码，用于节省空间：

  

```cpp

// dex_file.cpp: readUleb128()

size_t readUleb128(uint8_t const *const data, uint64_t *const val) {

uint64_t result = 0;

size_t read = 0;

for(int i = 0; i < 5; i++) {

uint8_t b = *(data + i);

uint8_t value = b & 0x7f; // 取低 7 位

result |= (value << (i * 7)); // 左移 7*i 位

read++;

if((b & 0x80) != 0x80) { // 最高位为 0，表示结束

break;

}

}

*val = result;

return read;

}

```

  

**示例：**

- `0x01` → 1

- `0x81 0x01` → 129 (0x01 | (0x01 << 7))

- `0xFF 0xFF 0xFF 0xFF 0x0F` → 0xFFFFFFFF

  

---

  

## 六、完整读取流程总结

  

### 6.1 编译时流程（dpt 模块）

  

```

1. 遍历 ClassDefs

↓

2. 读取 ClassData（通过 class_data_off_）

↓

3. 解析 ClassDataHeader（ULEB128）

↓

4. 读取 Methods（每个方法包含 code_off）

↓

5. 根据 code_off 定位 CodeItem

↓

6. 读取 CodeItem 头部（16 字节）

↓

7. 读取 insns 数组（insns_size_in_code_units_ * 2 字节）

↓

8. 保存到 Instruction 对象

```

  

### 6.2 运行时流程（shell 模块）

  

```

1. Hook DefineClass（拦截类加载）

↓

2. 获取 DEX 文件内存地址（begin_）

↓

3. 从 ClassDef 读取 class_data_off_

↓

4. 解析 ClassData（ULEB128）

↓

5. 获取每个方法的 code_off_

↓

6. 计算 CodeItem 地址：begin + code_off

↓

7. 计算 insns 地址：CodeItem + 16

↓

8. 从 dexMap 查找保存的 CodeItem

↓

9. 将字节码写回 insns 数组（memcpy）

```

  

---

  

## 七、关键技术点

  

### 7.1 为什么需要 code_off？

  

- **code_off** 是 CodeItem 在 DEX 文件中的**绝对偏移**

- 通过 `begin + code_off` 可以直接定位到 CodeItem

- 不需要遍历整个 DEX 文件

  

### 7.2 为什么 insns 偏移是 code_off + 16？

  

- CodeItem 头部固定 16 字节

- insns 数组紧跟在头部后面

- 所以 `insns_offset = code_off + 16`

  

### 7.3 为什么使用 ULEB128 编码？

  

- **节省空间**：小数值用 1 字节，大数值用多字节

- **DEX 文件格式标准**：ClassData 使用 ULEB128 编码

- **兼容性**：Android 官方标准

  

### 7.4 为什么需要修改内存权限？

  

- 系统加载 DEX 时使用**只读权限**（PROT_READ）

- 我们需要**写回 CodeItem**，需要写权限（PROT_WRITE）

- 通过 Hook mmap 添加写权限，或使用 mprotect 修改权限

  

---

  

## 八、参考代码位置

  

### 编译时读取

- `dpt/src/main/java/com/luoye/dpt/util/DexUtils.java::extractMethod()`

- 使用 Android dx 库：`dex.readCode(method)`

  

### 运行时读取

- `shell/src/main/cpp/dpt_hook.cpp::patchClass()` - 解析 ClassData

- `shell/src/main/cpp/dpt_hook.cpp::patchMethod()` - 定位并写回 CodeItem

- `shell/src/main/cpp/dex/dex_file.cpp::readMethods()` - 读取方法信息

  

### 数据结构定义

- `shell/src/main/cpp/dex/dex_file.h` - CodeItem、ClassData 结构定义

- `dpt/src/main/java/com/luoye/dpt/model/Instruction.java` - Instruction 模型

  

---

  

## 九、DEX 文件格式参考

  

- **官方文档**：https://source.android.com/devices/tech/dalvik/dex-format

- **CodeItem 结构**：https://source.android.com/devices/tech/dalvik/dex-format#code-item

- **ClassData 结构**：https://source.android.com/devices/tech/dalvik/dex-format#class-data-item