# 工具提示

技术项目文档通常包含许多缩写，这些缩写需要通常需要额外的解释，特别是对于项目的新用户。因为这些原因，Material for MkDocs 结合了 Markdown 扩展以在整个页面启用术语。

## 配置

This configuration enables abbreviations and allows to build a simple
project-wide glossary, sourcing definitions from a central location. Add the
following line to `mkdocs.yml`:

``` yaml
markdown_extensions:
  - abbr
  - attr_list
  - pymdownx.snippets
```

## 使用

### 添加工具提示

The [Markdown syntax] allows to specify a `title` for each link, which will
render as a beautiful tooltip when [improved tooltips] are enabled. Add a 
tooltip to a link with the following lines:

``` markdown title="Link with tooltip, inline syntax"
[Hover me](https://example.com "I'm a tooltip!")
```

<div class="result" markdown>

[Hover me](https://example.com "I'm a tooltip!")

</div>

Tooltips can also be added to link references:

``` markdown title="Link with tooltip, reference syntax"
[Hover me][example]

  [example]: https://example.com "I'm a tooltip!"
```

<div class="result" markdown>

[Hover me](https://example.com "I'm a tooltip!")

</div>

For all other elements, a `title` can be added by using the [Attribute Lists]
extension:

``` markdown title="Icon with tooltip"
:material-information-outline:{ title="Important information" }
```

<div class="result" markdown>

:material-information-outline:{ title="Important information" }

</div>

  [Markdown syntax]: https://daringfireball.net/projects/markdown/syntax#link
  [improved tooltips]: #improved-tooltips

### 添加缩写

Abbreviations can be defined by using a special syntax similar to URLs and 
[footnotes], starting with a `*` and immediately followed by the term or
acronym to be associated in square brackets:

``` markdown title="Text with abbreviations"
The HTML specification is maintained by the W3C.

*[HTML]: Hyper Text Markup Language
*[W3C]: World Wide Web Consortium
```

<div class="result" markdown>

The HTML specification is maintained by the W3C.

*[HTML]: Hyper Text Markup Language
*[W3C]: World Wide Web Consortium

</div>

  [footnotes]: footnotes.md

### 添加术语表

The [Snippets] extension can be used to implement a simple glossary by moving
all abbreviations in a dedicated file[^1], and [auto-append] this file to all
pages with the following configuration:

  [^1]:
    It's highly recommended to put the Markdown file containing the
    abbreviations outside of the `docs` folder (here, a folder with the name 
    `includes` is used), as MkDocs might otherwise complain about an
    unreferenced file.

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
