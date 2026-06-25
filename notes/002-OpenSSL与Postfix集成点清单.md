# OpenSSL 与 Postfix 集成点清单

## 1. 结论摘要

Postfix 对 OpenSSL 的使用不是“单点调用”，而是跨多个邮件链路环节的系统性依赖。最重要的场景有四类：

1. 入站 SMTP 的 STARTTLS / 受保护会话
2. 出站 SMTP / LMTP 的 TLS 连接
3. 证书、私钥、会话缓存、随机数等 TLS 基础设施
4. `postfix tls`、DANE/TLSA、`openssl_path` 这类运维辅助能力

因此，OpenSSL 在 Postfix 里的角色应该被视为“协议加密引擎 + 运维工具链”，而不是单纯的第三方库。

## 2. 集成总览

| 集成点 | 主要用途 | 依赖形态 | 典型文件 / 配置 |
|---|---|---|---|
| `smtpd` | 入站 SMTP STARTTLS、证书校验、会话加密 | `libssl` / `libcrypto` | `html/smtpd.8.html`、`conf/main.cf.default` |
| `smtp` | 出站 SMTP TLS、到远端 MTA 的加密传输 | `libssl` / `libcrypto` | `html/TLS_README.html`、`conf/main.cf.default` |
| `lmtp` | 出站 LMTP over TLS，常见于投递到下游 MDA/MTA | `libssl` / `libcrypto` | `conf/main.cf.default` |
| `tlsmgr` | TLS session cache、PRNG、会话材料管理 | Postfix TLS 框架 + OpenSSL | `html/OVERVIEW.html`、`src/tlsproxy/` |
| `tlsproxy` | TLS 握手与协议抽象层 | TLS library backend | `src/tlsproxy/` |
| `postfix tls` | 生成/检查 TLS 配置、证书、TLSA | `openssl(1)` CLI | `conf/postfix-tls-script` |
| DANE/TLSA | 计算 TLSA RRset、验证证书与公钥指纹 | `openssl(1)` CLI + TLS 配置 | `conf/postfix-tls-script` |
| 构建系统 | 检测 TLS 能力、链接库、宏开关 | `USE_TLS`、`libssl`、`libcrypto` | `makedefs`、相关 build 规则 |

## 3. 运行时路径

### 3.1 入站链路

典型链路是：

- 客户端连接 `smtpd`
- `smtpd` 提供 `STARTTLS`
- OpenSSL 完成握手、证书校验、加密套件协商
- Postfix 继续在同一会话里处理 SMTP 命令

如果启用了 wrapper mode，则 TLS 会在连接最早阶段建立，而不是等到 `STARTTLS` 升级。

### 3.2 出站链路

典型链路是：

- `smtp` 或 `lmtp` transport 连接远端
- 根据策略决定是否启用 TLS
- OpenSSL 执行握手、验证对端证书、记录会话状态
- 结果交回 Postfix 队列系统和投递逻辑

### 3.3 会话缓存与随机数

Postfix 文档里多次提到 `tlsmgr(8)`。它负责：

- TLS session cache
- PRNG 相关管理
- 连接复用中的 TLS 状态

这部分属于 Postfix 自己的 TLS 管理框架，但底层能力最终还是依赖 OpenSSL。

## 4. 配置层集成

`conf/main.cf.default` 里有直接可见的 TLS 相关配置项：

- `openssl_path = openssl`
- `smtp_tls_protocols`
- `smtp_tls_mandatory_protocols`
- `smtpd_tls_protocols`
- `smtpd_tls_mandatory_protocols`
- `lmtp_tls_protocols`
- `lmtp_tls_mandatory_protocols`
- `tls_ssl_options`

这说明 OpenSSL 在 Postfix 里的集成不是硬编码，而是通过配置参数和运行时策略驱动。

## 5. 运维工具链集成

`conf/postfix-tls-script` 是一个很重要的信号：Postfix 不只是在运行时使用 OpenSSL，还在运维面依赖它的命令行工具。

它会调用：

- `openssl version`
- `openssl ecparam`
- `openssl dgst`
- `openssl req`
- `openssl x509`
- `openssl genrsa`

用途包括：

- 生成证书签名请求
- 生成自签名证书
- 计算 TLSA 记录需要的指纹
- 检查本机 `openssl(1)` 是否具备所需算法

这意味着如果你在系统里替换 OpenSSL 版本，运维脚本也会受到影响，不只是守护进程。

## 6. 与 DANE / TLSA 的耦合

Postfix 的 DANE 支持高度依赖 OpenSSL 的以下能力：

- 证书解析
- 公钥导出
- 哈希摘要
- ECDSA / RSA 密钥处理

因此，DANE 不是“一个外部脚本功能”，而是 TLS 栈和 OpenSSL CLI 的组合能力。

## 7. 与认证的关系

OpenSSL 本身不负责 SMTP 用户认证。它只负责 TLS 层面的安全传输。

对认证来说，Postfix 通常还需要：

- SASL 机制
- 外部认证后端
- 证书或客户端身份校验

所以，即便 OpenSSL 已经接入，Postfix 仍然可能需要：

- 认证代理平面
- 认证决策服务
- 用户状态同步

如果你的整体架构里已经有单独的认证中心，OpenSSL 并不会替代它，只会替代加密传输部分。

## 8. 对 Go 改造的启示

如果你在考虑 Go 化改造，OpenSSL 相关部分通常最适合保持原样的原因是：

- TLS 协议实现本身成熟且复杂
- DANE / 证书 / 会话缓存逻辑已经和 Postfix 紧密耦合
- `postfix tls` 和运维脚本已经依赖 OpenSSL CLI

更合理的拆分方式是：

- Go：控制面、策略面、配置发布、审计
- Postfix：邮件协议面、队列、投递
- OpenSSL：TLS / 密码学执行与工具链

## 9. 不建议直接替换的部分

不建议把以下部分优先替换成自研代码：

- TLS 握手
- X.509 解析与验证
- DANE/TLSA 计算
- 密钥格式转换
- 会话缓存相关的低层逻辑

这些地方一旦替换，风险通常大于收益。

## 10. 优先集成的部分

如果你的目标是“降低耦合而不是重写加密栈”，优先做这几件事更有效：

- 统一 `openssl_path`
- 明确 OpenSSL 版本和安装前缀
- 把 TLS 参数下沉到控制面模板
- 把证书轮换、TLSA 生成、SNI 策略从人工脚本改成自动化
- 保留 Postfix 对 OpenSSL 的运行时调用

## 11. 结论

OpenSSL 对 Postfix 的集成点非常清晰：

- 运行时：SMTP / LMTP / STARTTLS / 会话缓存
- 运维面：证书、TLSA、DANE、命令行工具
- 构建面：TLS 支持检测和链接

如果你已经有独立的控制面，那么最好的策略不是替换 OpenSSL，而是把它纳入一个稳定、可版本化、可验证的 TLS 基础设施边界。
