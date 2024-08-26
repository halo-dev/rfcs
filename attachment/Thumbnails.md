## 背景

在当前的 Halo 系统中，附件管理功能虽然支持多种存储策略（如本地存储和通过插件扩展的 S3 协议存储），但缺乏缩略图生成功能。这导致在管理端上传大尺寸图片时，页面加载速度显著降低。

具体来说，在管理后台的附件管理界面中，由于图片文件往往较大，浏览器需要加载原始尺寸的图片，这会导致页面加载时间过长，影响管理效率。同样地，在主题端，图片加载缓慢也会显著影响用户体验，尤其是在网络状况不佳或访问量较大的情况下，这种问题会更加突出。

为了提升图片加载性能，缩略图功能显得尤为重要。缩略图不仅可以显著减少管理端加载图片的时间，还可以让主题端在展示图片时，根据不同设备和网络情况加载合适尺寸的图片，提升整体访问速度和用户体验。因此，引入一个灵活且可扩展的缩略图生成机制，支持不同存储策略下的缩略图生成与管理，成为当前 Halo 系统优化的关键。

## 已有需求

用户和开发者对于附件管理的需求日益增加，尤其是在高效管理和展示图片资源方面。以下是系统中已经存在的一些需求，这些需求促使我们考虑引入缩略图功能：

1. **附件展示问题**: 在管理后台中，图片仅有原图的访问地址，这导致图片列表加载时非常缓慢，影响页面流畅度 ​ ([halo-dev/halo#3167](https://github.com/halo-dev/halo/issues/3167))
2. **渲染性能问题**: 多个 issue 提到在附件库中，加载大量大图会导致页面卡顿，甚至出现浏览器崩溃的情况，这凸显了对缩略图的迫切需求 ​ ([halo-dev/halo#3830](https://github.com/halo-dev/halo/issues/3830))​ ([halo-dev/halo#2387](https://github.com/halo-dev/halo/issues/2387))
3. **用户体验问题**: 当用户上传过大的图片时，访问附件库和前端页面时加载速度缓慢，影响了整体用户体验 ​ ([halo-dev/halo#5196](https://github.com/halo-dev/halo/issues/5196))

由于 Halo 中缺乏缩略图功能，导致在管理和展示图片时出现性能瓶颈。为了解决这个问题，Halo 被迫进行了一些功能上的调整：

- **使用 CSS3 硬件加速**：通过利用 CSS3 的硬件加速特性，优化了附件库中图片的渲染性能，试图减轻大尺寸图片带来的加载压力。([halo-dev/halo#3831](https://github.com/halo-dev/halo/pull/3831))
- **调整默认排版模式**：为了减少界面渲染的负担，附件管理界面的默认排版模式从缩略图视图调整为列表模式，以减少页面加载时间。([halo-dev/console#827](https://github.com/halo-dev/console/pull/827))

这些调整在一定程度上缓解了由于缺少缩略图功能而带来的在附件管理上的性能问题，但它们并不能根本上解决问题。因此，缩略图功能的引入仍然是满足当前系统需求的关键所在。

## 目标

本 RFC 的主要目标是在 Halo 博客平台中引入一个灵活且可扩展的缩略图生成功能，以解决目前由于缺乏缩略图功能而导致的性能问题和用户体验下降。具体目标如下：

1. **提升系统性能**：

   - 通过生成和使用缩略图，减少管理端和主题端加载大尺寸图片时的网络带宽占用和页面渲染时间，从而显著提升系统的整体性能。

2. **优化用户体验**：

   - 在主题端使用缩略图替代原始图片，以加快页面加载速度，提升用户浏览体验，减少因图片加载慢而导致的页面卡顿或延迟问题。

3. **支持多样化的存储策略**：

   - 为不同的存储策略（如本地存储、S3 协议等）提供灵活的缩略图生成和调用机制。针对本地存储，系统将自动生成并存储缩略图；针对支持参数化缩略图生成的云存储，允许实现者根据配置动态生成缩略图 URL。

4. **提供可扩展的开发接口**：

   - 设计并实现一个扩展点，使开发者能够根据具体的存储需求和策略自定义缩略图生成逻辑。通过这个扩展点，开发者可以轻松地集成自定义的缩略图生成插件，以满足不同的业务需求。

5. **提高管理效率**：

   - 在管理后台，使用缩略图替代大尺寸图片，提升附件管理界面的加载和操作速度，使管理者能够更高效地浏览和处理附件资源。

6. **保持兼容性与灵活性**：
   - 在引入缩略图功能的同时，确保与现有的附件管理和存储机制保持兼容，并为未来的功能扩展预留灵活性。

通过实现这些目标，Halo 博客平台将能够更好地应对当前和未来的需求，不仅提升性能和用户体验，还为开发者提供更强大的扩展能力。

## 非目标

本 RFC 的目标是引入和实现缩略图生成功能，以提升系统性能和用户体验，但以下内容不在本 RFC 的范围之内：

1. **不涉及图像内容的分析与处理**：

   - 本功能不包含任何图像内容分析（如图像识别、分类）或处理（如滤镜应用、图像增强）的功能。缩略图功能仅限于生成和管理不同尺寸的图像缩略图。

2. **不改变现有的存储策略实现**：

   - 本 RFC 并不涉及对现有存储策略（如本地存储、S3 协议扩展）的底层逻辑和实现方式进行修改。缩略图功能将以插件或扩展点的形式集成到现有系统中。

3. **不提供图片的格式转换**：

   - 本功能不会提供图片格式的转换功能，所有缩略图将使用与原图相同的格式进行生成和存储，避免涉及到图片格式的兼容性问题。

4. **不影响现有的附件管理功能**：
   - 引入缩略图功能不会影响现有的附件上传、下载、删除等基本功能。这些功能将保持不变，并与缩略图功能兼容。

## 方案

实现方案分为两大部分：**缩略图生成**和**缩略图的使用**。下面将分别介绍这两部分的设计和实现。

### 缩略图生成

在缩略图生成方面，有以下几个关键问题需要解决：

1. **缩略图尺寸的设计**：如何定义和选择合适的缩略图尺寸，以满足不同场景的需求。
2. **缩略图生成策略**：如何根据原始图片生成不同尺寸的缩略图，以保证图片质量和加载速度。
3. **缩略图存储策略**：如何将生成的缩略图存储在系统中，并与原始图片关联。
4. **缩略图生成的性能优化**：如何通过缓存和预生成等方式，提升缩略图生成的性能和效率。
5. **缩略图生成的扩展性**：如何设计和实现一个可扩展的缩略图生成机制，以支持不同的存储策略和需求。

考虑到使用方式和性能优化等因素，我们选择使用固定尺寸的缩略图，而非由具体宽度动态生成。这种设计有以下优势：

- **性能优化**：使用预定义的固定尺寸缩略图，可以减少缩略图滥用的计算和生成压力，在满足图片加载速度的同时，可以减小缩略图的数量和对资源的消耗。
- **一致性和可控性**：固定尺寸的缩略图可以保证图片展示的一致性和布局稳定性，过定义固定的缩略图尺寸，开发者可以更好地控制图片在不同场景中的展示效果。这样可以避免动态生成的图片尺寸不一致，导致页面布局问题或视觉不统一。
- **缓存利用**：固定尺寸的缩略图更容易被浏览器和 CDN 缓存，有助于加快图片加载速度，并减少对服务器的重复请求。

#### 缩略图尺寸设计

基于以上考虑，我们为系统预定义了以下几种常见的缩略图尺寸：

- **Small (S)**：宽度 400px
- **Medium (M)**：宽度 800px
- **Large (L)**：宽度 1200px
- **Extra Large (XL)**：宽度 1600px

定义枚举类型以便扩展点接口中使用：

```java
public enum ThumbnailSize {
    S(400),
    M(800),
    L(1200),
    XL(1600);

    private final int width;

    ThumbnailSize(int width) {
        this.width = width;
    }

    public int getWidth() {
        return width;
    }
}
```

#### 自定义模型设计

为了将图片与缩略图关联同时避免重复生成，需要定义一个自定义模型类，用于存储图片的 URL 和缩略图的 URL 之间的映射关系：

```yaml
apiVersion: storage.halo.run/v1alpha1
kind: Thumbnail
metadata:
  name: thumbnail-1
spec:
  imageSignature: e99a18c428cb38d5f260853678922e03
  imageUrl: /path/to/original.jpg
  size: L
  thumbnailUrl: /path/to/thumbnail-L.jpg
```

基于 `imageUrl` 生成的 MD5 签名，然后为 `imageSignature` 和 `size` 建立组合索引，一方面可以显著提高查询性能，同时也可以减少索引的大小。

```java
indexSpec.add(new IndexSpec()
   .setName(Thumbnail.ID_INDEX)
   .setIndexFunc(simpleAttribute(Thumbnail.class, Thumbnail::idIndexFunc))
);

public static String idIndexFunc(Thumbnail thumbnail) {
   return idIndexFunc(thumbnail.getSpec().getImageSignature(),
      thumbnail.getSpec().getSize().name());
}

public static String idIndexFunc(String imageHash, String size) {
   return imageHash + "-" + size;
}
```

#### 扩展点接口设计

为了保证缩略图功能的灵活性，我们设计了一个扩展点接口，使开发者能够根据不同的存储策略自定义缩略图生成逻辑。
例如，OSS 可能提供了根据参数生成缩略图的 API，可以避免在 Halo 生成：

```java
public interface ThumbnailProvider extends ExtensionPoint {

   /**
    * 根据给定的图片 URL 和尺寸生成缩略图 URL。
    * @param context 缩略图上下文，包含图片 URL 和尺寸等信息
    * @return 缩略图 URL
    */
   Mono<String> generate(ThumbnailContext context);

   /**
    * 根据给定的图片 URL 删除对应的缩略图文件
    * @param imageUrl 原图片的 URL
    */
   Mono<Void> delete(String imageUrl);

    /**
     * 判断当前提供者是否支持给定的图片 URL。
     *
     * @param imageUrl 图片的 URL
     * @return 如果支持，返回 true；否则返回 false
     */
   Mono<Boolean> supports(ThumbnailContext context);

   @Data
   @Builder
   public static class ThumbnailContext {
      private final String imageUrl;
      private final ThumbnailSize size;
   }
}
```

系统将提供一个默认的 `ThumbnailProvider` 实现，针对本地存储生成缩略图。对于其他存储策略（如 S3 协议的 OSS），开发者可以通过插件实现该接口，动态生成和返回缩略图 URL。
如果找不到合适的 `ThumbnailProvider` 实现，则不生成缩略图，避免因为用户本来就不需要缩略图而浪费资源。

缩略图会在首次使用时调用 `generate` 方法生成链接，然后将其记录到 `Thumbnail` 自定义模型中，而这个链接对应的缩略图是否已经生成取决于实现者，可能在 `generate` 方法调用时就已经生成，也可能是缩略图 URL 被访问时生成。

当附件被删除时，会调用 `delete` 方法删除对应的缩略图文件，以避免无用的缩略图占用存储空间。

#### 本地存储缩略图生成

对于本地存储，我们将提供一个默认的 `ThumbnailProvider` 实现，用于生成和存储缩略图。具体实现如下：

1. 根据参数给定的图片 URL 和尺寸，生成缩略图 URL，但不生成缩略图文件。
2. 将生成的缩略图 URL 和原始图片 URL 以及存储路径等信息保存到自定义模型中。
3. 通过 Reconciler 监听资源变化，为其生成缩略图文件。
4. 当缩略图 URL 被访问时，根据缩略图 URL 查询关联的缩略图资源并在查询到时检查缩略图文件是否存在，如果不存在则生成缩略图文件，否则返回 404。

##### 本地存储的自定义模型设计

自定义模型设计如下:

```yaml
apiVersion: storage.halo.run/v1alpha1
kind: LocalThumbnail
metadata:
  name: local-thumbnail-1
spec:
  imageSignature: e99a18c428cb38d5f260853678922e03
  imageUrl: /path/to/original.jpg
  thumbnailUri: /path/to/thumbnail-L.jpg
  thumbSignature: e99a18c428cb38d5f260853678922e03
  filePath: /attachments/thumbnails/2024/w1200/2024-08-09.jpg
  size: L
```

索引设计如下:

```java
schemeManager.register(LocalThumbnail.class, indexSpec -> {
   indexSpec.add(new IndexSpec()
         .setName("spec.imageSignature")
         .setIndexFunc(simpleAttribute(LocalThumbnail.class,
            thumbnail -> thumbnail.getSpec().getImageSignature())
         ));
   indexSpec.add(new IndexSpec()
         .setName("spec.thumbSignature")
         .setUnique(true)
         .setIndexFunc(simpleAttribute(LocalThumbnail.class,
            thumbnail -> thumbnail.getSpec().getThumbSignature())
         ));
});
```

定义 `LocalThumbnailEndpoint` 用于提供缩略图 URL 的访问，本地存储策略的图片缩略图规则为：

- Endpoint: `/upload/thumbnails/{year}/w{size}/{image-name}`
- 存储路径: `attachments/thumbnails/{year}/w{size}/{image-name}`

示例：

- Endpoint: `/upload/thumbnails/2024/w1200/2022-01-01.jpg`
- 存储路径: `attachments/thumbnails/2024/w1200/2022-01-01.jpg`

##### 缩略图生成库选择

在库的选择上，[Thumbnailator](https://github.com/coobird/thumbnailator) 和 [imgscalr](https://github.com/rkalla/imgscalr) 是最常用的两个缩略图生成库，简单易用且高效。

我们选择使用 imgscalr 作为 Halo 的缩略图生成库，主要基于以下考虑：

- **性能优化：** imgscalr 对性能进行了较好的优化，尤其适合处理大量图像或要求较高图像处理速度的场景。
- **高质量的图像处理：** imgscalr 提供了多种图像处理模式，可以在性能和图像质量之间找到平衡。

示例代码：

```java
private final URL imageUrl;
private final ThumbnailSize size;
private final Path storePath;

public Mono<Void> generate() {
   return Mono.fromRunnable(() -> {
      String formatName = getFormatName(imageUrl);
      try {
            var img = ImageIO.read(imageUrl);
            BufferedImage thumbnail = Scalr.resize(img, size.getWidth());
            var thumbnailFile = getThumbnailFile(formatName);
            ImageIO.write(thumbnail, formatName, thumbnailFile);
      } catch (IOException e) {
            throw Exceptions.propagate(e);
      }
   });
}

private static String getFormatName(URL input) {
   try {
      return doGetFormatName(input);
   } catch (IOException e) {
      // 获取不到图片格式时，返回 jpg
      return "jpg";
   }
}

private static String doGetFormatName(URL input) throws IOException {
   try (var inputStream = input.openStream();
         ImageInputStream imageStream = ImageIO.createImageInputStream(inputStream)) {
      Iterator<ImageReader> readers = ImageIO.getImageReaders(imageStream);
      if (!readers.hasNext()) {
            throw new IOException("No ImageReader found for the image.");
      }
      ImageReader reader = readers.next();
      return reader.getFormatName().toLowerCase();
   } catch (IOException e) {
      throw new IIOException("Can't get input stream from URL!", e);
   }
}
```

### 缩略图使用

#### 主题端

在主题端展示图片时，开发者可以利用 `srcset` 来实现响应式图片加载。通过在 `srcset` 属性中定义多个尺寸的图片 URL，浏览器会根据设备的屏幕分辨率和窗口大小，自动选择最合适的图片加载。

选择使用 srcset 的原因包括：

- 响应式设计：srcset 能够根据不同设备的分辨率和屏幕大小，自动选择最佳的图片尺寸。这种机制确保了在高分辨率设备上显示高质量图片的同时，也能够在低分辨率设备上节省带宽。

- 提升加载性能：通过为图片提供多个尺寸的缩略图，浏览器可以选择最适合当前视窗的图片进行加载，从而减少不必要的带宽使用，提升页面加载速度，改善用户体验。

- 简单集成：srcset 的使用非常直观，并且已经得到了主流浏览器的广泛支持。通过在图片标签中定义不同尺寸的图片 URL，开发者可以很容易地利用这个功能。

开发者可以通过定义缩略图的不同尺寸，在 `srcset` 中实现自动选择最适合的图片。例如：

```html
<img
  th:src="${imageUrl}"
  th:srcset="|
      ${#thumbnail.for(imageUrl, 's')} 400w,
      ${#thumbnail.for(imageUrl, 'm')} 800w,
      ${#thumbnail.for(imageUrl, 'l')} 1200w,
      ${#thumbnail.for(imageUrl, 'xl')} 1600w
   |"
  sizes="(max-width: 1600px) 100vw, 1600px"
  alt="Example Image"
/>
```

通过调用 `thumbnail.for(imageUrl, size)` 方法，开发者可以简便地生成缩略图 URL，并将其应用在 `srcset` 中，实现图片的响应式加载。

#### 文章内容中的图片

可以考虑同时使用以下两种方式支持:

1. 通过实现 `ReactivePostContentHandler` 扩展点，然后在 `handle` 方法中获取文章的 HTML 内容并解析 `img` 标签的 `src` 属性得到图片 URL，再根据图片 URL 生成缩略图 URL
   设置到 `srcset` 属性中，设置 `srcset` 属性前需要先判断是否已经存在 `srcset` 属性，如果存在则不再设置，避免覆盖开发者自定义的 `srcset` 属性。
2. 富文本编辑器中插入图片时，自动为图片设置 `srcset` 属性，以便在文章发布时自动加载缩略图。

#### 管理端

在管理端，通过获取 Attachment 的 status 中保存的 thumbnails 使用。该值是由 AttachmentReconciler 监听附件的变化，当 permalink 被生成后，生成缩略图 URL 并填充到 status 中。

```java
// when permalink is generated
if (AttachmentUtils.isImage(attachment)) {
   populateThumbnails(permalink, status);
}

void populateThumbnails(String permalink, AttachmentStatus status) {
   var imageUri = URI.create(permalink);
   Flux.fromArray(ThumbnailSize.values())
      .flatMap(size -> thumbnailService.generate(imageUri, size)
            .map(thumbUri -> Map.entry(size.name(), thumbUri.toString()))
      )
      .collectMap(Map.Entry::getKey, Map.Entry::getValue)
      .doOnNext(status::setThumbnails)
      .block();
}
```

### 总结

通过以上详细的实现方案设计，Halo 的缩略图功能将具备高度的灵活性和可扩展性，能够满足不同存储策略下的需求，并显著提升系统的性能和用户体验。
开发者在实现和使用该功能时，将能够享受更简洁的代码风格和更优雅的集成方式。
