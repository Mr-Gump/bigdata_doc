# 图标，表情

Material for MkDocs 最好的特性之一是可以在你的项目文档中使用超过 10000 个图标和数千个表情，而无需额外的努力。

## 搜索

<div class="mdx-iconsearch" data-mdx-component="iconsearch">
  <input
    class="md-input md-input--stretch mdx-iconsearch__input"
    placeholder="Search the icon and emoji database"
    data-mdx-component="iconsearch-query"
  />
  <div class="mdx-iconsearch-result" data-mdx-component="iconsearch-result">
    <div class="mdx-iconsearch-result__meta"></div>
    <ol class="mdx-iconsearch-result__list"></ol>
  </div>
</div>
<small>
  :octicons-light-bulb-16:
  **提示:** 输入一些关键词来找到表情和图标，点击表情码可以把它复制到剪贴板。
</small>

## 配置

请将下面几行添加到 `mkdocs.yml`:

``` yaml
markdown_extensions:
  - attr_list
  - pymdownx.emoji:
      emoji_index: !!python/name:materialx.emoji.twemoji
      emoji_generator: !!python/name:materialx.emoji.to_svg
```

Material for MkDocs 自带了下面的图标集:

- :material-material-design: – [Material Design]
- :fontawesome-brands-font-awesome: – [FontAwesome]
- :octicons-mark-github-16: – [Octicons]
- :simple-simpleicons: – [Simple Icons]

  [Material Design]: https://materialdesignicons.com/
  [FontAwesome]: https://fontawesome.com/search?m=free
  [Octicons]: https://octicons.github.com/
  [Simple Icons]: https://simpleicons.org/


## 使用

### 使用表情

可以使用两个 `:`  加上中间的表情码在 Markdown 中集成表情。
如果你正在使用 [Twemoji] (推荐), 你可以在 [Emojipedia] 查看表情码:

``` title="表情"
:smile: 
```

<div class="result" markdown>

:smile:

</div>

  [Twemoji]: https://twemoji.twitter.com/
  [Emojipedia]: https://emojipedia.org/twitter/

### 使用图标

当表情被启用时，图标也可以通过类似于表情的方式使用，通过引用一个与主题绑定的有效路径，这个路径位于 [`.icons`][custom icons] 文件夹，并且需要用 `-` 替代 `/`:

``` title="图标"
:fontawesome-regular-face-laugh-wink:
```

<div class="result" markdown>

:fontawesome-regular-face-laugh-wink:

</div>

  [custom icons]: https://github.com/squidfunk/mkdocs-material/tree/master/material/.icons
