# ProxyShield 代理检测测试方案与误报评估

## 1. 测试目标

本测试方案用于验证 ProxyShield 在真实与对抗环境下的：

- 代理 / MITM 检测准确性
    
- 多层评分模型稳定性
    
- 误报（False Positive）与漏报（False Negative）情况
    
- 灰度 / 调试模式是否正确放行
    

目标不是“100% 检出代理”，而是：

> **在可接受误报率内，大幅提高代理分析成本**

---

## 2. 测试维度总览

|维度|说明|
|---|---|
|网络环境|家庭网络 / 公司网络 / 移动网络|
|代理类型|显式代理 / 透明代理 / MITM|
|绕过手段|Hook / Root / 证书注入|
|设备状态|原生 / Root / Jailbreak|
|评分边界|单信号 / 多信号叠加|

---

## 3. 分层测试用例设计

### 3.1 System Layer（系统代理）

#### 用例 S-1：无代理（基线）

- 环境：家庭 Wi-Fi
    
- 系统代理：关闭
    
- 期望：
    
    - detectSystemProxy = false
        
    - 总评分 < 阈值
        

#### 用例 S-2：系统 HTTP/HTTPS 代理

- 工具：系统设置 + Charles
    
- 期望：
    
    - System Layer 命中
        
    - 单独命中不触发阻断
        

#### 用例 S-3：代理端口为空但 Host 存在

- 目的：验证防御简单伪造
    
- 期望：
    
    - 不影响最终判定
        

---

### 3.2 HTTP Layer（Header 注入）

#### 用例 H-1：无 Header 注入

- 抓包工具：无
    
- 期望：HTTP Layer 不命中
    

#### 用例 H-2：显式代理 Header

- 工具：Fiddler / Burp
    
- Header：Via / X-Forwarded-For
    
- 期望：
    
    - HTTP Layer 命中
        
    - 不单独触发阻断
        

#### 用例 H-3：手动清除 Header

- 工具：Burp 手工修改
    
- 期望：
    
    - HTTP Layer 不命中
        
    - 其他层仍可命中
        

---

### 3.3 TLS Layer（证书 & MITM）

#### 用例 T-1：官方证书链

- 环境：未安装用户 CA
    
- 期望：TLS Layer 不命中
    

#### 用例 T-2：安装 Charles / mitmproxy CA

- 环境：HTTPS MITM
    
- 期望：
    
    - 证书链长度异常
        
    - TLS Layer 命中
        

#### 用例 T-3：Hook TrustManager

- 工具：Frida / objection
    
- 期望：
    
    - TLS Layer 失效
        
    - Native Socket Layer 仍命中
        

---

### 3.4 Network Layer（Native Socket）【重点】

#### 用例 N-1：直连网络

- TCP 443 直连
    
- 期望：
    
    - nativeDetectProxy = false
        

#### 用例 N-2：Charles / Burp 显式代理

- 设置：系统代理
    
- 期望：
    
    - TCP connect 行为异常
        
    - Native Layer 命中
        

#### 用例 N-3：透明代理 / 流量镜像

- 环境：公司出口代理
    
- 期望：
    
    - connect 延迟或异常
        
    - Native Layer 命中
        

#### 用例 N-4：Frida Hook Java Socket

- 目的：验证 Native 层抗 Hook
    
- 期望：
    
    - Java 层绕过
        
    - Native 层仍命中
        

---

### 3.5 Behavioral Layer（时序）

#### 用例 B-1：4G / 5G 网络

- 高 RTT 但无代理
    
- 期望：
    
    - Behavioral 可能命中
        
    - 不应单独触发阻断
        

#### 用例 B-2：代理 + 高延迟

- 工具：mitmproxy + 延迟注入
    
- 期望：
    
    - Behavioral + Native + TLS 命中
        

---

## 4. 评分模型边界测试

### 4.1 单信号命中

|命中层|期望结果|
|---|---|
|System|放行|
|HTTP|放行|
|Behavioral|放行|
|TLS|⚠️（依配置）|
|Native|⚠️（依配置）|

> 设计原则：**弱信号绝不单独致命**

---

### 4.2 多信号组合

|组合|判定|
|---|---|
|System + HTTP|放行|
|TLS + HTTP|阻断|
|Native + 任意|阻断|
|Behavioral + TLS|阻断|

---

## 5. 误报（False Positive）分析

### 5.1 是否会出现大量误报？

**结论：不会“大量”，但可能出现“边缘误报”。**

原因：

- 单一信号不足以触发阻断
    
- Native + TLS 才是核心
    

---

### 5.2 可能的误报场景

|场景|风险|说明|
|---|---|---|
|企业出口代理|中|Native + TLS 可能命中|
|高延迟卫星网络|低|Behavioral 误报|
|安全设备 SSL 检查|中|类 MITM|

> 这些场景在**安全视角本身就不可信**，允许阻断是合理的

---

### 5.3 误报控制策略

1. **权重设计**：弱信号权重低
    
2. **阈值策略**：Score ≥ 阈值才阻断
    
3. **灰度开关**：快速止血
    
4. **按 API 分级**：仅阻断高价值接口
    

---

## 6. 测试执行建议

### 自动化

- Root 模拟器 + Frida
    
- Charles / Burp 脚本化
    

### 手工

- 公司网络 / 公共 Wi-Fi
    
- 不同运营商网络
    

---

## 7. 测试输出物

- 每个用例：
    
    - 命中层
        
    - 最终评分
        
    - 是否阻断
        
- 统计：
    
    - 误报率
        
    - 漏报率
        

---

## 8. 结论

ProxyShield 的评分模型本质是：

> **将“代理检测”从判断题变成风险评估题**

在此模型下：

- 普通用户几乎不受影响
    
- 分析者绕过成本指数级上升
    

（文档结束）