# Halo 身份和访问管理概述

本文档为 Halo 系统中的身份和访问管理提出了一个方向。

## 高层次的目标

- 制定方案来确定如何将身份、授权和认证融入 API
- 轻松与现有企业和托管方案集成

### 背景和动机

安全性是非常重要的服务之一，任何应用程序都需要以一种非常优雅和简单的方式全面处理所有重要部分。

而我们应该从系统开发的一开始就考虑安全性，如果不及早考虑会随着时间的推移产生对软件品质从短期到长期的影响，甚至可能引发其他的系统故障。

## 设计

### 参与者

这些中的每一个都可以充当普通用户或攻击者：

- 外部用户：访问 Halo 应用的人（例如访问 Halo 应用前台，但没有 API 访问权限）
- Halo 访客用户：拥有部分基础功能访问权限的用户（例如通过 Halo 注册成为访客允许用该账号登录以对该系统的文章进行评论）
- Halo 普通用户：访问 Halo API 的人
- Halo 管理员：管理某些 Halo 用户访问权限的人员

### 威胁

故意攻击和意外使用特权都值得关注。

对于这两种情况，以不同的方式考虑这些类别可能很有用：

- 应用程序路径——通过 Halo 暴露出的应用程序输入端向服务器进行攻击
- Halo API 路径 ——通过响任何 Halo 的 API 端点发送网络消息进行攻击
- 内部路径——对 Halo 的系统功能的攻击。攻击者可能拥有访问管理后台或一些API的特权

### 要保护的资产

外部用户资产：

- 用户评论内容、上传的图片和个人信息

Halo 用户资产：

- 上传的图片、文件和个人信息
- Halo 应用私有的东西，例如：
  - 访问系统 API 的凭证
  - 系统 API
  - 系统主题文件
  - 备份文件
  - 插件文件

### 身份

> 一种跟踪 API 参与者的方式

Halo 会有一个 User 对象：

- user 有一个不可变的 id。这用于将用户与对象关联并在审计日志中记录操作
- user 有一个字符串 name，它在 user 中是人类可读且唯一的。它用于系统登录或展示以确保引用到 user 的对象是可读的。邮件地址可以作为此字段的默认值
- user 对象可以有其他属性以更多的标识特征
- user 具有指示组成员身份和担任某些角色的能力标签

系统可以将一个或多个身份验证方法应用到一个 user，但他们不是 user 的正式组成部分。

初始特点：

- 默认有一个超级用户 user

### 身份认证

> 如何确定和检查参与者的身份

认证的目标：

- 包括一个内置身份验证系统
- 避免混合认证和授权，以便集中在那里授权策略，并允许在不影响授权代码的情况下更改认证方法

初始阶段：

- 设计用于对用户进行身份验证的令牌
- 长期存在的令牌表示特定于 user
- 管理员可以在系统创建令牌
- OAuth2.0 Bearer Token 协议 <https://datatracker.ietf.org/doc/html/rfc6749#section-1.4>
- Token 具有 scopes，认证发生在 Halo 主应用
- 由 Halo 主应用生成令牌，用于识别 API 调用
- 在 Halo 中检查令牌
- 身份验证可以通过标志禁用，允许在未授权的情况下进行测试

改进：

- refresh token
- 允许外部系统处理身份验证，以允许与现有的企业授权系统集成。

### 授权

> 身份可以在哪个上下文中执行哪些操作

Halo 授权应该：

- 通过有明确语义的 scope 分配策略在进行系统授权
- 简化操作路径，必要时尽量通过对 scope 的分配来生成对后台菜单资源的访问策略，从而避免还要再次对菜单资源授权
- 允许集中管理用户和授权
- 尽可能与身份验证分开，以允许身份验证方法岁时间和空间变化，而不会影响授权策略
- 将授权公开为 API 对象，以便可以由管理员添加 scope，进而为自定义 API 提供授权服务

Halo 将实现基于 scope 的访问控制模型，为了简化批量用户的权限分配系统带有角色功能。

### 审计日志

可以记录 API 操作。

初步实施：

- 所有 API 调用都记录到 `access.log` 中

改进：

- 将现有的操作日志使用同一个 `access.log` 来完成，不在存储到数据库
- 提供日志的删除策略
  - 高速受信 API 调用的记录策略
  - 受信用户的访问记录
  - 其他敏感功能的访问记录
