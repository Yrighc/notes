# ProxyShield 移动端代理拦截检测与阻断设计文档

## 1. 项目背景

在生产环境中，移动应用面临大量流量劫持、调试代理、MITM 分析行为，例如：

- Charles / Fiddler / mitmproxy / Burp Suite
    
- 系统或 App 级 HTTP/HTTPS Proxy
    
- 局域网透明代理、流量复制设备
    

这些行为可能导致：

- API 协议被逆向
    
- 业务参数、加密逻辑被分析
    
- 风控/反作弊模型失效
    

**ProxyShield** 的目标是在**不依赖外部网络、不违反商店合规、不使用私有 API** 的前提下，实现高强度代理拦截检测与阻断能力。

---

## 2. 设计目标

### 核心目标

- 检测并识别常见代理与 MITM 环境
    
- 在检测命中时 **直接阻断敏感 API（策略 A）**
    
- 支持 **灰度 / 调试模式开关**，避免影响内部测试
    

### 设计约束

- ❌ 不使用外部 C2、DNS 探测、时间校验
    
- ❌ 不仅依赖系统 Proxy 配置（易被伪造）
    
- ❌ 不触发 Play Store / App Store 风险规则
    
- ✅ 支持离线运行
    
- ✅ 抵抗 Frida / Xposed / Hook 框架
    

---

## 3. 多层检测总体架构

ProxyShield 采用 **多信号加权评分模型**，而非单点判断：

```
┌────────────┐
│ System     │  ← 低可信（5%）
├────────────┤
│ HTTP       │  ← 辅助（10%）
├────────────┤
│ Behavioral │  ← 行为学（15%）
├────────────┤
│ TLS        │  ← 中强（30%）
├────────────┤
│ Native TCP │  ← 核心（40%）
└────────────┘
        ↓
   Score ≥ 阈值
        ↓
   阻断敏感 API
```

---

## 4. 各层检测设计说明

### 4.1 Network Layer（Native Socket 层）【核心】

**技术实现**

- 使用 JNI / POSIX socket 直接创建 TCP 连接
    
- 目标：IP:443（绕过 DNS & HTTP 栈）
    
- 设置极短超时（300ms）
    

**检测信号**

- connect 行为异常但非立即失败
    
- 透明代理常见：连接被中转 / 劫持
    

**优势**

- 不依赖系统代理配置
    
- 不经过 OkHttp / URLSession
    
- Frida 需 Native Hook，成本极高
    

**风险**

- 极端网络环境可能产生误报（已通过阈值控制）
    

---

### 4.2 TLS Layer（证书与握手完整性）

**技术实现**

- 自定义 TrustManager / SecTrust
    
- 检查系统信任链数量
    

**检测信号**

- 用户安装 CA 会导致证书链数量异常
    
- MITM 代理通常引入额外中间证书
    

**优势**

- 离线可用
    
- 直接命中代理核心能力
    

**绕过方式**

- Hook TrustManager（需配合 Network 层绕过）
    

---

### 4.3 HTTP Layer（协议语义）

**技术实现**

- 在请求发起前检测 Header
    

**检测信号**

- Via
    
- X-Forwarded-For
    
- Proxy-Connection
    

**说明**

- 这些 Header 理论上不应由客户端主动设置
    
- 仅作为辅助加权信号
    

---

### 4.4 System Layer（系统配置）

**Android**

- ConnectivityManager
    
- ProxyInfo / LinkProperties
    

**iOS**

- CFNetworkCopySystemProxySettings
    

**说明**

- 仅用于捕获普通用户代理配置
    
- 不作为强判断条件
    

---

### 4.5 Behavioral Layer（行为学 & 时序）

**技术实现**

- 测量：TCP 建连 → TLS 首包耗时
    
- 本地计算，无需联网
    

**检测信号**

- MITM / 流量镜像会显著放大 RTT
    

**特点**

- 无法通过简单 Hook 彻底消除
    
- 网络差异大，仅作辅助
    

---

## 5. 评分模型与阻断策略

|检测层|权重|
|---|---|
|Native Socket|40%|
|TLS|30%|
|Behavioral|15%|
|HTTP|10%|
|System|5%|

**判定规则**

- 累计 Score ≥ 阈值 → 判定为代理环境
    
- 触发 **策略 A：阻断敏感 API 调用**
    

---

## 6. 灰度 / 调试模式设计

### Android

- BuildConfig.DEBUG
    
- System Property：`proxyshield.gray=1`
    

### iOS

- DEBUG 编译宏
    
- 环境变量：`PROXYSHIELD_GRAY=1`
    

**行为**

- 灰度/调试模式下：所有检测直接返回 false
    
- 避免误伤测试、自动化、CI 环境
    

---

## 7. 合规性评估

|项目|结果|
|---|---|
|使用私有 API|❌|
|Root / Jailbreak 强检测|❌|
|外部网络探测|❌|
|数据上报|❌|
|商店审核风险|低|

---

## 8. 安全结论

ProxyShield 通过 **Native TCP + TLS 完整性 + 行为学** 的组合设计，使代理分析的绕过成本从：

> 修改系统设置 → 提升为 → Native 层 Hook + TLS 伪造 + 时序仿真

显著提高了对抗调试、逆向与 MITM 的整体门槛，适用于金融、风控、反作弊、高价值业务场景。

---

## 9. 后续可扩展方向

- TLS ClientHello 指纹校验
    
- QUIC / HTTP3 行为异常检测
    
- 与风控模型联动（非实时）
    

（文档结束）