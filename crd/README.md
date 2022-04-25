# 自定义模型设计

## 简介

本片文章主要讲解为什么需要设计“自定义模型”，以及“自定义模型”的概要设计。

## 动机

截止 Halo 1.5 的发布，Halo 几乎没有扩展性，插件机制在社区的呼声比较高，比如访问统计需求。插件机制可以非常好的解决自定义需求的问题，且预计在 2.0 中完成插件机制的功能。虽然我们可以提供一些扩展点供插件使用，但是插件的自定义数据需求是必不可少的。例如，访问统计插件需要记录某些文章或者页面的 pv，uv 等，需要将数据持久化到 Halo 中。所以，我们非常有必要设计自定义模型为插件提供数据支持。

### 目标

- 能够任意定义自定义资源，并生成对应的 schema 配置应用到 Halo Core 中。
- 能够方便对自定义资源进行查询，获取，更新和删除操作。
- 监听资源的变更，包括创建，更新。

### 非目标

- 允许多个模型版本共存
- 允许模型定义者删除或修改字段

## 需求

- Halo Core 的模型也能够抽象成自定义模型，如用户，文章，分类，标签，评论等。
- 为插件提供自定义数据支持。比如某插件需要存储自定义数据，同时也想读取和操作自定义数据。

## 提议

在 Halo Core 中提供自定义模型的机制，支持通过 YAML 的方式导入自定义模型，提供自定义模型数据访问和操作的能力。

## 自定义模型设计

Halo 自定义模型主要参考自 [Kubernetes CRD](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)。自定义模型主要遵循 [OpenAPI v3](https://spec.openapis.org/oas/v3.1.0)。

```yaml
version: extensions.halo.run/v1
kind: ExtensionDefinition
metadata:
  # CRD 名称，<名称复数形式.组名>
  name: person.my-plugin.johnniang.me
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
  versions:
    - name: v1
      # 标识当前版本用于存储，有且只能有一个版本才能设置 storage=true。
      storage: true
      # 这里的 schema 主要参考 OpenAPI v3 规范，后续将用于模型数据的合法性验证。
      schema:
        type: object
        properties:
          name:
            type: string
            description: First and Last name
            minLength: 4
          age:
            type: integer
            default: 21
            minimum: 18
            maximum: 99
          gender:
            type: string
            enum:
              - male
              - female
```

用户可通过 haloctl 添加一个该 ExtensionDefinition:

```shell
haloctl apply -f my-plugin.johnniang.me_person.yaml
```

Extension 基本模型：

```java
public final class Metadata(
  private String name;
  private Map<String, String> annotations;
  private Map<String, String> labels;
  private long resourceVersion;
)

abstract class Extension {
  private String version;
  private String kind,
  private String group,
  private Metadata metadata,
}
```

对应的 Person class 如下：

```java
@GVK(group="my-plugin.johnniang.me", version="v1alpha1", kind="Person")
record Person(
  String name,
  Integer age,
  Gender gender,
) extends Extension
```

对应的 Person 资源样例如下：

```yaml
group: my-plugin.johnniang.me
version: v1alpha1
kind: Person
metadata:
  name: johnniang
  annotations:
    my-plugin.johnniang.me/first_name: John
    my-plugin.johnniang.me/last_name: Niang
  resourceId:
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

  List<T> list(Class<T> kind, Predicate<T> criteria);

  Page<T> page(Class<T> kind, Pageable pageable, Predicate<T> criteria);

  T get(Class<T> kind, String name);

  T create(Class<T> kind,T extension);

  T update(Class<T>, kind, T extension);

  T patch(Class<T> kind, T extension);

  T delete(T extension);
}
```

### Extension 存储设计

关系型数据库设计：

```sql
create table if not exists halo (
  id integer auto_increment,
  name varchar(630), -- byte[]
  -- created bool,
  deleted bool,
  version int,
  value mediumblob, -- byte[]
  old_value mediumblob,
  primary key(id)
);

create index halo_name_idx on halo (name);
create index halo_name_id_idx on halo(name, id);
create index halo_id_deleted_idx on halo(deleted, id);
create unique index halo_name_prev_revision_idx on halo(name);
```

Value 的数据结构如下：

```java
record Value (byte[] key, byte[] value, long revision)
```

我们仅需提供几个简单的操作：

```java
interface Client {
  List<Value> list(String key, int rev);
  Value get(String key);
  void put(String key, byte[] value);
  void create(String key, byte[] value);
  void update(String key, long revision, byte[] value);
  void delete(String key, long revision);
  void watch(String key, Consumer<ValueChange> consumer);
}
```

> 问题：根据 category id 查询所有的 posts。
> 树形查询
> Prefix 查询功能考虑不同数据库的兼容
> 根据 label 进行查询

### Shema 验证设计

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
  }
  void onUpdate(Person person) throws ValidationException {
  }
}
```

注册 Schema Validator：

```java
schema.registerValidator(new PersonValidator());
```

### Extension Watch 设计

TBD.

### API 设计

#### Halo Core API

核心 API 组成模式为：`/apis/<version>/<coreextension>/{extensionname}/<subextension>`。例如：

```shell
GET /apis/v1/posts
GET /apis/v1/posts/my-post
GET /apis/v1/posts/my-post/categories
```

#### Extension API

注册 Extension 后，Halo API Server 将会为它生成 Extension API，组成模式为：`/apis/<group>/<version>/<extension>/{extensionname}/<subextension>`，例如：

```shell
GET /apis/my-plugin.johnniang.me/v1alpha1/persons
GET /apis/my-plugin.johnniang.me/v1alpha1/persons/johnniang
```

## 问题

TBD.

- Validator 调用时机
- POC
- 低内存实现查询

- 多版本问题

如果只是新增字段，可不用添加版本
但是删除字段或者修改字段需要手写转换方法。
我们定义好某一个版本存储到数据库中。其他版本围绕它写相互转换方法即可。

类似这样的模型：

v2(Spoker) <--> (v1)Hub <--> v3(Spoker)
^
|
v4(Spoker)

v1 和 v2 版本实体是否需要 copy 一份
v1 和 v2 版本转换问题


暂时不考虑多版本问题，太过于复杂，等待后续设计。
暂时不考虑修改或删除字段