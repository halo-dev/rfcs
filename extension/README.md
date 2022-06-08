# 自定义模型设计

## 简介

本片文章主要讲解为什么需要设计“自定义模型”，以及“自定义模型”的概要设计。

## 动机

截止 Halo 1.5 的发布，Halo 几乎没有扩展性，插件机制在社区的呼声比较高，比如访问统计需求。插件机制可以非常好的解决自定义需求的问题，且预计在 2.0
中完成插件机制的功能。虽然我们可以提供一些扩展点供插件使用，但是插件的自定义数据需求是必不可少的。例如，访问统计插件需要记录某些文章或者页面的 pv，uv 等，需要将数据持久化到 Halo
中。所以，我们非常有必要设计自定义模型为插件提供数据支持。

## 目标

- 能够任意定义自定义资源，并生成对应的 schema 配置应用到 Halo Core 中。
- 能够方便对自定义资源进行查询，获取，更新和删除操作。
- 监听资源的变更，包括创建，更新。

## 非目标

- 允许多个模型版本共存。
- 允许模型定义者删除或修改字段。
- 通过 YAML 的方式导入自定义模型。
- 开发 haloctl 命令工具。

## 需求

- Halo Core 的模型也能够抽象成自定义模型，如用户，文章，分类，标签，评论等。
- 为插件提供自定义数据支持。比如某插件需要存储自定义数据，同时也想读取和操作自定义数据。

## 提议

在 Halo Core 中提供自定义模型的机制，支持通过预定义的 API 注册自定义模型，提供自定义模型数据访问，操作和验证能力。

## 自定义模型设计

Halo
自定义模型主要参考自 [Kubernetes CRD](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
。自定义模型主要遵循 [OpenAPI v3](https://spec.openapis.org/oas/v3.1.0)。自定义模型的 YAML 模型如下：

```yaml
groupVersion: extensions.halo.run/v1
kind: ExtensionDefinition
metadata:
  # CRD 名称，<名称复数形式.组名>
  name: person.myplugin.johnniang.me
spec:
  names:
    # 组名，主要用于生成 RESTful API： /apis/<组名>/<版本>。
    group: my-plugin.johnniang.me
    # 名称复数形式，用于生成 RESTful API：/apis/<组名>/<版本>/<名称复数形式>
    plural: persons
    # 用于插件注册使用，采用驼峰命名法。
    kind: Person
    # 名称单数形式。
    singular: person
    # 全限定类名
    class: person.my-plugin.johnniang.me.extension.Person
  schema:
    type: object
    properties:
      group:
        type: string
      version:
        type: string
      kind:
        type: string
      metadata:
        $ref: '#/components/schemas/Metadata'
      name:
        maxLength: 100
        type: string
        description: The description on name field
      age:
        maximum: 150
        minimum: 0
        type: integer
        description: The description on age field
        format: int32
      gender:
        type: string
        description: The description on gender field
        enum:
          - MALE
          - FEMALE
  referencedSchemas:
    Metadata:
      type: object
      properties:
        name:
          type: string
        annotations:
          type: object
          additionalProperties:
            type: string
        labels:
          type: object
          additionalProperties:
            type: string
        resourceVersion:
          type: integer
          format: int32
        creationTimestamp:
          type: string
          format: date-time
        updationTimestamp:
          type: string
          format: date-time
        deletionTimestamp:
          type: string
          format: date-time
```

Extension 基本模型：

```java
public interface Extensible {

    GroupVersionKind groupVersionKind();

    void groupVersionKind(GroupVersionKind gvk);

    Metadata metadata();

    void metadata(Metadata metadata);
}

@Data
public abstract class Extension implements Extensible {

    private String group;
    private String version;
    private String kind;

    private Metadata metadata;

    @Override
    public GroupVersionKind groupVersionKind() {
        return new GroupVersionKind(group, version, kind);
    }

    public void groupVersionKind(String group, String version, String kind) {
        this.group = group;
        this.version = version;
        this.kind = kind;
    }

    public void groupVersionKind(GroupVersionKind gvk) {
        this.groupVersionKind(gvk.group(), gvk.version(), gvk.kind());
    }

    public Metadata metadata() {
        return metadata;
    }

    public void metadata(Metadata metadata) {
        this.metadata = metadata;
    }
}

@Data
public final class Metadata {
    private String name;
    private Map<String, String> annotations;
    private Map<String, String> labels;
    private long resourceVersion;
}
```

对应的 Person class 如下：

```java

@Data
@EqualsAndHashCode(callSuper = true)
@ToString(callSuper = true)
@GVK(group = "my-plugin.johnniang.me",
        version = "v1alpha1",
        kind = "Person",
        plural = "persons",
        singular = "person")
public class Person extends Extension {

    @Schema(description = "The description on name field", maxLength = 100)
    private String name;

    @Schema(description = "The description on age field", maximum = "150", minimum = "0")
    private Integer age;

    @Schema(description = "The description on gender field")
    private Gender gender;

    private Person otherPerson;

    public enum Gender {
        MALE, FEMALE,
    }
}
```

对应的 Person 资源样例如下：

```yaml
groupVersion: my-plugin.johnniang.me/v1alpha1
kind: Person
metadata:
  name: johnniang
  annotations:
    my-plugin.johnniang.me/first_name: John
    my-plugin.johnniang.me/last_name: Niang
  resourceVersion: 123
name: johnniang
age: 18
gender: male
```

### Halo Core 设计

注册 Person 到 Halo Core 中：

```java
schema.register(Person.class);
```

此时，可通过 `ExtensionClient` 接口获取和操作数据：

```java
interface ExtensionClient {

    <T extends Extensible> List<T> list(Class<T> type, Predicate<T> criteria);

    <T extends Extensible> Page<T> page(Class<T> type, Predicate<T> criteria, Pageable pageable);

    <T extends Extensible> Optional<T> get(Class<T> type, String name);

    <T extends Extensible> void create(T extension);

    <T extends Extensible> T update(T extension);

    <T extends Extensible> T delete(T extension);
}
```

### Extension 存储设计

关系型数据库设计：

```sql
create table halo
(
    name    varchar(630),
    version bigint,
    value   mediumblob,
    primary key (name)
);

-- 列表查询仅依赖索引的左前缀查询。
create unique index halo_name_idx on halo (name);
```

我们仅需提供几个简单的操作，接口定义如下：

```java
public interface Client {

    List<Value> list(String key);

    List<Value> listByPrefix(String prefix);

    Optional<Value> get(String key);

    void create(String key, byte[] value);

    void update(String key, long version, byte[] value);

    void delete(String key, long version);

    void watch(String prefix, Consumer<ValueChange> consumer);

    record Value(String key, byte[] value, long version) {
    }

    record ValueChange(ChangeEvent event, Value oldValue, Value newValue) {
    }
}

```

### Schema 验证设计

Scheme Validator 接口设计：

```java
interface Validator<T> {

    void onCreate(T extension) throws ValidationException;

    void onUpdate(T extension) throws ValidationException;
}
```

```java
class PersonValidator implements Validator<Person> {

    public void onCreate(Person person) throws ValidationException {
        if (!StringUtils.hasText(person.name)) {
            throw new ValidationException("name required.")
        }
        if (person.age == null) {
            person.age = 0;
        }
    }

    void onUpdate(Person person) throws ValidationException {
        if (person.age == null) {
            person.age = 0;
        }
    }
}
```

注册 Schema Validator：

```java
schema.registerValidator(new PersonValidator());
```

## API 设计

### Halo Core API

核心 API 组成模式为：`/api/<version>/<coreextension>/{extensionname}/<subextension>`。例如：

```shell
GET /api/v1/posts
GET /api/v1/posts/my-post
GET /api/v1/posts/my-post/categories
```

### Extension API

注册 Extension 后，Halo API Server 将会为它生成 Extension
API，组成模式为：`/apis/<group>/<version>/<extension>/{extensionname}/<subextension>`，例如：

```shell
GET /apis/my-plugin.johnniang.me/v1alpha1/persons
GET /apis/my-plugin.johnniang.me/v1alpha1/persons/johnniang
```

## 问题

1. 查询效率问题：如何低内存实现查询？

   目前查询过程都在内存中进行。后续可实现磁盘查询（牺牲时间换空间）。

2. 前端模型如何兼容？

   我们可以创建“前端模型”的自定义模型，用于描述前端模型。

3. 是否支持通过接口创建自定义模型？

   目前暂时不支持。不过设计和实现的过程中考虑一下这种场景，方便未来添加该功能。

4. 如何查询自定义模型数据？

   目前查询的排序和过滤逻辑都需要手动编写。系统可仅对 metadata 和 labels 创建索引进行过滤和排序。

5. 如何注入模型？

    1. 核心模型会默认实现在 Halo 的核心部分；
    2. 其他模型则通过插件机制注册。
