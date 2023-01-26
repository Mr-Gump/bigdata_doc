# 角注

脚注是为特定的单词、短语或句子添加补充或额外信息的好方法，而不会打断文档的流程。Material for MkDocs 提供了定义、引用和渲染脚注的能力。

## 配置

请将下面几行添加到 `mkdocs.yml`:

``` yaml
markdown_extensions:
  - footnotes
```

## 使用

### 添加角注引用

脚注引用必须括在方括号中，并且必须以符号 `^` 开头，后面直接跟一个任意标识符，这类似于标准 Markdown 链接语法。

``` title="Text with footnote references"
Lorem ipsum[^1] dolor sit amet, consectetur adipiscing elit.[^2]
```

<div class="result" markdown>

Lorem ipsum[^1] dolor sit amet, consectetur adipiscing elit.[^2]

</div>

### 添加角注内容

脚注内容必须声明为与引用相同的标识符。它可以被插入到文档中的任意位置，并且总是呈现在页面的底部。此外，还会自动添加到脚注引用的反向链接。

#### 在一行上

短脚注可以写在同一行上:

``` title="Footnote"
[^1]: Lorem ipsum dolor sit amet, consectetur adipiscing elit.
```

<div class="result" markdown>

[:octicons-arrow-down-24: Jump to footnote](#fn:1)

</div>

  [^1]: Lorem ipsum dolor sit amet, consectetur adipiscing elit.

#### 在多行上

段落可以写在下一行，并且必须缩进四个空格:

``` title="Footnote"
[^2]:
    Lorem ipsum dolor sit amet, consectetur adipiscing elit. Nulla et euismod
    nulla. Curabitur feugiat, tortor non consequat finibus, justo purus auctor
    massa, nec semper lorem quam in massa.
```

<div class="result" markdown>

[:octicons-arrow-down-24: Jump to footnote](#fn:2)

</div>

[^2]:
    Lorem ipsum dolor sit amet, consectetur adipiscing elit. Nulla et euismod
    nulla. Curabitur feugiat, tortor non consequat finibus, justo purus
    auctor massa, nec semper lorem quam in massa.
