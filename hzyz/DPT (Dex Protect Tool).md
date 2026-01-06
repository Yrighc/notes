## DPT 命令行工具使用说明书

用法格式：

java -jar dpt.jar [选项] -f <安装包文件>

### 核心参数 (必选)

- -f, --package-file <文件路径>
    
    需要保护的 Android 包文件路径，支持 .apk 和 .aab 格式。
    

### 通用配置选项

- -o, --output <目录>
    
    指定保护后的文件输出目录。
    
- -c, --protect-config <文件>
    
    指定保护配置文件。
    
- -r, --rules-file <文件>
    
    规则文件：用于指定不进行保护的类名（白名单）。
    
- -x, --no-sign
    
    不进行签名：处理完成后不对安装包进行二次签名。
    

### 性能与体积优化

- -K, --keep-classes
    
    保留类：在包中保留部分原始类名，可以在一定程度上提高 App 启动速度，但并非所有安装包都兼容此功能。
    
- -S, --smaller
    
    更小体积：牺牲一部分运行性能来换取更小的 App 安装包体积。
    
- -e, --exclude-abi <参数>
    
    排除特定的 ABI（架构）：多个架构用逗号分隔（例如：x86,x86_64）。
    
    支持的架构包括：arm (armeabi-v7a), arm64 (arm64-v8a), x86, x86_64。
    

### 调试与日志

- --debug
    
    使生成的安装包处于可调试状态。
    
- --dump-code
    
    导出代码：将 DEX 的代码项（code item）导出并保存为 .json 文件。
    
- --disable-acf
    
    禁用应用组件工厂（App Component Factory）：仅用于调试。
    
- --noisy-log
    
    开启详细日志：输出更琐碎的运行日志。
    
- -v, --version
    
    显示程序的版本号。
    

---

### 使用示例

1. 基础保护（最常用）：

java -jar dpt.jar -f my-app.apk -o ./output

2. 排除特定架构并指定排除规则：

java -jar dpt.jar -f my-app.apk -e x86,x86_64 -r ignore_rules.txt -o ./output

3. 以性能优化模式保护（保留部分类以提升启动速度）：

java -jar dpt.jar -f my-app.apk -K -o ./output

---
