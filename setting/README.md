## 背景和动机

主题和插件的都需要配置管理以用于运行参数的动态调整，例如配置主题的代码展示风格、分页大小、布局等。

Halo 的受众用户可能并不是懂技术的开发者，为了让用户能方便的修改配置需要提供可视化的界面来修改配置的值。

本篇将设计提供一种能让主题或插件开发者定义 `Yaml` 的方式来动态生成配置表单实现可视化修改值的方式，同时提供通过 `ConfigMap` 数据结构来存储这种类型的配置值但 ConfigMap 在设计上不是用来保存大量数据的。

## 目标

- 提供 ConfigMap 自定义模型
- 提供主题和插件配置自定义模型

## 设计

### ConfigMap

ConfigMap 是一个自定义模型， 让你可以存储其他对象所需要使用的配置。 ConfigMap 使用 `data` 和 `binaryData` 字段。这些字段能够接收键-值对作为其取值。`data` 和 `binaryData` 字段都是可选的。`data` 字段设计用来保存 UTF-8 字符串，而 `binaryData` 则被设计用来保存二进制数据作为 base64 编码的字串。

ConfigMap 的名字必须是一个合法的 [DNS 子域名](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/names#dns-subdomain-names)。`data` 或 `binaryData` 字段下面的每个键的名称都必须由字母数字字符或者 `-`、`_` 或 `.` 组成。在 `data` 下保存的键名不可以与在 `binaryData` 下出现的键名有重叠。

ConfigMap 将你的环境配置信息同主题和插件等解耦，便于应用配置的修改。

ConfigMap 并不提供保密或者加密功能。

```yaml
apiVersion: v1alpha1
kind: ConfigMap
metadata:
  name: game-demo
data:
  #...
```

参考：[Kubernetes ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/)

### 主题配置

主题配置示例：

`formSchema` 遵循 [Formkit form generation](https://formkit.com/essentials/generation)

```yaml
apiVersion: v1alpha1
kind: Setting
metadata:
  name: theme-setting-test
spec:
  - group: sns
    label: 社交资料
    formSchema:
      - $el: h1
        children: Register
      - $formkit: text
        help: This will be used for your account.
        label: Email
        name: email
        validation: required|email
      - $formkit: password
        help: Enter your new password.
        label: Password
        name: password
        validation: required|length:5,16
      - $formkit: password
        help: Enter your new password again to confirm it.
        label: Confirm password
        name: password_confirm
        validation: required|confirm
        validationLabel: password confirmation
      - $formkit: checkbox
        id: eu
        label: Are you a european citizen?
        name: eu_citizen
      - $formkit: select
        help: How often should we display a cookie notice?
        if: $get(eu).value
        label: Cookie notice frequency
        name: cookie_notice
        options:
          daily: Every day
          hourly: Ever hour
          refresh: Every page load
```

根据如上 Setting 配置将生成形如以下格式 HTML

```html
<form class="formkit-form" id="input_0" name="form_1">
  <h1>Register</h1>
  <div class="formkit-outer" data-type="text">
    <div class="formkit-wrapper">
      <label class="formkit-label" for="input_1">Email</label>
      <div class="formkit-inner">
        <input
          class="formkit-input"
          type="text"
          name="email"
          id="input_1"
          aria-describedby="help-input_1"
        />
      </div>
    </div>
    <div class="formkit-help" id="help-input_1">
      This will be used for your account.
    </div>
  </div>
  <div class="formkit-outer" data-type="password">
    <div class="formkit-wrapper">
      <label class="formkit-label" for="input_2">Password</label>
      <div class="formkit-inner">
        <input
          class="formkit-input"
          type="password"
          name="password"
          id="input_2"
          aria-describedby="help-input_2"
        />
      </div>
    </div>
    <div class="formkit-help" id="help-input_2">Enter your new password.</div>
  </div>
  <div class="formkit-outer" data-type="password">
    <div class="formkit-wrapper">
      <label class="formkit-label" for="input_3">Confirm password</label>
      <div class="formkit-inner">
        <input
          class="formkit-input"
          type="password"
          name="password_confirm"
          id="input_3"
          aria-describedby="help-input_3"
        />
      </div>
    </div>
    <div class="formkit-help" id="help-input_3">
      Enter your new password again to confirm it.
    </div>
  </div>
  <div class="formkit-outer" data-type="checkbox">
    <label class="formkit-wrapper">
      <div class="formkit-inner">
        <input
          class="formkit-input"
          type="checkbox"
          name="eu_citizen"
          id="eu"
          value="true"
        />
        <span class="formkit-decorator" aria-hidden="true"></span>
      </div>
      <span class="formkit-label">Are you a european citizen?</span>
    </label>
  </div>
  <div class="formkit-actions">
    <div class="formkit-outer" data-type="submit">
      <div class="formkit-wrapper">
        <button
          class="formkit-input"
          type="submit"
          name="submit_2"
          id="input_4"
        >
          Submit
        </button>
      </div>
    </div>
  </div>
</form>
<pre wrap="">
{}
</pre>
```
在主题描述文件中关联 Setting 和 ConfigMap
```yaml
apiVersion: theme.halo.run/v1alpha1
kind: Theme
metadata:
  name: gtvg
spec:
  displayName: GTVG
  author:
    name: guqing
    website: https://guqing.xyz
  description: 测试主题
  logo: https://guqing.xyz/logo.png
  website: https://github.com/guqing/halo-theme-test.git
  repo: https://github.com/guqing/halo-theme-test.git
  version: 1.0.0
  require: 2.0.0
  setting-name: theme-setting-test
  configmap-name: theme-configmap-test
```
当 Setting 名为 `theme-setting-test` 的表单被提交时其表单值会被保存到名称为 `theme-configmap-test` 的 ConfigMap 中。

`ConfigMap` 的格式如下：

```yaml
apiVersion: v1alpha1
kind: ConfigMap
metadata:
  name: theme-configmap-test
data:
  sns: |
    {
      "email": "111",
      "password": "xxx",
      "password_confirm": "xxx",
      "cookie_notice": ["hello", "world"]
    }
  basic: |
    {
      "color": "red"
    }
```
data 中的每个 key 都表示 group name，而 value 为 group 下的表单值 JSON。

### 插件配置

插件配置与主题大致相同

```yaml
apiVersion: v1alpha1
kind: Setting
metadata:
  name: plugin-setting-name
spec:
  - group: default
    label: 默认配置
    formSchema:
      #...
```

配置值同为 ConfigMap

```yaml
apiVersion: v1alpha1
kind: ConfigMap
metadata:
  name: plugin-setting-configmap
data:
  some-group: |
   {}
```
