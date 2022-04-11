# Halo 中的安全性草案

## 目标

 - 提出针对系统安全性的具体实施方案
 - 充分考虑到对安全机制的可扩展性、易用性和可维护性
 - 提出如何降低针对后续版本升级可能存在的对安全性改动难度的解决方案

## 动机

基于现有 Halo 1.0 基础，我们有如下需求：

- Halo 需要一种可以方便接入其他认证方式的安全性机制
 - 需要一种可以同时兼容 API key 和 basic 认证的解决方案
 - 需要可以灵活创建 API Key
- 需要一种简单分配权限的机制
 - 要考虑到自定义 API 和插件 API 的有效认证和授权
 - 可以针对不同的使用场景分配不同的 API Key

## 设计

### 用户认证

Halo中的用户分为三类，管理员、普通用户、和访客。

Halo允许管理员查看或管理普通用户和访客，访客用户可由管理员开启注册功能后自行注册。

#### 身份认证策略 

Halo 通过身份认证模块利用持有者令牌（Bearer Token）来认证 API 请求的身份。HTTP 请求发给服务器时，认证模块会将以下属性关联到请求本身：

- 用户名：用来辩识最终用户的字符串。常见的值可以是 `admin` 或 `example@example.com`。
- 用户 ID：用来辩识最终用户的字符串，旨在比用户名有更好的一致性和唯一性。
- 附加字段：一组额外的键-值映射，键是字符串，值是一组字符串；用来保存一些鉴权组件可能 觉得有用的额外信息。

所有（属性）值对于身份认证系统而言都是不透明的，只有被 **鉴权组件** 解释过之后才有意义。

你可以同时启用多种身份认证方法，当启用了多个身份认证模块时，第一个成功地对请求完成身份认证的模块会直接做出评估决定。服务器并不保证身份认证模块的运行顺序。

#### 令牌格式 

- 用户登录成功后统一生成 `jwt` 格式的 `token`，`jwt` 使用的密钥由系统安装时生成 `RSAKey` 到工作目录。
- 管理员可以创建 `personal access token `, 格式为：`类型前缀_36 位短 token` 例如：`hc_ac5fQe9pERSXRlud3WydzpRVDI4nSh3Iqkcq`

[正如我们从 Slack](https://api.slack.com/authentication/token-types) 和 [Stripe](https://stripe.com/docs/api/authentication) 等公司看到的那样，令牌前缀是使令牌可识别的一种明确方法。`h` 表示 `halo`，`c` 表示令牌类型的第一个字母

 - `ha` 用于访问 `admin api` 的令牌
 - `hc` 用于访问 `content api` 的令牌

使这些前缀在令牌中清晰可辨，以提高可读性。因此，添加了一个分隔符：`_` 并且双击它时，它会可靠地选择整个令牌。

当需要身份验证（例如，对于跨域请求）进行身份验证，便可以使用 `-H "Authorization: Bearer ha_ac5fQe9pERSXRlud3WydzpRVDI4nSh3Iqkcq"`

 所有 `person access token` 颁发时都需要选择 `scope` 以确保令牌的访问权限在一个可控的范围内。

前端的菜单访问权限由前端根据登录用户的 scope 自行限制。

#### 认证方式

针对不同的 token 验证方式可以提供不同的 `Provider` 而后统一使用 `AuthenticationManager` 来进行认证，当然这会由 `Spring security` 帮我们处理，我们只需要根据不同的认证方式创建不同的 `Provider` 和过滤器即可。

例如：

针对使用用户名密码登录时颁发 `jwt token` 的过滤器和 Provider

- ```
  org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter
  ```

- ```
  org.springframework.security.authentication.dao.DaoAuthenticationProvider
  ```

 根据以上类针对 `personal access token` 提供

 - RestrictedApiKeyAuthenticationFilter
- RestrictedApiKeyAuthenticationProvider

 优化：

- `Content api` 可以不为其指定 `scope` 即允许访问所有公共的 `api` 资源，可以称之为 `publishable key`
- 一次性令牌也可使用同样的认证方式，例如格式为：`ho_ac5fQe9pERSXRlud3WydzpRVDI4nSh3Iqkcq`，并提供 `OttAuthenticationFilter` 和 `OttAuthenticationProvider` 来为一次性令牌策略进行认证。
- 对短令牌使用 CRC32 进行 checksum，可以过滤掉伪造的 token 的请求，减少查询数据库的次数

![authentication processing filter](https://docs.spring.io/spring-security/reference/_images/servlet/authentication/architecture/abstractauthenticationprocessingfilter.png)

#### 在请求中放入持有者令牌 

当使用持有者令牌来对某 HTTP 客户端执行身份认证时，API 服务器希望看到 一个名为 `Authorization` 的 HTTP 头，其值格式为 `Bearer THETOKEN`。 持有者令牌必须是一个可以放入 HTTP 头部值字段的字符序列，至多可使用 HTTP 的编码和引用机制。 例如：如果持有者令牌为 `31ada4fd-adec-460c-809a-9e56ceb75269`，则其 出现在 HTTP 头部时如下所示：

```
Authorization: Bearer 31ada4fd-adec-460c-809a-9e56ceb75269
```

#### 匿名请求 

启用匿名请求支持之后，如果请求没有被已配置的其他身份认证方法拒绝，则被视作 匿名请求（Anonymous Requests）。这类请求获得用户名 `system:anonymous` 和 对应的角色为 `system:unauthenticated`。

例如，在一个配置了令牌身份认证且启用了匿名访问的服务器上，如果请求提供了非法的 持有者令牌，则会返回 `401 Unauthorized` 错误。 如果请求没有提供持有者令牌，则被视为匿名请求。

### 鉴权概述

为了避免做一个全面的权限系统，本方案计划通过[基于角色的权限控制（RBAC）](https://en.wikipedia.org/wiki/Role-based_access_control)实施一个简单易用的授权方案。

相比与其他授权方式:

- [基于表达式的访问控制 (EBAC)](https://docs.spring.io/spring-security/site/docs/5.0.7.RELEASE/reference/html/el-access.html)
- [基于属性的访问控制 (ABAC)](https://en.wikipedia.org/wiki/Attribute-based_access_control)

### 使用 RBAC 鉴权

基于角色（Role）的访问控制（RBAC）是一种基于组织中用户的角色来调节控制对 计算机或网络资源的访问的方法。

提供一种通过API动态配置策略的方式来进行授权，配置示例如下：

```yaml
apiVersion: halo.run/v1
kind: "Role"
metadata:
  name: role-template-manage-categories
  labels:
    halo.run/role-template: true
  annotations:
    halo.run/dependencies: ["role-template-view-categories"]
    halo.run/module: "Categories Management"
    halo.run/alias-name: "Categories Management"
rules:
  - apiGroups: ["halo.run"]
    resources: ["categories"]
    verbs: ["*"] #["create", "delete", "deletecollection", "get", "list", "patch", "update"]
---
apiVersion: halo.run/v1
kind: "Role"
metadata:
  name: role-template-view-posts
  labels:
    halo.run/role-template: true
  annotations:
    halo.run/module: "Posts Management"
    halo.run/alias-name: "Posts View"
rules:
  - apiGroups: ["halo.run"]
    resources: ["posts", categories", "tags", "options", "settings", "users"]
    verbs: ["get", "list"]
---
apiVersion: halo.run/v1
kind: "Role"
metadata:
  name: role-template-manage-posts
  # creationTimestamp: "2022-03-23T03:10:55Z"
  labels:
    halo.run/role-template: true
  annotations:
    halo.run/dependencies:
      ["role-template-view-posts", "role-template-manage-categories"]
    halo.run/module: "Posts Management"
    halo.run/alias-name: "Posts Management"
rules: []
---
apiVersion: halo.run/v1
# 此角色绑定允许 "jane" 读取 "default" 名字空间中的 Pods
kind: RoleBinding
metadata:
  name: manage-posts
subjects:
  # 你可以指定不止一个“subject（主体）”
  - kind: User
    name: jane # "name" 是区分大小写的
    apiGroup: halo.run
roleRef:
  # "roleRef" 指定与某 Role 的绑定关系
  kind: Role # 此字段必须是 Role
  name: role-template-manage-posts # 此字段必须与你要绑定的 Role 的名称匹配
  apiGroup: halo.run
```

#### API 对象

RBAC 声明了两种可用对象：*Role*、 *RoleBinding* 来描述或修改对象。

#### Role

RBAC *角色*包含表示一组权限的规则。这些权限是纯粹累加的（不存在拒绝某操作的规则）。

#### Role 示例

```yaml
apiVersion: halo.run/v1
kind: "Role"
metadata:
  name: role-manage-categories
rules:
  - apiGroups: [""] # "" 标明 core API 组
    resources: ["categories"]
    verbs: ["*"]
```

它表示定义了一个名为`role-manage-categories`的角色，拥有核心`API`下所有`categories`的权限。

Halo API （以及生态系统中的相关 API）的约定旨在简化客户端开发并确保可以实现在各种不同用例中一致地工作的配置机制。

Halo API 的一般风格是 `RESTful`——客户端通过标准 HTTP 动词（POST、PUT、DELETE 和 GET）创建、更新、删除或检索对象的描述——这些 API 优先接受和返回 JSON。Halo 还为非标准动词公开了额外的端点，并允许替代`contentTypes`.。服务器接受和返回的所有 JSON 都有一个模式，由“kind”和“apiVersion”字段标识。如果存在相关的 HTTP 标头字段，它们应该反映 JSON 字段的内容，但信息不应仅在 HTTP 标头中表示。

以下的术语定义:

- **Kind** - 特定对象`schema`的名称（例如，“Cat”和“Dog”类型将具有不同的特征和属性）
- **Resource** - 系统实体的表示，通过 HTTP 以 JSON 形式发送或检索到服务器。资源通过以下方式公开：
  - 集合 - 相同类型的资源列表，可能是可查询的
  - 元素 - 单个资源，可通过 URL 寻址
- **API Group** - 表示一组被暴露在一起的资源以及在“apiversion”字段中暴露的版本，如“GROUP/VERSION”为“halo.run/v1”

- **apiVersion** - 表示一组被暴露在一起的资源以及在“apiversion”字段中暴露的版本，如“组/版本”，例如“组/版本”。“halo.run/v1”。

每个资源通常接受并返回单一种类的数据。一个种类可以被反映特定用例的多个资源接受或返回。

资源在 API 组中绑定在一起——每个组可能有一个或多个独立于其他 API 组发展的版本，并且组内的每个版本都有一个或多个资源。组名通常采用域名形式。——Halo 项目保留使用空组，选择群组名称时，我们建议选择您的组或组织拥有的子域，例如“widget.mycompany.com”。

资源集合应该都是小写和复数，而种类是 CamelCase 和单数。组名必须小写并且是有效的 DNS 子域。

> Reference: [types-kinds](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#types-kinds)

#### 资源动词

API 资源应该使用传统的 REST 模式：

- GET` /<resourceNamePlural>` - 检索` <resourceName>` 类型的列表，例如 GET /posts 返回 Post 列表。
- POST `/<resourceNamePlural>` - 从客户端提供的 JSON 对象创建一个新资源。
- GET `/<resourceNamePlural>/<name>` - 检索具有给定名称的单个资源，例如 GET `/users/halo` 返回一个名为“halo”的 用户。
- DELETE `/<resourceNamePlural>/<name>` - 删除具有给定名称的单个资源。
- DELETE `/<resourceNamePlural>` - 删除 `<resourceName>` 类型的列表，例如 DELETE /categories 一个 Category 列表。
- PUT `/<resourceNamePlural>/<name>` - 使用客户端提供的 JSON 对象更新或创建具有给定名称的资源。
- PATCH `/<resourceNamePlural>/<name>` - 有选择地修改资源的指定字段。

#### 确定请求动词

**非资源请求** 对除`/api/v1/...`或`/apis/<group>/<version>/...` 之外的端点的请求被视为 “非资源请求”，并使用请求的小写 HTTP 方法作为动词。例如，`GET`对端点的请求就像`/api`或`/healthz`将`get`用作动词。

| HTTP verb | request verb                                                 |
| --------- | ------------------------------------------------------------ |
| POST      | create                                                       |
| GET, HEAD | get (for individual resources), list (for collections, including full object content), watch (for watching an individual resource or collection of resources) |
| PUT       | update                                                       |
| PATCH     | patch                                                        |
| DELETE    | delete (for individual resources), deletecollection (for collections) |

允许针对非资源端点 `/healthz` 和其子路径上发起 GET 和 POST 请求示例：

```yaml
rules:
  - nonResourceURLs: ["/healthz", "/healthz/*"] # nonResourceURL 中的 '*' 是一个全局通配符
    verbs: ["get", "post"]
```

#### RoleBinding

角色绑定（Role Binding）是将角色中定义的权限赋予一个或者一组用户。 它包含若干 **主体**（用户、组或服务账户）的列表和对这些主体所获得的角色的引用。 

一个 RoleBinding 可以引用任何 Role。

RoleBinding 示例：

下面的例子中的 RoleBinding 将 "manage-posts" Role 授予用户 "jane"。 这样，用户 "jane" 就具有了管理文章的权限。

```yaml
apiVersion: halo.run/v1
# 此角色绑定允许 "jane" 读取 "default" 名字空间中的 Pods
kind: RoleBinding
metadata:
  name: manage-posts
subjects:
  # 你可以指定不止一个“subject（主体）”
  - kind: User
    name: jane # "name" 是区分大小写的
    apiGroup: halo.run
roleRef:
  # "roleRef" 指定与某 Role 的绑定关系
  kind: Role # 此字段必须是 Role
  name: role-template-manage-posts # 此字段必须与你要绑定的 Role 的名称匹配
  apiGroup: halo.run
```

创建了绑定之后，你不能再修改绑定对象所引用的 Role 。 试图改变绑定对象的 `roleRef` 将导致合法性检查错误。 如果你想要改变现有绑定对象中 `roleRef` 字段的内容，必须删除重新创建绑定对象。

#### 对资源的引用 

在 Halo API 中，大多数资源都是使用对象名称的字符串表示来呈现与访问的。 例如，对于 Post 应使用 "posts"。 RBAC 使用对应 API 端点的 URL 中呈现的名字来引用资源。 有一些 Halo API 涉及 **子资源（subresource）**，例如 分类下的文章。 对 根据分类获取文章的请求看起来像这样：

```
GET /api/v1/categories/{name}/posts
```

在这里，`categories` 对应名字空间作用域的 Category 资源，而 `posts` 是 `categories` 的子资源。 在 RBAC 角色表达子资源时，使用斜线（`/`）来分隔资源和子资源。 要允许某主体读取 `categories` 同时访问这些 Category 的 `posts` 子资源，你可以这么写：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: category-and-posts-reader
rules:
- apiGroups: [""]
  resources: ["categories", "categories/posts"]
  verbs: ["get", "list"]
```

#### 聚合Role

你可以将若干 Role **聚合（Aggregate）** 起来，形成一个复合的 Role。

```yaml
apiVersion: halo.run/v1
kind: "Role"
metadata:
  name: role-template-manage-categories
  annotations:
    halo.run/dependencies: ["role-template-view-categories"]
rules:
  - apiGroups: ["halo.run"]
    resources: ["categories"]
    verbs: ["*"]
```

它表示将`role-template-view-categories`角色的规则和当前`role-template-manage-categories`的规则聚合起来形成`role-template-manage-categories`角色的规则。

#### 对主体的引用

RoleBinding 可绑定角色到某 *主体（Subject）* 上。 主体可以是组，用户或者 服务账户。

Halo 用字符串来表示用户名。 用户名可以是普通的用户名，像 "alice"；或者是邮件风格的名称，如 "bob@example.com"， 或者是以字符串形式表达的数字 ID。

> 前缀 `system:` 是系统保留的，所以你要确保 所配置的用户名或者组名不能出现上述 `system:` 前缀。 

RoleBinding 示例 :

下面示例是 `RoleBinding` 中的片段，仅展示其 `subjects` 的部分。

对于名称为 `alice@example.com` 的用户：

```yaml
subjects:
  # 你可以指定不止一个“subject（主体）”
  - kind: User
    name: "alice@example.com"
    apiGroup: halo.run
```

#### 角色模板

系统模块或插件可以通过预设角色来作为模板由管理员通过API或客户端创建角色时绑定，比如文章管理模板创建有两个角色模板，文章管理角色和文章查看角色，而文章管理角色依赖文章查看角色。

如下示例，通过制定`metadata.labels.role-template:true`来指定该角色属于角色模板，那么在客户端角色列表界面将不会展示该角色，而是在权限分配页展示。

```yaml
apiVersion: halo.run/v1
kind: "Role"
metadata:
  name: role-template-manage-categories
  labels:
    halo.run/role-template: true
  annotations:
    halo.run/dependencies: ["role-template-view-posts"]
    halo.run/module: "Posts Management"
    halo.run/alias-name: "Posts Management"
rules:
  - apiGroups: ["halo.run"]
    resources: ["posts"]
    verbs: ["*"]
```



## 考虑的替代方案

 [Generate a Secure Random](https://www.baeldung.com/java-generate-secure-password)

[AES Encryption and Decryption](https://www.baeldung.com/java-aes-encryption-decryption)

[Java Security Standard Algorithm Names](https://docs.oracle.com/en/java/javase/13/docs/specs/security/standard-names.html)

- 通过url匹配的方式去授权
- 用户、角色和权限之间绑定关系的yaml设计
- apiGroups和apiResources的yaml结构

