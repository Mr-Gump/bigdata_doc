# 工具提示

技术项目文档通常包含许多缩写，这些缩写需要通常需要额外的解释，特别是对于项目的新用户。因为这些原因，Material for MkDocs 结合了 Markdown 扩展以在整个页面启用术语。

## 配置

请添加下面几行到 `mkdocs.yml`:

``` yaml
markdown_extensions:
  - abbr
  - attr_list
  - pymdownx.snippets
```

## 使用

### 添加工具提示

Markdown 语法允许为每一个链接指定一个 `title` 。使用下面几行给链接添加一个工具提示:

``` markdown title="带工具提示的链接（行内语法）"
[Hover me](https://example.com "I'm a tooltip!")
```

<div class="result" markdown>

[Hover me](https://example.com "I'm a tooltip!")

</div>

工具提示也可以通过引用添加:

``` markdown title="带工具提示的链接（引用语法）"
[Hover me][example]

  [example]: https://example.com "I'm a tooltip!"
```

<div class="result" markdown>

[Hover me](https://example.com "I'm a tooltip!")

</div>

对于其他所有元素, 一个 `title` 可以通过 [Attribute Lists] 插件添加:

``` markdown title="带工具提示的图标"
:material-information-outline:{ title="Important information" }
```

<div class="result" markdown>

:material-information-outline:{ title="Important information" }

</div>

### 添加缩写

缩写可以通过一种类似于 URL 和角注的语法定义，以 `*` 开始并且后面紧跟着紧接方括号内的术语或缩略语:

``` markdown title="带缩写的文本"
The HTML specification is maintained by the W3C.

*[HTML]: Hyper Text Markup Language
*[W3C]: World Wide Web Consortium
```

<div class="result" markdown>

The HTML specification is maintained by the W3C.

*[HTML]: Hyper Text Markup Language
*[W3C]: World Wide Web Consortium

</div>

### 添加术语表

Snippets 插件可以通过把所有的缩写移动到一个特定的文件 [^1]，并且使用 [auto-append] 将这个文件添加到所有页面:

  [^1]:
    强烈建议将包含缩写的 Markdown 文件放在 docs 文件夹之外(这里使用了一个名为 includes 的文件夹)，否则 MkDocs 可能会出现未引用的文件的警告。

=== ":octicons-file-code-16: `includes/abbreviations.md`"

    ```` markdown
    *[HTML]: Hyper Text Markup Language
    *[W3C]: World Wide Web Consortium
    ````

=== ":octicons-file-code-16: `mkdocs.yml`"

    ```` yaml
    markdown_extensions:
      - pymdownx.snippets:
          auto_append:
            - includes/abbreviations.md
    ````

  [auto-append]: https://facelessuser.github.io/pymdown-extensions/extensions/snippets/#auto-append-snippets
