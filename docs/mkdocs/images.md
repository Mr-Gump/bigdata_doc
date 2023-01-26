# 图像

由于图像是 Markdown 的一等公民也是核心语法的一部分，处理它们可能比较困难。
Material for MkDocs 让处理图像更加轻松，通过提供不同风格的图像样式和图像那个说明。

## 配置

请添加下面几行到 `mkdocs.yml` 以支持将图片转换为数字并为大图片开启懒加载:

``` yaml
markdown_extensions:
  - attr_list
  - md_in_html
```

### Lightbox

如果你想为你的文档添加图像缩放的功能, 
[glightbox] 插件是一个很好的选择,因为它和 Material for MkDocs 集成得很好。使用 `pip` 安装它:

```
pip install mkdocs-glightbox
```

然后，把下面几行添加到 `mkdocs.yml`:

``` yaml
plugins:
  - glightbox
```

## 使用

### 图像对齐

当 [Attribute Lists] 被启用，图像可以通过 `align` 属性添加对齐方向 ，例如 `align=left` 或
`align=right`:

=== "Left"

    ``` markdown title="Image, aligned to left"
    ![Image title](https://dummyimage.com/600x400/eee/aaa){ align=left }
    ```

    <div class="result" markdown>

    ![Image title](https://dummyimage.com/600x400/f5f5f5/aaaaaa&text=–%20Image%20–){ align=left width=300 }

    Lorem ipsum dolor sit amet, consectetur adipiscing elit. Nulla et euismod
    nulla. Curabitur feugiat, tortor non consequat finibus, justo purus auctor
    massa, nec semper lorem quam in massa.

    </div>

=== "Right"

    ``` markdown title="Image, aligned to right"
    ![Image title](https://dummyimage.com/600x400/eee/aaa){ align=right }
    ```

    <div class="result" markdown>

    ![Image title](https://dummyimage.com/600x400/f5f5f5/aaaaaa&text=–%20Image%20–){ align=right width=300 }

    Lorem ipsum dolor sit amet, consectetur adipiscing elit. Nulla et euismod
    nulla. Curabitur feugiat, tortor non consequat finibus, justo purus auctor
    massa, nec semper lorem quam in massa.

    </div>

如果没有足够的空间渲染在内容旁边的图像, 图像会膨胀到整个视图大小, 例如移动设备试图

??? 问题 "为什么没有居中对齐?"

    [`align`][align] 属性不允许居中对齐，这就是这个选项不被 Material for MkDocs 支持的原因。[^1] 相反，
    图像说明语法可以使用，因为说明是可选项。

  [^1]:
    你可能也意识到 [`align`][align] 数形已经被 HTML5 弃用，所以为什么总是使用它呢? 很可能的原因是
    – 它仍然被大多数的浏览器和客户端支持，并且几乎不可能被完全移除，因为许多更老的网站仍然在使用。这确保了在 Material for MkDocs 生成的网站之外查看带有这些属性的 Markdown 文件时外观一致。

  [align]: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#deprecated_attributes
  [image captions]: #image-captions

### 图像说明

令人失望的是，MarkDown 语法并不原生支持图像说明，但是使用 [Markdown in HTML] 插件的 `figure` 和 `figcaption` 标签使添加图像说明变为可能:

``` html title="带说明的图像"
<figure markdown>
  ![Image title](https://dummyimage.com/600x400/){ width="300" }
  <figcaption>Image caption</figcaption>
</figure>
```

<div class="result">
  <figure>
    <img src="https://dummyimage.com/600x400/f5f5f5/aaaaaa&text=–%20Image%20–" width="300" />
    <figcaption>Image caption</figcaption>
  </figure>
</div>

### 图像懒加载

现代浏览器提供对懒加载图像的原生支持 [lazy-loading]
通过 `loading=lazy` 标签，对于不支持懒加载的浏览器会降级为直接加载:

``` markdown title="Image, lazy-loaded"
![Image title](https://dummyimage.com/600x400/){ loading=lazy }
```

<div class="result" markdown>
  <img src="https://dummyimage.com/600x400/f5f5f5/aaaaaa&text=–%20Image%20–" width="300" />
</div>

  [lazy-loading]: https://caniuse.com/#feat=loading-lazy-attr
