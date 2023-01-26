# 代码块

代码块和示例代码在技术项目文档中至关重要。Material for MkDocs 提供了不同的方式来为代码块开启语法高亮。

## 配置

请把下面几行添加到 `mkdocs.yml`:

``` yaml
markdown_extensions:
  - pymdownx.highlight:
      anchor_linenums: true
  - pymdownx.inlinehilite
  - pymdownx.snippets
  - pymdownx.superfences
```

### 代码复制按钮

代码块可以自动在右边渲染一个按钮来允许用户复制块中的代码到剪贴板。请添加下面几行到
`mkdocs.yml` 来全局开启:

``` yaml
theme:
  features:
    - content.code.copy
```

### 代码标记

代码标记提供了一种友好的方式为特定代码块段落附加解释内容，通过在代码块中添加一些标记和行内评论. 
请添加一下几行到
`mkdocs.yml` 来全局开启:

``` yaml
theme:
  features:
    - content.code.annotate # (1)!
```

1.  :man_raising_hand: I'm a code annotation! I can contain `code`, __formatted
    text__, images, ... basically anything that can be written in Markdown.


#### 锚点链接

为了更简单的链接和分享代码标记, 每个代码标记都被添加了锚点链接, 你可以通过右键复制或者在新标签页打开:

``` yaml
# (1)!
```

1.  If you ++cmd++ :material-plus::material-cursor-default-outline: me, I'm
    rendered open in a new tab. You can also right-click me to __copy link
    address__ to share me with others.


## 使用

代码块必须被两个三引号包围，
为这些代码块添加代码高亮, 在块开头处添加语言码。查看 [list of available lexers] 以找到对应的语言码:

```` markdown title="代码块"
``` py
import tensorflow as tf
```
````

<div class="result" markdown>

``` py
import tensorflow as tf
```

</div>

  [list of available lexers]: https://pygments.org/docs/lexers/

### 添加一个标题

为了提供额外的上下文，使用 `title="<custom title>"` 可以在语言码后面直接给代码块添加一个自定义标题，
例如展示文件名:

```` markdown title="带标题的代码块"
``` py title="bubble_sort.py"
def bubble_sort(items):
    for i in range(len(items)):
        for j in range(len(items) - 1 - i):
            if items[j] > items[j + 1]:
                items[j], items[j + 1] = items[j + 1], items[j]
```
````

<div class="result" markdown>

``` py title="bubble_sort.py"
def bubble_sort(items):
    for i in range(len(items)):
        for j in range(len(items) - 1 - i):
            if items[j] > items[j + 1]:
                items[j], items[j + 1] = items[j + 1], items[j]
```

</div>

### 添加标记

Code annotations can be placed anywhere in a code block where a comment for the
language of the block can be placed, e.g. for JavaScript in `#!js // ...` and
`#!js /* ... */`, for YAML in `#!yaml # ...`, etc.[^1]:

  [^1]:
    Code annotations require syntax highlighting with [Pygments] – they're
    currently not compatible with JavaScript syntax highlighters, or languages
    that do not have comments in their grammar. However, we're actively working
    on supporting alternate ways of defining code annotations, allowing to
    always place code annotations at the end of lines.

```` markdown title="Code block with annotation"
``` yaml
theme:
  features:
    - content.code.annotate # (1)
```

1.  :man_raising_hand: I'm a code annotation! I can contain `code`, __formatted
    text__, images, ... basically anything that can be written in Markdown.
````

<div class="result" markdown>

``` yaml
theme:
  features:
    - content.code.annotate # (1)
```

1.  :man_raising_hand: I'm a code annotation! I can contain `code`, __formatted
    text__, images, ... basically anything that can be written in Markdown.

</div>

#### 删除注释

如果你想删除围绕着代码标记的注释自负，只需要在它后面加一个 `!` :

```` markdown title="Code block with annotation, stripped"
``` yaml
# (1)!
```

1.  Look ma, less line noise!
````

<div class="result" markdown>

``` yaml
# (1)!
```

1.  Look ma, less line noise!

</div>

Note that this only allows for a single code annotation to be rendered per
comment. If you want to add multiple code annotations, comments cannot be
stripped for technical reasons.

  [Stripping comments support]: https://github.com/squidfunk/mkdocs-material/releases/tag/8.5.0

### 添加行号

可以在语言码后使用 `linenums="<start>"` 选项，`<start>` 代表起始行号。 一个代码块可以使用 `1` 以外的起始行号，这通常被用来切分大代码块以增加可读性:

```` markdown title="带行号代码块"
``` py linenums="1"
def bubble_sort(items):
    for i in range(len(items)):
        for j in range(len(items) - 1 - i):
            if items[j] > items[j + 1]:
                items[j], items[j + 1] = items[j + 1], items[j]
```
````

<div class="result" markdown>

``` py linenums="1"
def bubble_sort(items):
    for i in range(len(items)):
        for j in range(len(items) - 1 - i):
            if items[j] > items[j + 1]:
                items[j], items[j + 1] = items[j + 1], items[j]
```

</div>

### 高亮指定行

Specific lines can be highlighted by passing the line numbers to the `hl_lines`
argument placed right after the language shortcode. Note that line counts start
at `1`, regardless of the starting line number specified a art of
[`linenums`][Adding line numbers]:

```` markdown title="带高亮行的代码块"
``` py hl_lines="2 3"
def bubble_sort(items):
    for i in range(len(items)):
        for j in range(len(items) - 1 - i):
            if items[j] > items[j + 1]:
                items[j], items[j + 1] = items[j + 1], items[j]
```
````

<div class="result" markdown>

``` py linenums="1" hl_lines="2 3"
def bubble_sort(items):
    for i in range(len(items)):
        for j in range(len(items) - 1 - i):
            if items[j] > items[j + 1]:
                items[j], items[j + 1] = items[j + 1], items[j]
```

</div>

  [Adding line numbers]: #adding-line-numbers

### 高亮行内代码块

当 InlineHilite 被启用时，语法高亮可以被应用到行内代码块，通过在前面加上 `#!` 和语言码:

``` markdown title="Inline code block"
The `#!python range()` function is used to generate a sequence of numbers.
```

<div class="result" markdown>

The `#!python range()` function is used to generate a sequence of numbers.

</div>

### 嵌入外部文件

当 Snippets 被启用，其他文件的内容（包括源文件）
可以通过使用 [`--8<--` notation][Snippets notation] 嵌入到一个代码块中
from within a code block:

```` markdown title="带有外部文件的代码块"
``` title=".browserslistrc"
.--8<-- ".browserslistrc"
```
````

<div class="result" markdown>

``` title="hello.txt"
--8<-- "hello.txt"
```

</div>

  [Snippets notation]: https://facelessuser.github.io/pymdown-extensions/extensions/snippets/#snippets-notation

