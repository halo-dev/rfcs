## 背景和动机

主题是一个很好的工具，可以帮助用户建立自己喜欢的门户风格，并使你的网站以最佳方式脱颖而出。它可以按照自己喜欢的方式进行定制。

主题是每个 CMS 网站不可或缺的一部分。除了为你提供多种自定义选项外，它还可以帮助您控制展示网站的方式。

主题会提高 Halo 的适用范围，只需要使用 Halo 进行内容管理并使用不同的主题即可帮助用户快速完成建站，只需要简单的界面点击可以切换门户外观。

## 目标

- 主题开发

- 主题渲染
- 主题安装
- 主题切换
- 主题配置
- 主题国际化
- 主题扩展

## 非目标

- 页面静态化
- 主题更新

## 设计

### 主题开发与工程化

插件结构示例

```text
├── i18n
│   └── zh.properties
│   └── en.properties
├── templates
│   └── assets
      ├── css
      │   └── style.css
      ├── js
      │   └── main.js
│   └── index.html
│   └── post.html
│   └── archives.html
│   └── categories.html
│   └── links.html
│   └── journals.html
├── README.md
└── settings.yaml
└── theme.yaml
```

创建主题描述文件 `theme.yaml`

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
```

创建 `settings.yaml` 用于插件配置表单生成
格式示例如下：

```yaml
apiVersion: theme.halo.run/v1alpha1
kind: Setting
metadata:
  name: theme-setting-${GENERATE_ID}
spec:
  - group: sns
    label: 社交资料
    formSchema:
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
```

目前主题开发仅可通过主题模板仓库生成。

### 主题预览

为了支持在非启用状态下的主题预览，需要考虑链接生成，受影响的主要是静态资源引用链接和模板引擎引用链接。

为了防止模板被下载，静态资源的访问将被限制在`src/assets/**`目录下，即只会对外暴露主题的 `src/assets/` 目录。

并定义两种链接表达式来代替 Thymeleaf 提供的 `@{}` 表达式以便于对启用主题和未启用主题的访问进行处理。

定义静态资源表达式方言如下：

```html
<!-- 语法：@assets{/css/gtvg.css} -->
<link
  rel="stylesheet"
  type="text/css"
  media="all"
  th:href="@assets{/css/gtvg.css}"
/>
```

模板渲染时会将此链接变为

```html
<link
  rel="stylesheet"
  type="text/css"
  media="all"
  href="/themes/{THEME_ID}/assets/css/gtvg.css"
/>
```

对于模板路径

```html
<a th:href="@route{/posts}">Next</a>
```

渲染后，对于激活的主题，实际内容如下

```html
<a href="/posts">Next</a>
```

当开起主题预览时会染内容如下

```html
<a href="/themes/{THEME_ID}/posts">Next</a>
```

从而实现主题预览而无须先启用主题。

### 主题安装

通过上传 `Zip` 压缩文件到 Halo `workdir`的 `themes` 目录或拷贝解压缩后的主题目录到此 `themes` 目录。

文件夹名称应该为主题名称。

### 主题切换

主题同时只能有一个被启用，当修改了正在使用的主题时，模板文件渲染指向的路径切换到被启用的主题目录。

激活的主题挂载到根目录

### 主题配置

管理端通过解析 `settings.yaml `中的配置生成动态表单，填写后点击保存会将所有值保存到一个 ConfigMap 中。示例如下：

```yaml
apiVersion: v1alpha1
kind: ConfigMap
metadata:
  name: halo-theme-gtvg-settings
  labels:
    theme.halo.run/theme-name: THEME_NAME
  	theme.halo.run/theme-setting-name: theme-setting-${GENERATE_ID}
data:
  setting: |
   {
      "sns": {
        "email": "111",
        "password": "xxx",
        "password_confirm": "xxx",
        "cookie_notice": ["hello", "world"]
      }
    }
```

在 `metadata.labels` 中添加对 `theme-name` 和 `theme-settings-name` 的引用是为了便于通过配置的值来反向找到配置定义，便于做数据类型转换等操作。

### 主题国际化

如下目录结构所示

```text
├── i18n
│   └── default.properties
│   └── zh.properties
│   └── en.properties
```

`i18n`目录表示此目录下的 `.properties` 文件多语言配置，`default.properties` 表示默认语言时使用的文件，`en.properties`表示 `Locale` 的语言为 `English` 时使用的配置文件，命名规则为 `{language}.properties`，`language` 遵循 [ISO 639 alpha-2 or alpha-3 language code](https://en.wikipedia.org/wiki/List_of_ISO_639-1_codes)，语言字段不区分大小写，但语言环境总是规范化为小写，`default.properties`除外，它是找不到对应语言环境时默认使用的语言包。

示例："en" (English), "ja" (Japanese), "kok" (Konkani)

### 主题渲染

主题渲染使用 [Thymeleaf](https://www.thymeleaf.org/) 模板引擎渲染。示例：

1. 定义模板视图 APIs 并填充 Context

```java
@Bean
public RouterFunction<ServerResponse> home() {
  return RouterFunctions.route(GET("/posts"),
                               request -> ServerResponse.ok().contentType(MediaType.TEXT_HTML)
                               .bodyValue(storeThemeService.renderPosts()));
}

public String renderPosts(PostParam postParam) {
  List<Post> posts = postService.list(postParam);
  // 获取文章列表
  Map<String, Object> params = Map.of();//...
  params.put("posts", posts);
  params.put("postService", postService);
  return renderService.render("posts", params);
}
```

2. 使用

- 直接使用变量

```html
<div th:each="post in posts">
  <div th:text="${post.name}"></div>
</div>
```

- 调用方法

```html
<div
  th:with="posts=${postService.list(page, size, sort)}"
  th:each="post in posts"
>
  <div th:text="${post.name}"></div>
</div>
```

附录：

```java
public class RenderService {
  	private final TemplateEngine templateEngine;
    private final FileTemplateResolver fileTemplateResolver;

    public RenderService() {
        fileTemplateResolver = fileTemplateResolver();
        templateEngine = new TemplateEngine();
        templateEngine.setTemplateResolver(fileTemplateResolver);
        templateEngine.setDialect(new SpringStandardDialect());
        templateEngine.setMessageResolver(new HaloThemeMessageResolver());
    }

    private FileTemplateResolver fileTemplateResolver() {
        FileTemplateResolver templateResolver = new FileTemplateResolver();
        templateResolver.setCacheable(false);
        templateResolver.setPrefix(getActivateTheme(activateTheme));
        templateResolver.setSuffix(".html");
        return templateResolver;
    }

    public String render(String template, Map<String, Object> model) {
        Context context = new Context(getLocale(), model);
      	// 加一些处理器
        return templateEngine.process(template, context);
    }

    private String getActivateTheme(String theme) {
        return themeBase + theme + "/";
    }

    public Locale getLocale() {}
}
```

#### 主题公共模板扩展

1. 主题中的扩展位只是占位标记，允许渲染时插入固定代码片段类似 [Halo 1.x 公共宏模板](https://docs.halo.run/developer-guide/theme/public-template-tag)。

2. 前期扩展模板只由 Halo 提供，后续可以使用自定义扩展模板内容，例如通过配置的方式使用其他评论模板。

当在主题模版中写了如下 `Tag`

```html
<halo-extension name="comment"></halo-extension>
```

渲染该模版时会获取其中的 `name` 属性表示扩展点名称来选择需要注入到此处的代码。代码片段为固定的内容用来简写公共组件。

示例：

`index.html`

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
  <head>
    <title>测试标题</title>
  </head>
  <body>
    <div>this is a short line</div>
    <halo-extension name="comment"></halo-extension>
  </body>
</html>
```

扩展点对应的`comment` 对应的代码片段为

```html
<div>oh this is comment!</div>
```

则渲染后的结果为:

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
  <head>
    <title>测试标题</title>
  </head>
  <body>
    <div>this is a short line</div>
    <div>oh this is comment!</div>
  </body>
</html>
```

#### 渲染后置处理

允许对渲染后的成品 HTML 内容进行后置处理，例如可以追加模板渲染时间等

```java
// ...
String raw = render(template, model);
return applyTemplateRawPostProcessors(raw);
```

### 页面静态化

页面静态化应该只是一个可选项，允许用户设置是否开启。

静态渲染所带来的问题：

1. 主题开发时渲染不实时

2. 分页问题，添加数据后需要重新生成所有静态页面
3. 浏览量统计不实时
4. 评论显示不实时

为了加快网站的访问速度，可以使用模板渲染缓存来实现。
