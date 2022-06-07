## 插件化功能设计

实现 Halo 插件化系统，以便对核心功能进行扩展，在不缺失主要功能的同时防止 core 过大。插件能力有助于社区生态的构建。

### 目标

后端插件化

- 后端支持 API 拓展机制，提供统一的 API 聚合，通过 core 的访问控制模块（IAM） 进行统一鉴权。
- 支持扩展点机制，插件通过实现 core 中暴露的扩展点接口来增强 core 的能力，如扩展附件存储提供者。
- 插件允许通过 core 提供的数据持久化机制来进行数据操作（CRUD）。
- 插件允许调用 core 提供的公开接口对 core 的数据进行操作。

前端插件化

- 前端项目支持插件化，可通过插件在各级导航栏插入新的功能入口，实现功能页面的动态添加。

- 通过固定协议加载插件中提供的前端页面或 JavaScript 来扩展前端功能。

公共目标

- 插件管理：提供可视化的插件管理机制，支持插件的安装、卸载、启用、停用、配置、升级。
- 插件仓库：提供插件的打包、发布机制，提供内置的插件仓库。
- 插件框架：提供插件开发、打包、发布相关的脚手架，提供完善的插件开发文档。

### 非目标

- 插件安全检查。
- 插件代码风格检查。

### 背景和动机

目前 1.0 版本中社区提出了很多非普适功能的 issue，为了控制 core 的大小会挑选重要的功能进行实现， 这对使用者来说很不方便。
包括但不限于以下 issue：

- [期望增加微信公众号管理功能](https://github.com/halo-dev/halo/issues/1990)
- [希望可以添加对 UML 渲染支持](https://github.com/halo-dev/halo/issues/1419)
- [私有文章功能增强。附件功能增强。文章 url 混淆](https://github.com/halo-dev/halo/issues/994)
- [集成 umami 到 halo](https://github.com/halo-dev/halo/issues/1285)

如果增加了插件化能力，社区可以根据自己的需要进行插件开发以扩展核心功能，使用者也可以在插件仓库查找自己需要的插件来满足功能。

该功能的实现会很大程度的提高社区活跃度并壮大社区，同时很多在 1.0 版本中加入的许多功能都可以抽取为独立插件以减小 core 的大小，用户可以根据需要选择合适的插件来达到目的。

### 设计

#### 术语

- **Extension Point**
  - 由 Halo 定义的用于添加特定功能的接口。
  - 扩展点应该在服务的核心功能和它所认为的集成之间的交叉点上。
  - 扩展点是对服务的扩充，但不是影响服务的核心功能：区别在于，如果没有其核心功能，服务就无法运行，而扩展点对于特定的配置可能至关重要该服务最终是可选的。
  - 扩展点应该小且可组合，并且在相互配合使用时，可为 Halo 提供比其各部分总和更大的价值。
- **Extension**
  - Extension Point（扩展点）的一种具体实现。

#### Backend

##### 描述

插件启用时由 PluginManager 负责加载，包括 :

- APIs: 委托给 Request mapping registrar 管理。
- Extension Point：委托给 Extension Finder 管理。
- Static files：由 PluginClassLoader 加载。
- 类似 manifest 和 role template 的 yaml。
- Listeners：由 PluginApplicationContext 管理。
- Spring bean components：委托给 PluginApplicationContext 管理。
- core 中标注了 `@SharedEvent` 注解的事件被发布时由 `PluginApplicationEventBridgeDispatcher` 桥接给已启用的插件使用。

![image-20220507180723198](assets/image-20220507180723198.png)

当插件被启用时由 `PluginManager` 创建一个新的 `PluginClassLoader` 实例负责加载插件类和资源，加载顺序符合双亲委派机制（见图 Figure 2–1 Class Loader Runtime Hierarchy）
`PluginClassLoader` 的 `parent` 为 Halo 使用的类加载器，因此它能够访问 Halo 中已被加载的所有类。

![Figure 2–1 Class Loader Runtime Hierarchy](https://docs.oracle.com/cd/E19501-01/819-3659/images/dgdeploy2.gif)

> 参考链接：
>
> [Chapter 5. Loading, Linking, and Initializing](https://docs.oracle.com/javase/specs/jvms/se17/html/jvms-5.html)
>
> [Chapter 12. Execution](https://docs.oracle.com/javase/specs/jls/se17/html/jls-12.html#jls-12.6)
>
> [Class Loader API Doc](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/ClassLoader.html)
>
> [Oracle Chapter 2 Class Loaders](https://docs.oracle.com/cd/E19501-01/819-3659/beade/index.html)

##### 资源配置

**plugin-manifest**

```yaml
apiVersion: v1
kind: Plugin
metadata:
  # 'name' must match the filename of the manifest. The name defines how
  # the plugin is invoked
  name: plugin-1
  labels:
    extensions.guqing.xyz/category: attachment
spec:
  # 'version' is a valid semantic version string (see semver.org).
  version: 0.0.1
  requires: ">=2.0.0"
  author: guqing
  logo: https://guqing.xyz/avatar
  pluginClass: xyz.guqing.plugin.potatoes.PotatoesApp
  pluginDependencies:
   "plugin-2": 1.0.0
  # 'homepage' usually links to the GitHub repository of the plugin
  homepage: https://github.com/guqing/halo-plugin-1
  # 'displayName' explains what the plugin does in only a few words
  displayName: "a name to show"
  description: "Tell me more about this plugin."
  license: MIT
```

- `version`: 指定当前插件版本号，规则参考[插件版本控制](#plugin-versioning)
- `requires: >=2.0.0` 表示 halo 系统版本必须大于 2.0.0，支持使用`>`, `<`, `=`, `>=`or`<=`进行比较，或`-`指定包含范围，可以用来`||`结合

> 例如：
>
> requires: 2.1
>
> requires: 1.0.0 - 1.2.0 (连字符两边必须有空格)
>
> requires: >2.0.x
>
> requires: <=2.0.x || >2.2.x

- `author`：插件作者名称。
- `logo`：插件的 logo。
- `pluginClass` ： 继承了`run.halo.app.extensions.SpringPlugin`的类全限定名，用于干预插件生命周期和类扫描。
- `pluginDependencies`: 如果依赖了其他插件则使用`pluginId:version`的格式[可选]。
- `homepage`: 插件的主页[可选]。
- `displayName`:插件的显示名称。
- `description`: 详细介绍[可选]。
- `license`：插件遵循的软件协议[可选]。

**plugin role templates**

如果需要对插件提供的 API 进行权限控制，可以定义 role template，当插件启用时会被加载以使用 Halo 的权限控制体系进行统一的 API 权限控制。

```yaml
apiVersion: v1
kind: Role
metadata:
  name: role-manage-plugin-apis
  labels:
    guqing.xyz/role-template: true
  annotations:
    guqing.xyz/dependencies: ["role-template-view-plugin-apis"]
    guqing.xyz/module: "Test Plugin"
    guqing.xyz/alias-name: "Test Plugin"
rules:
  - apiGroups: ["plugin1.guqing.xyz"]
    resources: ["plugin-tests"]
    verbs: ["*"]
```

##### Extension Point 插件

Halo 使用 [Java 插件框架 (PF4J)](https://github.com/pf4j/pf4j) 来表示服务的 **扩展点** 接口。您可以创建一个插件来实现扩展点中声明的方法。基于扩展点创建插件有很多优点：

- 这是最简单的 - 使用 `@Extension` 注解并实现扩展点中声明的方法。
- Halo 将插件加载到隔离的类路径中。
- 它的维护工作量最少。
- Halo 的更新不太可能破坏你的插件。

这里有一个 [PoC](https://github.com/guqing/halo-plugin-experimental/tree/main/core/src/main/java/run/halo/app/extensions)可供预览

##### 定义 Extension Point

扩展点接口必须继承 `ExtensionPoint` 以表示该接口为扩展点

```java
public interface FileHandler extends ExtensionPoint {

  UploadResult upload(@NonNull MultipartFile file);

  void delete(@NonNull String key);

  boolean supports(@NonNull FileHandler handler);
}
```

为了安全性考虑，不能让插件调用 core 中所有方法或 APIs，而是单独提供一个工具包，其中包含了插件可使用的 interface，这里称之为: `pluggable-suite`，插件依赖 `pluggable-suite` 后实现其中的某些扩展点或调用一些方法来制作插件。

core 中会对已经定义的可扩展的代码使用 ExtensionPointFinder 来查找扩展点，当实现了指定扩展点的插件被启用时就会被发现，结果是一个有序集合。

```java
@Slf4j
@Component
public class FileHandlers {

  private final ExtensionComponentsFinder extensionComponentsFinder;

  public FileHandlers(ExtensionComponentsFinder extensionComponentsFinder) {
    this.extensionComponentsFinder = extensionComponentsFinder;
  }

  @NonNull
  public UploadResult upload(@NonNull MultipartFile file,
                             @NonNull AttachmentType attachmentType) {
    return getSupportedType(attachmentType).upload(file);
  }

  public void delete(@NonNull Attachment attachment) {
    Assert.notNull(attachment, "Attachment must not be null");
    getSupportedType(attachment.getType())
      .delete(attachment.getFileKey());
  }

  public boolean supports(FileHandler fileHandler) {
    AttachmentType attachmentType = optionService.getEnumByPropertyOrDefault(
      AttachmentProperties.ATTACHMENT_TYPE, AttachmentType.class, AttachmentType.LOCAL);
    return instance.getAttachmentType() == attachmentType;
  }

  private FileHandler getSupportedType(AttachmentType type) {
    ExtensionList<FileHandler> extensionList = extensionComponentsFinder.lookup(FileHandler.class);

    for (ExtensionComponent<FileHandler> extensionComponent : extensionList.getComponents()) {
      FileHandler instance = extensionComponent.getInstance();
      if (supports(instance.getAttachmentType())) {
        log.info("Used {} file handler(s)", instance);
        return instance;
      }
    }
    throw new FileOperationException("No available file handlers to operate the file")
      .setErrorData(type);
  }
}
```

##### 生命周期方法

通过继承`Plugin`来表示该类为插件的主类，加载时会从此类开始并扫描此类同级目录及子目录下的类，将其加载到 Halo 中。

它具有`start`、`stop`、`delete`生命周期方法，分别会在插件启动、停止和卸载时被调用。

```java
public class PotatoesApp extends Plugin {

    public PotatoesApp(PluginWrapper wrapper) {
        super(wrapper);
    }

    @Override
    public void start() {
        super.start();
    }

    @Override
    public void stop() {
        super.stop();
    }

    @Override
    public void delete() {
        super.delete();
    }
}
```

##### 定义 APIs

使用`@RestController`和`@Controller`来声明是插件的控制器

```java
@RestController
@RequestMapping("/plugins/potatoes")
public class PotatoesController {

    @Autowired
    private PotatoService potatoService;

    @GetMapping("/name")
    public String name() {
        potatoService.create();
        return "Lycopersicon esculentum";
    }

    @GetMapping("/boom")
    public String boom() {
        return String.valueOf(1 / 0);
    }
}
```

##### 发布订阅

1. 插件内部允许存在独立的事件和监听器

```java
@Slf4j
@Component
public class PotatoesVisitListener {

    @EventListener(PotatoesVisitEvent.class)
    public void onPluginStarted(PotatoesVisitEvent event) {
        log.info("Potato visited event: {}", event);
    }
}
```

2. 插件监听 Halo 中发布的事件

当 Halo 中标注了 `@SharedEvent` 注解的事件被发布时插件中可以使用同样的方式监听到

例如：

```java
@SharedEvent
public class PostVisitEvent extends ApplicationEvent {
  private final Integer id;
  public PostVisitEvent(Object source, @NonNull Integer postId) {
    super(source);

    Assert.notNull(id, "The postId must not be null.");
    this.id = id;
  }

  @NonNull
  public Integer getId() {
    return id;
  }
}
```

插件中监听它，你可以使用两种方式

1. 使用 `@EventListener`注解监听事件

```java
@Slf4j
@Component
public class HaloEventListener {

    @EventListener(PostVisitEvent.class)
    public void onPostVisited(PostVisitEvent event) {
        log.info("Post visited event listener: {}", event);
    }
}
```

2. 实现 `ApplicationListener` 指定范型来表明具体要监听的事件

```java
@Component
public class HaloPostVisitListener implements ApplicationListener<PostVisitEvent> {
    @Override
    public void onApplicationEvent(PostVisitEvent event) {
        System.out.println("The posts was visited...");
    }
}
```

##### 数据持久化

通过自定义模型来完成插件数据持久化功能。
TODO 细节待补充

##### 插件版本控制 <a id="plugin-versioning"></a>

为了保持 Halo 生态系统的健康、可靠和安全，每次您对自己拥有的插件进行重大更新时，我们建议在遵循 [semantic versioning spec](http://semver.org/) 的基础上，发布新版本。遵循语义版本控制规范有助于其他依赖你代码的开发人员了解给定版本的更改程度，并在必要时调整自己的代码。

我们建议你的包版本从`1.0.0`开始并递增，如下：

| Code status                               | Stage         | Rule                                         | Example version |
| ----------------------------------------- | ------------- | -------------------------------------------- | --------------- |
| First release                             | New product   | 从 1.0.0 开始                                | 1.0.0           |
| Backward compatible bug fixes             | Patch release | 增加第三位数字                               | 1.0.1           |
| Backward compatible new features          | Minor release | 增加中间数字并将最后一位重置为零             | 1.1.0           |
| Changes that break backward compatibility | Major release | 增加第一位数字并将中间和最后一位数字重置为零 | 2.0.0           |

##### 插件依赖插件

MVP(minimum viable product)版本中不实现

TBD

##### 插件版本更新

MVP(minimum viable product)版本中不实现（可先通过先卸载后安装的方式解决）

TBD

##### 插件工程化

插件可以使用 Maven 或 Gradle 等项目构建工具依赖 `pluggable-suite`，该工具中提供了扩展点接口、公共接口和一些工具帮助快速构建插件。

一个常见的使用 Gradle 作为构建工具的插件目录结构如下

```
apples
├── build
│   ├── classes
│   │   └── java
│   │       └── main
│   │           ├── META-INF
│   │           │   └── plugin-components.idx
│   ├── libs
│   │   └── apples-1.0.0.jar
└── src
    └── main
        ├── java
        │   └── xyz
        │       └── guqing
        │           └── plugin
        │               └── apples
        │                   ├── ApplesPlugin.java
        └── resources
            ├── plugin.yaml
            ├── roleTemplate.yaml
            └── index.html
```

插件可以引入 `pluggable-suite` 中没有提供的依赖，例如使用 `Gradle` 作为项目构建工具时，单独在插件中引入 `commons-lang3` 示例：

```java
implementation "org.apache.commons:commons-lang3:3.10"
```

需要注意的是：如果 Halo 已经存在了相同`group:name`的依赖，那么插件再引入该依赖即使版本不同，也会以 Halo 中的为准。
Reason: 根据 [描述](#描述)中关于类加载的说明，插件使用的 `PluginClassLoader` 的 `parent` 为 Halo 的类加载器且插件类加载规则符合双亲委派机制，
所以 Halo 中已加载的类对插件是可见的，那么插件类加载时 Halo 中存在的类就[不会被重复加载](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/ClassLoader.html)。

> - 每个类加载器都有自己的命名空间，命名空间由加载该类的加载器及所有父加载器所加载的类组成，
>   因此各插件和 Halo 所属同一个命名空间，但插件与插件之间属于不同的命名空间。
> - 在同一个命名空间中，不会出现类的完整名字（包括类的包名） 相同的两个类。
> - 在不同的命名空间中，有可能会出现类的完整名字（包括类的包名）相同的两个类。

**如何开发一个插件**

TBD.

插件如何调试

##### Halo 可扩展功能

1. 附件上传的方式可以默认提供本地文件上传功能，然后通过插件扩展其他上传方式如 OSS。
2. 针对文章、评论、上传的文件流对象等提供可扩展的对象前置和后置处理器扩展点，可实现例如数据脱敏、文件去除 EXIF 元信息等功能。
3. 独立页面功能抽取出去通过插件实现，例如友情链接、图库、日志页面通过插件实现。
4. 可通过插件替换不同的编辑器类型，例如 Markdown 编辑器、富文本编辑器。
5. 缓存策略可扩展，默认实现 InMemeryCache 插件可扩展 Redis 等缓存方式。
6. 认证方式可扩展，默认提供用户名密码认证方式，可扩展手机号登录、邮件登录、三方认证等。
7. 搜索功能可扩展，默认实现内存级搜索方式，可使用插件扩展为 Elasticsearch 等使用外部搜索引擎。
8. 主题可使用插件来对渲染后的内容插入全局 JavaScript 或 CSS 实现比如图片点击预览，看板娘等功能。
9. SEO 插件，例如通过插件对渲染后内容 Header 中插入 openGraph 标签等。
10. 通知方式可扩展，例如文章被评论时可默认选择邮件通知，通过插件扩展其他通知方式如短信、telegram-bot 等。
11. 静态存储可通过插件扩展，上传了文件后将该文件的访问路径注册到 request mapping 中。
12. 插件实现资源监控和告警等功能。
13. 系统日志功能通过插件实现。
14. 插件实现小工具，如数据备份，导入导出 Markdown 或整站、导入导出 json 等。

**如何调试**

TBD.

#### Frontend

TBD

#### 附录

##### 插件启动速度优化

插件启动时扫描插件类所需耗时会随着插件包层次结构的复杂度而增加，例如

```markdown
## Reading extensions storages from classpath 308ms -> StopWatch '': running time = 308620936 ns

## ns % Task name

308620936 100% readClasspathStorages

## Reading extensions storages from plugins 403ms -> StopWatch '': running time = 403391485 ns

## ns % Task name

403391485 100% readPluginsStorages
```

总计：711ms

而如果将这个扫描的过程前置到插件打包时通过在 `pluggable-suite` 中提供一个`PluggableAnnotationProcessor`处理器提前将类扫描好生成索引文件`META-INFO/spring-components.idx`，插件启动时直接读取索引文件进行操作，这样可以在插件被启用时快速完成初始化。

优化后（26ms）：

```markdown
## total millis: 26ms ->StopWatch 'findCandidateComponents': running time = 26146095 ns

## ns % Task name

023519087 090% getExtensionClassNames
000607725 002% loadClass
001025490 004% loadClass
000718673 003% loadClass
000275120 001% loadClass
```

##### 插件卸载

插件卸载时要:

1. 插件 Class 对象不再被引用，即不可触及（没有引用指向）时
2. 断开插件 Class 实例与 PluginClassLoader 之间的双向关联关系

有效性验证:

1. 启动 `Halo` 时添加 `JVM` 参数

```
-Xlog:class+load=info -Xlog:class+unload=info
```

2. 以启用 apples 插件为例，使用 `jconsole` 连接到 Halo 线程，观察 apples 启用前后和卸载前后的 Class load 和 unload 信息

```shell
# 启动前 jconsole class details
已加装当前类: 17,671
已加载类总数: 17,777
已卸载类总数: 100

# 启动 apples 插件，查看 class+load jvm info
[681.850s][info][class,load  ] xyz.guqing.plugin.apples.controller.ApplesController source: file:/Users/guqing/Develop/workspace/halo-plugin-experimental/plugins/apples/build/classes/java/main/
[681.851s][info][class,load  ] xyz.guqing.plugin.apples.service.impl.TestExtImpl source: file:/Users/guqing/Develop/workspace/halo-plugin-experimental/plugins/apples/build/classes/java/main/
[681.851s][info][class,load  ] xyz.guqing.plugin.apples.service.AppleService source: file:/Users/guqing/Develop/workspace/halo-plugin-experimental/plugins/apples/build/classes/java/main/
[681.851s][info][class,load  ] xyz.guqing.plugin.apples.service.impl.AppleServiceImpl source: file:/Users/guqing/Develop/workspace/halo-plugin-experimental/plugins/apples/build/classes/java/main/
[681.851s][info][class,load  ] xyz.guqing.plugin.apples.controller.ApplesSimpleController source: file:/Users/guqing/Develop/workspace/halo-plugin-experimental/plugins/apples/build/classes/java/main/
[681.856s][info][class,load  ] xyz.guqing.plugin.apples.ApplesPlugin source: file:/Users/guqing/Develop/workspace/halo-plugin-experimental/plugins/apples/build/classes/java/main/

# 启动后 jconsole class details
已加装当前类: 17,677
已加载类总数: 17,777
已卸载类总数: 100

# 卸载 apples 插件后执行一次 gc，查看 class+unload jvm info
[716.579s][info][class,unload] unloading class xyz.guqing.plugin.apples.ApplesPlugin 0x0000000800d2f478
[716.579s][info][class,unload] unloading class xyz.guqing.plugin.apples.controller.ApplesSimpleController 0x0000000800d2f268
[716.579s][info][class,unload] unloading class xyz.guqing.plugin.apples.service.impl.AppleServiceImpl 0x0000000800d2f040
[716.579s][info][class,unload] unloading class xyz.guqing.plugin.apples.service.AppleService 0x0000000800d2fcb8
[716.579s][info][class,unload] unloading class xyz.guqing.plugin.apples.service.impl.TestExtImpl 0x0000000800d2fa70
[716.579s][info][class,unload] unloading class xyz.guqing.plugin.apples.controller.ApplesController 0x0000000800d2f840

# 卸载后 jconsole class details
已加装当前类: 17,671
已加载类总数: 17,777
已卸载类总数: 106
```
