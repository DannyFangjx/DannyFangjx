# OIDC 身份认证：从账号孤岛到现代 SSO

先回答一个常见问题：

> SSO 是不是登录一次后就完全授权了，因此才有 OAuth 2.0？

不是。SSO、OAuth 2.0 和 OIDC 解决的是不同问题：

| 概念 | 解决的问题 |
| --- | --- |
| 认证 Authentication | 你是谁？ |
| 授权 Authorization | 你能做什么？ |
| SSO | 访问多个系统时，是否需要重复认证？ |

SSO 复用的是统一身份中心的登录状态，不是把所有系统的权限都交给用户。即使用户通过 SSO 进入了订单系统，订单系统仍然要自己判断他能查看哪些订单。

## 从账号孤岛到 SSO

早期每个系统都维护自己的账号密码：OA、邮箱、CRM、财务系统需要分别登录。用户要记住多套密码，管理员也需要在多个系统里重复处理员工入职和离职。

SSO（Single Sign-On）由此产生：

> 用户完成一次认证后，可以访问多个互相信任的系统，不需要反复输入密码。

SSO 是一种目标和用户体验，不是某个具体协议。共享 Cookie、Kerberos、CAS、SAML 和 OIDC 都可以在不同场景中实现 SSO。

早期方案通常依赖共享 Cookie 或私有协议，只适用于同一家公司控制的系统。跨域名、跨厂商后，需要一种标准方式传递认证结果。

## SAML：标准化企业 SSO

SAML 2.0 在 2005 年成为 OASIS 标准，主要包含两个角色：

- IdP（Identity Provider）：负责认证用户，例如企业身份中心。
- SP（Service Provider）：信任 IdP 的业务系统，例如 OA、CRM。

典型流程如下：

```text
用户访问 SP
    ↓
SP 将用户重定向到 IdP
    ↓
IdP 完成认证
    ↓
IdP 返回签名后的 SAML Assertion
    ↓
SP 验证 Assertion，建立自己的登录 Session
```

SAML Assertion 可以携带认证结果、用户属性，甚至简单的授权决定。但实际应用中，SAML 主要负责企业身份联邦，具体业务权限仍由 SP 管理。

SAML 解决了跨域和跨厂商的企业 SSO，但它基于 XML，协议和配置相对复杂，而且最初主要面向浏览器，对现代 App、API 和前后端分离系统不够自然。

## OAuth 2.0：另一条授权演进线

OAuth 2.0 不是 SAML 的下一代，它来自另一类问题。

假设一个日程助手需要读取用户的 Google Calendar。直接把 Google 密码交给日程助手显然不安全：应用会获得过大的权限，用户也很难单独撤销授权。

OAuth 2.0 使用 Access Token 解决这个问题：

```text
用户同意 calendar.readonly
    ↓
授权服务器向日程助手签发 Access Token
    ↓
日程助手使用 Access Token 调用 Calendar API
```

OAuth 2.0 的主要角色是：

- Resource Owner：资源所有者，通常是用户。
- Client：希望访问资源的客户端。
- Authorization Server：处理授权请求并签发 Token。
- Resource Server：接收 Access Token 并提供 API。

Access Token 表示客户端被允许访问哪些资源，`scope` 用来描述授权范围。OAuth 2.0 也支持没有用户参与的机器授权，因此它本身不是登录协议，更没有规定统一的用户身份格式。

## OIDC：在 OAuth 2.0 上增加认证

互联网应用曾普遍使用 OAuth 2.0 获取用户资料，再自行把它解释成“登录成功”。但不同平台的接口、字段和安全规则各不相同，缺少统一的身份认证标准。

OIDC（OpenID Connect）在 OAuth 2.0 之上增加了身份层，最重要的变化是引入 ID Token。

```text
OAuth 2.0：Access Token → 客户端能访问什么 API
OIDC：      ID Token     → 当前用户是谁
```

OIDC 请求包含 `openid` scope。ID Token 通常是签名后的 JWT，包含：

- `iss`：由谁签发。
- `sub`：用户在该签发方下的唯一标识。
- `aud`：Token 签发给哪个应用。
- `exp`：Token 何时过期。

OIDC 中的角色名称也有所变化：

- OP（OpenID Provider）：提供认证并签发 ID Token，是 OIDC 语境下的 IdP。
- RP（Relying Party）：信任 OP 的业务系统，对应 SAML 中的 SP。

OIDC 复用了 OAuth 2.0 的授权端点、授权码和 Token 端点，但 ID Token 与 Access Token 不能混用：ID Token 交给 RP 验证身份，Access Token 交给 Resource Server 调用 API。

## OIDC 如何实现 SSO

假设公司把 Google Workspace 作为统一身份中心，系统 A 和系统 B 都接入 Google OIDC。

用户首次访问系统 A：

```text
系统 A → 跳转 Google → 用户登录 → 返回授权码
       → 后端换取并验证 ID Token → 建立系统 A Session
```

之后用户访问系统 B：

```text
系统 B → 跳转 Google → Google 已有登录 Session
       → 无需再次输入密码 → 系统 B 建立自己的 Session
```

SSO 的关键不是系统 A 把 Session 共享给系统 B，而是两个系统都信任同一个 OP，并且 OP 已经保存了用户的登录状态。

每个系统仍然拥有独立 Session 和权限体系：

```text
Google：证明这是 Alice
系统 A：判断 Alice 能查看哪些订单
系统 B：判断 Alice 能管理哪些项目
```

因此，禁用 IdP 账号通常只能阻止后续认证。业务系统里尚未过期的 Session、API Token 或本地账号，还需要通过会话撤销、账号同步等机制回收。

## 概念对照

| 概念 | 本质 | 核心产物或角色 |
| --- | --- | --- |
| SSO | 目标和体验 | 一次认证，访问多个系统 |
| IdP | 通用身份提供者角色 | 认证用户、提供身份信息 |
| SAML | 身份联邦协议 | IdP、SP、SAML Assertion |
| OAuth 2.0 | 授权框架 | Access Token、Scope |
| OIDC | OAuth 2.0 之上的认证协议 | OP、RP、ID Token |
| Session | 某个系统自己的登录状态 | Cookie 或其他会话凭证 |

这些概念的真实演进关系不是一条直线。SAML 和 OIDC 都能实现 SSO，而 OIDC 复用了 OAuth 2.0 的框架：

```text
账号孤岛 → SSO 需求 ─┬→ SAML：传统企业身份联邦
                     └→ OIDC：现代身份认证与 SSO
                            ↑
第三方委托访问 → OAuth 2.0 ┘
```

最后记住五点：

1. SSO 不等于完全授权。
2. OAuth 2.0 是授权框架，不是登录协议。
3. OIDC 使用 ID Token 标准化用户认证。
4. ID Token 和 Access Token 的接收者、用途不同。
5. 登录成功后，业务权限仍由各个系统管理。

## 参考资料

- [OAuth 2.0 Authorization Framework（RFC 6749）](https://www.rfc-editor.org/rfc/rfc6749)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0-18.html)
- [SAML 2.0 Technical Overview](https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0-cd-02.html)
