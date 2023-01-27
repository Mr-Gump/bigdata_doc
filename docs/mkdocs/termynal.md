# Termynal

[Termynal] 是一个轻量级的在网页中嵌入炫酷的动画终端的前端工具。

[Termynal]:https://github.com/ines/termynal

## 配置

1. 下载 js 和 css 文件到本地

| 文件             | 链接                                |
| --------------- | ----------------------------------- |
| `termynal.css`  | :material-link:     [termynal.css]  |
| `custom.css`    | :material-link:     [custom.css]    |
| `termynal.js`   | :material-link:     [termynal.js]   |
| `custom.js`     | :material-link:     [custom.js]     |

[termynal.css]: https://raw.githubusercontent.com/tiangolo/typer/master/docs/css/termynal.css
[custom.css]: https://raw.githubusercontent.com/tiangolo/typer/master/docs/css/custom.css
[termynal.js]: https://raw.githubusercontent.com/tiangolo/typer/master/docs/js/termynal.js
[custom.js]: https://raw.githubusercontent.com/tiangolo/typer/master/docs/js/custom.js

2. 添加下面几行到 `mkdocs.yml`
```yaml
extra_css:
- css/termynal.css
- css/custom.css
extra_javascript:
- js/termynal.js
- js/custom.js
```

## 使用
### 语法
使用 Termynal 时只需要将代码块的语言码设置为 `console` ,并使用 包含 `class="termy"` 的 `<div>` 标签包围:

```` html title="Termynal"
<div class="termy">
```console
Hello termynal!
```
</div>
````

<div class="termy">
```console
Hello termynal!
```
</div>



### 命令
命令动画需要加上 `$` 前缀进行指定
```` html title="Termynal"
<div class="termy">
```console
$ pwd
/root
```
</div>

````
<div class="termy">
```console
$ pwd
/root
```
</div>


### 进度条

进度条动画需要使用 `---> 100%` 置于单独一行进行表示
```` html title="Termynal"
<div class="termy">
```console
$ pip install mkdocs
---> 100%
Requirement already satisfied!
```
</div>
````


<div class="termy">
```console
$ pip install mkdocs
---> 100%
Requirement already satisfied!
```
</div>


### 注释
注释动画需要加上 `//` 前缀进行指定

```` html title="Termynal"
<div class="termy">
```console
# test
$ ping baidu.com
```
</div>
````
<div class="termy">
```console
// ping test
$ ping baidu.com
PING baidu.com (198.18.3.115): 56 data bytes
64 bytes from 198.18.3.115: icmp_seq=0 ttl=63 time=0.138 ms
64 bytes from 198.18.3.115: icmp_seq=1 ttl=63 time=0.366 ms
64 bytes from 198.18.3.115: icmp_seq=2 ttl=63 time=0.468 ms
64 bytes from 198.18.3.115: icmp_seq=3 ttl=63 time=0.490 ms
--- baidu.com ping statistics ---
4 packets transmitted, 4 packets received, 0.0% packet loss
```
</div>
