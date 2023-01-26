# 内容标签页

有时，很需要在不同的标签页下组织不同的内容，例如当描述如何使用不同的语言或环境访问一个 API，Material for MkDocs 允许使用漂亮的功能性标签来组织代码块或其他内容。 

## 配置

请将下面几行添加到 `mkdocs.yml`:

``` yaml
markdown_extensions:
  - pymdownx.superfences
  - pymdownx.tabbed:
      alternate_style: true 
```

### 连接内容标签页

当启用时，当用户点击一个标签页时，整个文档中的所有内容标签页都会被链接和切换到同一个标签。添加下面几行到 `mkdocs.yml`:

``` yaml
theme:
  features:
    - content.tabs.link
```

内容标签页根据它们的标签被链接，而不是偏移量。这意味着所有的带有相同标签的标签页当用户点击一个内容标签页时会被激活，无论它们在块中的顺序。而且这个特性完全和
[instant loading] 集成并且在整个页面加载时生效。

=== "Feature enabled"

    [![Linked content tabs enabled]][Linked content tabs enabled]

=== "Feature disabled"

    [![Linked content tabs disabled]][Linked content tabs disabled]

  [Linked content tabs support]: https://github.com/squidfunk/mkdocs-material/releases/tag/8.3.0
  [Linked content tabs enabled]: ../assets/screenshots/content-tabs-link.png
  [Linked content tabs disabled]: ../assets/screenshots/content-tabs.png

## 使用

### 分组代码块

代码块是被分组的首要目标之一，可以被认为是内容标签页的一个特例，因为只有一个代码块的标签页总是被渲染为不带水平分割的样式:

``` title="含有代码块的内容标签页"
=== "C"

    ``` c
    #include <stdio.h>

    int main(void) {
      printf("Hello world!\n");
      return 0;
    }
    ```

=== "C++"

    ``` c++
    #include <iostream>

    int main(void) {
      std::cout << "Hello world!" << std::endl;
      return 0;
    }
    ```
```

<div class="result" markdown>

=== "C"

    ``` c
    #include <stdio.h>

    int main(void) {
      printf("Hello world!\n");
      return 0;
    }
    ```

=== "C++"

    ``` c++
    #include <iostream>

    int main(void) {
      std::cout << "Hello world!" << std::endl;
      return 0;
    }
    ```

</div>

### 分组其他内容

当一个内容标签页包含不止一个代码块，它们会被渲染成水平分布。垂直分布没有被添加，但是可以通过在其他块中嵌套标签页实现:

``` title="内容标签页"
=== "Unordered list"

    * Sed sagittis eleifend rutrum
    * Donec vitae suscipit est
    * Nulla tempor lobortis orci

=== "Ordered list"

    1. Sed sagittis eleifend rutrum
    2. Donec vitae suscipit est
    3. Nulla tempor lobortis orci
```

<div class="result" markdown>

=== "Unordered list"

    * Sed sagittis eleifend rutrum
    * Donec vitae suscipit est
    * Nulla tempor lobortis orci

=== "Ordered list"

    1. Sed sagittis eleifend rutrum
    2. Donec vitae suscipit est
    3. Nulla tempor lobortis orci

</div>

### 嵌入内容

当 SuperFences 被起用时，内容标签页可以包含嵌套内容，包括更深的内容标签页，并且也可以被其他块（例如提示或块引用）嵌套:

``` title="提示中的内容标签页"
!!! example

    === "Unordered List"

        ``` markdown
        * Sed sagittis eleifend rutrum
        * Donec vitae suscipit est
        * Nulla tempor lobortis orci
        ```

    === "Ordered List"

        ``` markdown
        1. Sed sagittis eleifend rutrum
        2. Donec vitae suscipit est
        3. Nulla tempor lobortis orci
        ```
```

<div class="result" markdown>

!!! example

    === "Unordered List"

        ``` markdown
        * Sed sagittis eleifend rutrum
        * Donec vitae suscipit est
        * Nulla tempor lobortis orci
        ```

    === "Ordered List"

        ``` markdown
        1. Sed sagittis eleifend rutrum
        2. Donec vitae suscipit est
        3. Nulla tempor lobortis orci
        ```

</div>

  [admonitions]: admonitions.md
