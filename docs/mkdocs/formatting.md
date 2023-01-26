# 格式

Material for MkDocs 提供了对几种 HTML 元素的支持，这些元素可用于突出显示文档的部分或应用特定的格式。

## 配置

请将下面几行添加到 `mkdocs.yml`:

``` yaml
markdown_extensions:
  - pymdownx.critic
  - pymdownx.caret
  - pymdownx.keys
  - pymdownx.mark
  - pymdownx.tilde
```

## 使用

### 高亮改变

启用 Critic 后，可以使用 CriticMarkup，这增加了突出显示建议更改以及向文档添加内联注释的能力：

``` title="Text with suggested changes"
Text can be {--deleted--} and replacement text {++added++}. This can also be
combined into {~~one~>a single~~} operation. {==Highlighting==} is also
possible {>>and comments can be added inline<<}.

{==

Formatting can also be applied to blocks by putting the opening and closing
tags on separate lines and adding new lines between the tags and the content.

==}
```

<div class="result" markdown>

Text can be <del class="critic">deleted</del> and replacement text
<ins class="critic">added</ins>. This can also be combined into
<del class="critic">one</del><ins class="critic">a single</ins> operation.
<mark class="critic">Highlighting</mark> is also possible
<span class="critic comment">and comments can be added inline</span>.

<div>
  <mark class="critic block">
    <p>
      Formatting can also be applied to blocks by putting the opening and
      closing tags on separate lines and adding new lines between the tags and
      the content.
    </p>
  </mark>
</div>

</div>

### 高亮文本

启用 Caret，Mark&Tilde 时，可以使用简单的语法突出显示文本，这比直接使用相应的 [`mark`][mark], [`ins`][ins] and [`del`][del] HTML 标记更方便:

``` title="Text with highlighting"
- ==This was marked==
- ^^This was inserted^^
- ~~This was deleted~~
```

<div class="result" markdown>

- ==This was marked==
- ^^This was inserted^^
- ~~This was deleted~~

</div>

  [mark]: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/mark
  [ins]: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/ins
  [del]: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/del

### 下标和上标

当启用 Caret&Tilde Caret，Mark&Tilde 时，文本可以用简单的语法进行下标和上标，这比直接使用相应的 [`sub`][sub] and [`sup`][sup] HTML 标记更方便:

``` markdown title="Text with sub- und superscripts"
- H~2~O
- A^T^A
```

<div class="result" markdown>

- H~2~O
- A^T^A

</div>

  [sub]: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/sub
  [sup]: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/sup

### 添加键盘键

启用 Key 后，可以使用简单语法渲染键盘键。请参阅 [Python Markdown Extensions] 文档以了解所有可用的快捷方式：

``` markdown title="Keyboard keys"
++ctrl+alt+del++
```

<div class="result" markdown>

++ctrl+alt+del++

</div>

  [Python Markdown Extensions]: https://facelessuser.github.io/pymdown-extensions/extensions/keys/#extendingmodifying-key-map-index
