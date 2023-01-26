# MathJax

[MathJax] 是一个优雅的可用的在浏览器中展示数学类内容的方式，为包含不同符号数学排版添加支持(例如 [LaTeX], [MathML], [AsciiMath])，并且可以很容易地与
Material for MkDocs 集成.

  [MathJax]: https://www.mathjax.org/
  [LaTeX]: https://en.wikibooks.org/wiki/LaTeX/Mathematics
  [MathML]: https://en.wikipedia.org/wiki/MathML
  [AsciiMath]: http://asciimath.org/

## 配置

创建一个配置文件并且添加下面几行到 `mkdocs.yml`:

=== ":octicons-file-code-16: `docs/javascripts/mathjax.js`"

    ``` js
    window.MathJax = {
      tex: {
        inlineMath: [["\\(", "\\)"]],
        displayMath: [["\\[", "\\]"]],
        processEscapes: true,
        processEnvironments: true
      },
      options: {
        ignoreHtmlClass: ".*|",
        processHtmlClass: "arithmatex"
      }
    };

    document$.subscribe(() => { // (1)!
      MathJax.typesetPromise()
    })
    ```

=== ":octicons-file-code-16: `mkdocs.yml`"

    ``` yaml
    markdown_extensions:
      - pymdownx.arithmatex:
          generic: true

    extra_javascript:
      - javascripts/mathjax.js
      - https://polyfill.io/v3/polyfill.min.js?features=es6
      - https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js
    ```

<script src="https://polyfill.io/v3/polyfill.min.js?features=es6"></script>
<script id="MathJax-script" async src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>
<script>
  window.MathJax = {
    tex: {
      inlineMath: [["\\(", "\\)"]],
      displayMath: [["\\[", "\\]"]],
      processEscapes: true,
      processEnvironments: true
    },
    options: {
      ignoreHtmlClass: ".*|",
      processHtmlClass: "arithmatex"
    }
  };
</script>

## 使用

### 使用块语法

块必须被包含在 `#!latex $$...$$` 或 `#!latex \[...\]` 中:

``` latex title="MathJax, block syntax"
$$
\operatorname{ker} f=\{g\in G:f(g)=e_{H}\}{\mbox{.}}
$$
```

<div class="result" markdown>

$$
\operatorname{ker} f=\{g\in G:f(g)=e_{H}\}{\mbox{.}}
$$

</div>

### 使用行内块语法

行内块必须被包含在 `#!latex $...$` 或 `#!latex \(...\)` 中:

``` latex title="MathJax, inline syntax"
The homomorphism $f$ is injective if and only if its kernel is only the 
singleton set $e_G$, because otherwise $\exists a,b\in G$ with $a\neq b$ such 
that $f(a)=f(b)$.
```

<div class="result" markdown>

The homomorphism $f$ is injective if and only if its kernel is only the 
singleton set $e_G$, because otherwise $\exists a,b\in G$ with $a\neq b$ such 
that $f(a)=f(b)$.

</div>
