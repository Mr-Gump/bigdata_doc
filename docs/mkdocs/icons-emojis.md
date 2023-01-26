# 图标，表情

One of the best features of Material for MkDocs is the possibility to use [more
than 10,000 icons][icon search] and thousands of emojis in your project 
documentation with practically zero additional effort. Moreover, custom icons 
can be added and used in `mkdocs.yml`, documents and templates.

  [icon search]: #search

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
  **Tip:** Enter some keywords to find icons and emojis and click on the
  shortcode to copy it to your clipboard.
</small>

## 配置

This configuration enables the use of icons and emojis by using simple
shortcodes which can be discovered through the [icon search]. Add the following
lines to `mkdocs.yml`:

``` yaml
markdown_extensions:
  - attr_list
  - pymdownx.emoji:
      emoji_index: !!python/name:materialx.emoji.twemoji
      emoji_generator: !!python/name:materialx.emoji.to_svg
```

The following icon sets are bundled with Material for MkDocs:

- :material-material-design: – [Material Design]
- :fontawesome-brands-font-awesome: – [FontAwesome]
- :octicons-mark-github-16: – [Octicons]
- :simple-simpleicons: – [Simple Icons]

See additional configuration options:

- [Attribute Lists]
- [Emoji]
- [Emoji with custom icons]

  [Material Design]: https://materialdesignicons.com/
  [FontAwesome]: https://fontawesome.com/search?m=free
  [Octicons]: https://octicons.github.com/
  [Simple Icons]: https://simpleicons.org/
  [Attribute Lists]: ../setup/extensions/python-markdown.md#attribute-lists
  [Emoji]: ../setup/extensions/python-markdown-extensions.md#emoji
  [Emoji with custom icons]: ../setup/extensions/python-markdown-extensions.md#custom-icons

## 使用

### 使用表情

Emojis can be integrated in Markdown by putting the shortcode of the emoji
between two colons. If you're using [Twemoji] (recommended), you can look up
the shortcodes at [Emojipedia]:

``` title="Emoji"
:smile: 
```

<div class="result" markdown>

:smile:

</div>

  [Twemoji]: https://twemoji.twitter.com/
  [Emojipedia]: https://emojipedia.org/twitter/

### 使用图标

When [Emoji] is enabled, icons can be used similar to emojis, by referencing
a valid path to any icon bundled with the theme, which are located in the
[`.icons`][custom icons] directory, and replacing `/` with `-`:

``` title="Icon"
:fontawesome-regular-face-laugh-wink:
```

<div class="result" markdown>

:fontawesome-regular-face-laugh-wink:

</div>

  [custom icons]: https://github.com/squidfunk/mkdocs-material/tree/master/material/.icons
