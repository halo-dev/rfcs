# Halo 中的安全性草案

### 用户

用户层面，实现多用户功能

用例：

- 允许管理员创建/修改用户
- 允许管理员封禁用户
- 允许访客注册，但管理员可以选择是否开启访客注册，访客可以在前台登录以对文章进行评论

### 角色

角色是对于 Halo 中用户身份属性的一个分类，它的主要作用是简化用户授权，对角色分配权限范围后，再将角色分配给用户，便可轻松实现批量用户的权限分配工作。

角色可以用作用户管理的分类条件，即可以通过角色视图管理用户。

Halo可以默认提供提供三个角色：管理员、普通用户、访客角色，通过外化配置文件的方式配置好角色名称、描述和权限范围。

不采用脚本初始化到数据库的好处是可以很容易的控制默认角色不可以删除，后续版本升级对默认初始化数据的维护也更容易。

### 域

域（scope）是对Halo中系统API资源的描述，该通用语言被限定在身份与访问上下文中。

采用spring的`@PreAuthorize`来定义域，例如：

```java
@PreAuthorize("hasAuthority('create:user')")
public ResponseEntity<String> addUser(@RequestBody UserParam userParam) {
  userService.createUser(userParam);
  return ResponseEntity.ok();
}
```

用户所拥有的一组scopes，表示用户所拥有的权限范围，用户只能在权限范围内操作系统资源。

为了更为直观的让管理员理解域的概念，我们需要一个资源组的抽象

```java
public class ApiResource {
  String name;
  String displayName;
  String description;
  List<ApiScope> scopes;
}
```

```java
public class ApiScope {
  private String name;
  private String displayName;
  private String description;
}
```

`ApiResource`表示一组权限范围，通过对`API`资源的分组可以更方便的对系统模块或功能的权限分配。

它又`name`、`displayName`、`description`和`scopes`组成

- name: 资源组的唯一标识
- displayName： 用于页面展示的，管理员可以理解的名称
- description: 对该资源组职能的描述
- scopes： 表示资源组所拥有的`API`资源

`ApiScope`表示系统`API`资源:

- name：是该`API`资源的唯一标识，用于资源定位和权限检查，它的组成规则为`action:ApiResource.name`，action共有三种：read、write、delete，分别表示了对系统资源的读取、修改、和删除动作。
- displayName：用于在用户界面展示的名称
- description：资源描述。

数据结构`json`示例：

```json
{
  "name": "user",
  "displayName": "用户",
  "description":"授予对用户的完全访问权限",
  "scopes": [{
    "name": "read:user",
    "displayName": "读取用户",
    "description": "授予对系统用户的读访问权限"
  },
  {
    "name": "write:user",
    "displayName": "修改用户",
    "description": "授予对系统用户的创建"
  }]
}
```

用户可以拥有多个`ApiScope`，并使用`scope`属性来承载，例如:

```json
{
  "scope": ["read:user", "write:user"]
}
```

优化：

当用户拥有一整个资源组的权限时可以简化`scope`名为资源组的name，例如将上述`scope`简化为：

```json
{
  "scope": ["user"]
}
```

则表示用户拥有对`name`为`user`的资源组的所有操作权限。

### Token

#### 令牌格式 

- 用户登录成功后统一生成`jwt`格式的`token`，`jwt`使用的密钥由系统安装时生成`RSAKey`到工作目录。
- 管理员可以创建`personal access token `,格式为：`类型前缀_36位短token`例如：`hc_ac5fQe9pERSXRlud3WydzpRVDI4nSh3Iqkcq`

[正如我们从 Slack](https://api.slack.com/authentication/token-types)和[Stripe](https://stripe.com/docs/api/authentication)等公司看到的那样，令牌前缀是使令牌可识别的一种明确方法。`h`表示`halo`，`c`表示令牌类型的第一个字母

- `ha`用于访问`admin api`的令牌
- `hc`用于访问`content api`的令牌

使这些前缀在令牌中清晰可辨，以提高可读性。因此，添加了一个分隔符：`_`并且双击它时，它会可靠地选择整个令牌。

当需要身份验证（例如，对于跨域请求）进行身份验证，便可以使用 `-H "Authorization: Bearer ha_ac5fQe9pERSXRlud3WydzpRVDI4nSh3Iqkcq"`

所有`person access token`颁发时都需要选择`scope`以确保令牌的访问权限在一个可控的范围内。

前端的菜单访问权限由前端根据登录用户的scope自行限制。

#### 认证方式

针对不同的token验证方式可以提供不同的`Provider`而后统一使用`AuthenticationManager`来进行认证，当然这会由`Spring security`帮我们处理，我们只需要根据不同的认证方式创建不同的`Provider`和过滤器即可。

例如：

针对使用用户名密码登录时颁发`jwt token`的过滤器和Provider

- ```
  org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter
  ```

- ```
  org.springframework.security.authentication.dao.DaoAuthenticationProvider
  ```

根据以上类针对`personal access token`提供

- RestrictedApiKeyAuthenticationFilter
- RestrictedApiKeyAuthenticationProvider

优化：

- `Content api`可以不为其指定`scope`即允许访问所有公共的`api`资源，可以称之为`publishable key`

- 一次性令牌也可使用同样的认证方式，例如格式为：`ho_ac5fQe9pERSXRlud3WydzpRVDI4nSh3Iqkcq`，并提供`OttAuthenticationFilter`和`OttAuthenticationProvider`来为一次性令牌策略进行认证。
- 对短令牌使用CRC32进行checksum，可以过滤掉伪造的token的请求，减少查询数据库的次数



![authentication processing filter](https://docs.spring.io/spring-security/reference/_images/servlet/authentication/architecture/abstractauthenticationprocessingfilter.png)

权限分配示例：

![IMAGE 2022-03-14 15:54:49](./assets/158128456-ef984faa-bff6-4b8b-aed0-c92f6ae3ddd8.jpg)