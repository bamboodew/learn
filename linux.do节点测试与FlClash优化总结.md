# linux.do 节点测试与 FlClash 优化总结

## 一、FlClash 窗口优化

### 1.1 问题：全屏 FAB 按钮偏移

FlClash 最大化后悬浮按钮（FAB）超出屏幕可视区域。

**根因**：`window.dart` 中 `setMaximizable(false)` 禁用了窗口最大化能力，导致窗口管理器行为异常。

**修复**：

```dart
// D:\git\FlClash-git\lib\common\window.dart:38
setMaximizable(false) → setMaximizable(true)
```

### 1.2 启动默认最大化

在 `Window.show()` 方法中添加 `windowManager.maximize()`：

```dart
Future<void> show() async {
    render?.resume();
    await windowManager.show();
    await windowManager.maximize();  // 新增
    await windowManager.focus();
    await windowManager.setSkipTaskbar(false);
}
```

---

## 二、linux.do 节点可用性批量测试

### 2.1 背景

linux.do 使用 Cloudflare 防护，部分代理节点访问返回 403（被 Cloudflare 拦截）。需要批量测试所有代理节点，找出可正常访问的节点。

### 2.2 技术方案演进

| 阶段   | 方案                                    | 结果                            | 问题                                      |
| ------ | --------------------------------------- | ------------------------------- | ----------------------------------------- |
| v1     | PowerShell + curl                       | 脚本无法运行                    | Win PS5.1 中文编码问题（GBK vs UTF-8）    |
| v2     | Python + curl                           | 全部 403                        | curl TLS 指纹被 Cloudflare 识别为非浏览器 |
| v3     | Python + Playwright（共享 context）     | 前14个 OK，后续全 429           | 浏览器连接池复用旧节点连接                |
| **v4** | **Python + Playwright（独立 context）** | **25 OK / 11 429 / 17 TIMEOUT** | **最终方案，工作正常**                    |

### 2.3 核心发现：Cloudflare TLS 指纹检测

**关键实验**：同一代理节点、同一出口 IP，curl 返回 403，浏览器返回 200。

**结论**：Cloudflare 通过 TLS 指纹（JA3/JA4）区分真实浏览器和程序化客户端。curl、Go HTTP 客户端等都会被识别并拦截，只有真实浏览器引擎（如 Chromium）才能通过。

这意味着：

- 节点 IP 本身并没有被封禁
- 403 是针对客户端 TLS 指纹的拦截，不是针对 IP 的封禁
- 必须使用真实浏览器引擎进行测试才能获得准确结果

### 2.4 最终方案：Playwright 批量测试

**文件**：`C:\Users\mi\Desktop\test_linuxdo_nodes.py`

**原理**：

1. 通过 mihomo External Controller API（端口 9090）获取代理组和节点列表
2. 逐一切换活跃节点（`PUT /proxies/{group}`）
3. 每个节点创建新的 `BrowserContext`（强制新 TCP 连接）
4. 用真实 Chromium 浏览器访问 linux.do
5. 检测 HTTP 状态码：200=OK, 403=BANNED, 429=Rate Limited
6. 获取出口 IP（通过 api.ip.sb）
7. 关闭 context 释放连接，继续下一个节点
8. 导出 CSV 结果

**关键设计**：

```python
# 每个节点必须创建新的 BrowserContext
# 否则浏览器连接池会复用旧节点的 TCP 连接
for node in test_nodes:
    api_put(f"{api}/proxies/{group}", {"name": node})  # 切换节点
    time.sleep(1.5)  # 等待切换生效
    context = browser.new_context(...)  # 新 context = 新 TCP 连接
    page = context.new_page()
    resp = page.goto("https://linux.do/", ...)
    # ... 检测状态码和 IP
    context.close()  # 关闭释放连接
```

**使用方法**：

```bash
pip install playwright && playwright install chromium
python test_linuxdo_nodes.py
python test_linuxdo_nodes.py --api http://127.0.0.1:9090 --proxy 127.0.0.1:7890
```

### 2.5 测试结果对比

#### 第一机场（宝可梦）节点测试结果

| 测试方案          | OK (200) | BANNED (403) | Rate Limited (429) | TIMEOUT | 说明                     |
| ----------------- | -------- | ------------ | ------------------ | ------- | ------------------------ |
| curl 版本         | 0        | 41           | 0                  | 13      | TLS 指纹问题，403 是误判 |
| Playwright v1     | 14       | 0            | 27                 | 13      | context 复用导致 429     |
| **Playwright v2** | **25**   | **0**        | **11**             | **17**  | **真实结果**             |

#### 第二机场节点测试结果

| 状态         | 数量 | 说明                              |
| ------------ | ---- | --------------------------------- |
| BANNED (403) | 18   | 全部 403，IP 段被 Cloudflare 标记 |

#### 各类型节点可用性分析（第一机场 Playwright v2）

| 节点类型 | OK   | 429  | TIMEOUT | 说明                 |
| -------- | ---- | ---- | ------- | -------------------- |
| 直连     | 4/7  | 3/7  | 0       | 部分 429 是 IP 限速  |
| AWS      | 6/8  | 2/8  | 0       | 表现较好             |
| 专线 3x  | 6/12 | 4/12 | 2/12    | 中等                 |
| 家宽     | 4/5  | 0/5  | 1/5     | 表现最好             |
| V6       | 0/8  | 0/8  | 8/8     | 全部超时（连接问题） |
| IPLC     | 0/6  | 0/6  | 6/6     | 全部超时（连接问题） |

### 2.6 HTTP 状态码含义

| 状态码  | 含义         | 说明                                          |
| ------- | ------------ | --------------------------------------------- |
| 200     | OK           | 节点可正常访问 linux.do                       |
| 403     | BANNED       | Cloudflare 拦截（IP 被标记或 TLS 指纹不通过） |
| 429     | Rate Limited | 请求频率过高，临时限制（不是真正封禁）        |
| TIMEOUT | 超时         | 节点本身连接不通                              |

---

## 三、技术知识点

### 3.1 mihomo External Controller API

FlClash 底层使用 mihomo（原 Clash.Meta）核心，通过 RESTful API 暴露控制接口：

```
GET  http://127.0.0.1:9090/proxies          # 获取所有代理组和节点
PUT  http://127.0.0.1:9090/proxies/{group}  # 切换指定组的活跃节点
     Body: {"name": "节点名称"}
```

FlClash GUI 通过此 API 与核心通信。外部脚本也可直接调用。

### 3.2 Cloudflare TLS 指纹 (JA3/JA4)

Cloudflare 通过分析 TLS 握手参数（支持的密码套件、扩展、椭圆曲线等）生成客户端指纹。不同客户端（Chrome、Firefox、curl、Go HTTP）的 TLS 实现差异会产生不同的指纹。Cloudflare 维护白名单，非浏览器指纹会被拦截返回 403。

### 3.3 代理模式 vs TUN 模式

- **代理模式**（HTTP/SOCKS）：mihomo 重建 TCP 连接，暴露 Go 网络栈的 TLS 指纹
- **TUN 模式**：在 IP 层透明转发，保留浏览器原始的 TLS 连接特征

TUN 模式理论上可以绕过 TLS 指纹检测，但实测 FlClash 的 TUN 模式在 linux.do 场景下无明显区别（可能 Cloudflare 还有其他检测维度）。

### 3.4 浏览器 Context 隔离

Playwright 的 `BrowserContext` 是浏览器级别的隔离单元，拥有独立的：

- Cookie 存储
- 网络连接池
- 缓存

切换代理节点后必须创建新的 Context，否则旧的 TCP 连接会被复用，请求仍走旧节点。

---

## 四、嵌入 FlClash 的可行性分析

**结论：不实际。**

| 维度     | 评估                                                                 |
| -------- | -------------------------------------------------------------------- |
| 体积     | Chromium 约 100MB，FlClash 本体仅 30MB                               |
| 架构     | FlClash = Dart GUI + Go Core + Rust Helper，插入 Chromium 引擎不兼容 |
| 依赖     | 需要 Playwright/Puppeteer 运行时                                     |
| 替代方案 | 外部脚本 + mihomo API 调用是最优方案                                 |

---

## 五、文件清单

| 文件                                              | 说明                           |
| ------------------------------------------------- | ------------------------------ |
| `C:\Users\mi\Desktop\test_linuxdo_nodes.py`       | 最终版 Playwright 批量测试脚本 |
| `C:\Users\mi\Desktop\linuxdo_test_*.csv`          | 各次测试的 CSV 结果            |
| `D:\git\FlClash-git\lib\common\window.dart`       | FlClash 窗口逻辑（已修改）     |
| `D:\git\learn\Windows代理配置指南.md`             | 代理配置指南                   |
| `D:\git\learn\Flutter桌面项目开发构建部署指南.md` | 开发构建部署指南               |
| `D:\git\learn\FlClash内核断开排查报告.md`         | 内核断开排查报告               |
