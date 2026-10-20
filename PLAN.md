# Keep Password Android — 计划书

> 技术栈：**Kotlin + Jetpack Compose**  
> 对齐产品：系统 Autofill（类 1Password）+ 与 PC 同一保险库/同步协议  
> 依赖：`keep-password-spec` 定稿前，可先搭工程骨架，**加密与同步逻辑等向量就绪再写死**

---

## 1. 目标

| 优先级 | 目标 |
|--------|------|
| P0 | 解锁、登录条目 CRUD、系统 Autofill 填充网页/App、云同步拉齐 PC |
| P1 | 生物识别解锁、锁定态填充再认证、TOTP、密码生成器、基础审计 |
| P2 | 服务器/数据库等运维条目完整编辑、扫码 TOTP、设备管理完善 |

**不做（首期）：** 桌面式全局热键、浏览器扩展、Windows UIA。

---

## 2. 建议工程结构

```text
keep-password-android/
  README.md
  PLAN.md
  settings.gradle.kts
  app/
    src/main/
      java/.../keeppassword/
        ui/                 # Compose 界面
        domain/             # 用例：Unlock、Sync、AutofillMatch
        data/
          vault/            # 读写加密保险库（对齐 spec）
          sync/             # Retrofit/OkHttp + OpenAPI
          secure/           # Keystore、加密偏好
        autofill/           # AutofillService + 可选 CredentialProvider
        unlock/             # 生物识别 / 短密钥 / 主密码
      res/
      AndroidManifest.xml
  autofill-lib/             # 可选：把填充逻辑拆 module
  crypto/                   # 可选：纯 Kotlin 实现或 JNI 调 Rust
```

最低 SDK：建议 **API 28+**（Autofill 体验）；Credential Manager 相关功能以 **API 34+** 渐进增强。

---

## 3. 技术选型

| 领域 | 选型 | 备注 |
|------|------|------|
| UI | Jetpack Compose + Material 3 | 列表/详情/解锁仿 1Password 信息架构即可 |
| 异步 | Kotlin Coroutines + Flow | |
| DI | Hilt 或 Koin | 任选其一，全仓统一 |
| 网络 | OkHttp + Retrofit；模型与 `spec/openapi` 对齐 | |
| 本地 | 加密 blob 文件（优先与 PC 同结构）或 SQLCipher | **优先文件 blob**，少一层映射 |
| 安全存储 | Android Keystore | 存设备包装密钥、refresh token 等 |
| 生物识别 | BiometricPrompt | |
| 填充 | `AutofillService`（必做） | Android 14+ 再评估 Credential Manager |
| 测试 | JUnit + 消费 `keep-password-spec/vectors` | |

---

## 4. 模块计划

### 4.1 Crypto / Vault（对接 spec）

- [ ] 实现 Argon2id + AES-256-GCM（或链接 Rust `.so`）
- [ ] 解析 Envelope / Payload，版本 2 可读、3 读写
- [ ] 短密钥解锁、主密码解锁（若有 passwordWrap）、加密密钥文件导入
- [ ] **跑通 spec 黄金向量** 后再接 UI

### 4.2 Sync

- [ ] 登录 / 绑定设备 / GET·PUT `/sync/vault`
- [ ] 处理 409 冲突（策略：提示用户保留本地或云端，首期可「拉取覆盖需确认」）
- [ ] 轮询间隔参考桌面 `SyncDefaults.DefaultPollInterval`（5 分钟），可配置
- [ ] Base URL：可配置；默认读 spec/server 文档（开发可用 `http://10.0.2.2:5000` 指模拟器访问本机）

### 4.3 UI（Compose）

- [ ] 解锁页（短密钥为主，生物识别快捷入口）
- [ ] 条目列表（搜索、类型筛选；首期突出 Login）
- [ ] 登录详情：复制、显示/隐藏、编辑保存
- [ ] 设置：同步账号、Autofill 引导、锁定超时、关于

### 4.4 AutofillService（对齐 1Password）

- [ ] 在系统设置中可被选为自动填充服务
- [ ] `onFillRequest`：根据 domain / package 匹配 Login
- [ ] 保险库锁定时：返回「解锁」认证 Intent，解锁后再填（两阶段）
- [ ] 首次填充未知 App：Allow Once / Always Allow，持久化 package ↔ entry 关联
- [ ] **不要**在 Chrome 里重复造扩展；引导用户开启系统自动填充（Chrome 新版本支持第三方服务）

匹配策略首期：

1. 网站：URL/域名与条目 `url` 匹配（规则与桌面 `DomainGrouping` 对齐，规则写入 spec）  
2. 原生 App：关联表（packageName → entryId）；无关联则用户搜索后 Always Allow  

### 4.5 安全与生命周期

- [ ] 后台超时自动锁定（参考桌面空闲锁定思路，时长可配置）
- [ ] 内存中的密码尽快清零（Kotlin 注意 String 不可变限制，敏感数据尽量用 `CharArray`/`ByteArray`）
- [ ] 禁止日志打印密钥/密码
- [ ] 任务切换是否隐藏内容（FLAG_SECURE 可配置）

---

## 5. 实施分期

### M0 — 工程骨架（不依赖完整向量）

- [ ] 创建 Android Studio 工程、Compose 空壳、导航
- [ ] Manifest 声明 AutofillService（可先空实现）
- [ ] 配置 flavor：`dev` / `prod` baseUrl

### M1 — 本地保险库

- [ ] 导入/打开与 PC 同步下来的 vault 文件（或开发用样例）
- [ ] 解锁 + 列表展示 Login
- [ ] 向量测试接入 CI（GitHub Actions `ubuntu` 即可）

### M2 — 云同步

- [ ] 登录官网账号、设备绑定、上下传 vault
- [ ] 与桌面互相同步冒烟

### M3 — Autofill P0

- [ ] 系统填充服务可用，浏览器/常用 App 能填用户名密码
- [ ] 锁定态引导解锁后再填

### M4 — P1 体验

- [ ] 生物识别、TOTP、生成器、Always Allow 关联管理页

---

## 6. 与 spec / 桌面的依赖顺序

```text
spec S0 文档清单
    → Android M0 骨架
spec S1 向量
    → Android M1 Vault
server 可用 + OpenAPI
    → Android M2 Sync
Android M3 Autofill
```

**阻塞点：** 没有稳定 Envelope/Payload 说明与向量前，不要大规模写业务 UI 绑定字段。

---

## 7. 对标 1Password 的检查清单

- [ ] 用户可在系统「密码、密钥与数据自动填充」中选择 Keep Password  
- [ ] 聚焦密码框出现填充建议（或锁定时出现解锁入口）  
- [ ] 填充前可要求生物识别/短密钥（产品决策：是否每次，写入设置）  
- [ ] App 关联可 Always Allow，且可在设置中解除  
- [ ] 不自己做无障碍的悬浮窗抢填充（优先系统框架）

桌面短密钥确认策略：移动端可用「生物识别代替短密钥」，短密钥作备用——需在产品说明中写清。

---

## 8. 验收（P0）

- [ ] 同一账号：PC 上传 → 手机下载 → 解锁看到相同 Login  
- [ ] 手机改密码 → 上传 → PC 拉取一致  
- [ ] Chrome（开启第三方自动填充）或系统浏览器可填充  
- [ ] 至少 1 个第三方 App（如 Twitter/X 或本地测试 App）可 Always Allow 后再次建议  
- [ ] spec 向量 CI 全绿  

---

## 9. 风险

| 风险 | 缓解 |
|------|------|
| Argon2 参数过重导致手机解锁卡顿 | 与桌面同参数但放后台线程；UI 明确进度 |
| Chrome 与系统 Autofill 冲突 | 文档写清开启步骤（参考 1P / Google 说明） |
| 字段映射与桌面不一致 | 强制以 spec `item-kinds.md` 为准 |
| String 驻留敏感数据 | 规范敏感缓冲类型；Code review 清单 |

---

## 10. 参考

- 桌面同步：`keep-password/src/KeepPassword.Core/Sync/`
- 桌面匹配：`AutofillMatcher` / `DomainGrouping`
- 服务端：`keep-password-server/backend/app/api/`
- 规范：`keep-password-spec/PLAN.md`
