# 按钮

Material for MkDocs 为能够把链接转化为一级和二级按钮提供了丰富的样式（`label` 或者 `button` 元素）. 这对文档或者复杂页面中的中 _调用动作_ 特别有用。

## 配置

请把下面几行添加到 `mkdocs.yml`:

``` yaml
markdown_extensions:
  - attr_list
```

## 使用

### 添加按钮

为了把链接渲染为按钮, 在它的后缀添加大括号并在其中添加 `.md-button` 类选择器。

``` markdown title="按钮"
[Subscribe to our newsletter](#){ .md-button }
```

<div class="result" markdown>

[Subscribe to our newsletter](#){ .md-button }

</div>

### 添加一级按钮

如果想展示一个一级被填充的按钮, 同时添加 `.md-button` 和 `.md-button--primary`
CSS 类选择器.

``` markdown title="一级按钮"
[Subscribe to our newsletter](#){ .md-button .md-button--primary }
```

<div class="result" markdown>

[Subscribe to our newsletter](#){ .md-button .md-button--primary }

</div>


### 添加图标按钮

当然,通过使用图标语法和图标码可以在任何类型的按钮上添加图标。

``` markdown title="带图标按钮"
[Send :fontawesome-solid-paper-plane:](#){ .md-button }
```

<div class="result" markdown>

[Send :fontawesome-solid-paper-plane:](#){ .md-button }

</div>
