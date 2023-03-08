# 第四章 更改提交的操作

## git reset —— 回溯历史版本

通过前面学习的操作， 我们已经学会如何在实现功能后进行提交， 累积提交日志作为历史记录，借此不断培育一款软件。

Git 的另一特征便是可以灵活操作历史版本。借助分散仓库的优势， 可以在不影响其他仓库的前提下对历史版本进行操作。

在这里， 为了让各位熟悉对历史版本的操作， 我们先回溯历史版本，创建一个名为 fix-B 的特性分支。

![image-20230308094721560](https://cos.gump.cloud/uPic/image-20230308094721560.png)

### 回溯到创建 feature-A 分支前

让我们先回溯到上一节 feature-A 分支创建之前， 创建一个名为 fix-B 的特性分支。

要让仓库的 HEAD、暂存区、当前工作树回溯到指定状态，需要用到 git rest --hard 命令。只要提供目标时间点的哈希值  ，就可以完全恢复至该时间点的状态。事不宜迟，让我们执行下面的命令。

![image-20230308094811982](https://cos.gump.cloud/uPic/image-20230308094811982.png)

我们已经成功回溯到特性分支（feature-A）创建之前的状态。由于所有文件都回溯到了指定哈希值对应的时间点上，README.md 文件的内容也恢复到了当时的状态。

### 创建 fix-B 分支

现在我们来创建特性分支（fix-B）。

![image-20230308094843208](https://cos.gump.cloud/uPic/image-20230308094843208.png)

作为这个主题的作业内容，我们在 README.md 文件中添加一行文字。

```markdown
# Git教程

- fix-B
```

然后直接提交 README.md 文件。

![image-20230308094915341](https://cos.gump.cloud/uPic/image-20230308094915341.png)

现在的状态如图1所示。 接下来我们的目标是图2中所示的状态，即主干分支合并 feature-A 分支的修改后，又合并了 fix-B 的修改。

=== "图1"

    <figure markdown>
      ![image-20230308095005211](https://cos.gump.cloud/uPic/image-20230308095005211.png)
      <figcaption>当前fix-B状态</figcaption>
    </figure>

=== "图2"

    <figure markdown>
      ![image-20230308095054503](https://cos.gump.cloud/uPic/image-20230308095054503.png)
      <figcaption>fix-目标</figcaption>
    </figure>

### 推进至 feature-A 分支合并后的状态

首先恢复到 feature-A 分支合并后的状态。不妨称这一操作为“推进历史”。

git log 命令只能查看以当前状态为终点的历史日志。所以这里 要使用 git reflog 命令，查看当前仓库的操作日志。在日志中找出回溯历史之前的哈希值，通过 git reset --hard 命令恢复到回溯历史前的状态。

首先执行 git reflog 命令，查看当前仓库执行过的操作的日志。

![image-20230308095225774](https://cos.gump.cloud/uPic/image-20230308095225774.png)

在日志中，我们可以看到 commit、checkout、reset、merge 等 Git 命令的执行记录。只要不进行 Git 的 GC（Garbage Collection，垃圾回收）， 就可以通过日志随意调取近期的历史状态，就像给时间机器指定一个时间点， 在过去未来中自由穿梭一般。 即便开发者错误执行了 Git 操作， 基本也都可以利用 git reflog 命令恢复到原先的状态，所以请各位读者务必牢记本部分。

从上面数第四行表示 feature-A 特性分支合并后的状态，对应哈希值为 83b0b94 。我们将 HEAD、暂存区、工作树恢复到这个时间点的状态。

![image-20230308095327391](https://cos.gump.cloud/uPic/image-20230308095327391.png)

之前我们使用 git reset --hard 命令回溯了历史，这里又再次通过它恢复到了回溯前的历史状态。当前的状态如图所示。

![image-20230308095411559](https://cos.gump.cloud/uPic/image-20230308095411559.png)

### 消除冲突

现在只要合并 fix-B 分支，就可以得到我们想要的状态。让我们赶快进行合并操作。

![image-20230308095432301](https://cos.gump.cloud/uPic/image-20230308095432301.png)

这时，系统告诉我们 README.md 文件发生了冲突（Conflict）。系统在合并 README.md 文件时，feature-A 分支更改的部分与本次想要合并的 fix-B 分支更改的部分发生了冲突。

不解决冲突就无法完成合并，所以我们打开 README.md 文件，解决这个冲突。

### 查看冲突部分并将其解决

用编辑器打开 README.md 文件，就会发现其内容变成了下面这个样子。

![image-20230308095532359](https://cos.gump.cloud/uPic/image-20230308095532359.png)

以上的部分是当前 HEAD 的内容，以下的部分是要合并的 fix-B 分支中的内容。我们在编辑器中将其改成想要的样子。

```markdown
# Git教程

- feature-A
- fix-B
```

如上所示，本次修正让 feature-A 与 fix-B 的内容并存于文件之中。 但是在实际的软件开发中，往往需要删除其中之一，所以各位在处理冲突时，务必要仔细分析冲突部分的内容后再行修改。

### 提交解决后的结果

冲突解决后，执行 git add 命令与 git commit 命令。

![image-20230308095634498](https://cos.gump.cloud/uPic/image-20230308095634498.png)

由于本次更改解决了冲突，所以提交信息记为 " Fix conflict " 。

## git commit --amend —— 修改提交信息

要修改上一条提交信息，可以使用 git commit --amend 命令。 

我们将上一条提交信息记为了 " Fix conflict " ，但它其实是 fix-B 分支的合并， 解决合并时发生的冲突只是过程之一， 这样标记实在不妥。 于是，我们要修改这条提交信息。

![image-20230308095716984](https://cos.gump.cloud/uPic/image-20230308095716984.png)

执行上面的命令后，编辑器就会启动。

![image-20230308095737661](https://cos.gump.cloud/uPic/image-20230308095737661.png)

编辑器中显示的内容如上所示， 其中包含之前的提交信息。 请将提交信息的部分修改为 Merge branch 'fix-B'，然后保存文件，关闭编 辑器。

![image-20230308095801225](https://cos.gump.cloud/uPic/image-20230308095801225.png)

随后会显示上面这条结果。现在执行 git 可以看到提交日志中的相应内容也已经被修改。log --graph 命令，

![image-20230308095852999](https://cos.gump.cloud/uPic/image-20230308095852999.png)

## git rebase -i —— 压缩历史

在合并特性分支之前，如果发现已提交的内容中有些许拼写错误等， 不妨提交一个修改，然后将这个修改包含到前一个提交之中，压缩成一个历史记录。这是个会经常用到的技巧，让我们来实际操作体会一下。

### 创建 feature-C 分支

首先，新建一个 feature-C 特性分支。

![image-20230308095937728](https://cos.gump.cloud/uPic/image-20230308095937728.png)

作为 feature-C 的功能实现，我们在 README.md 文件中添加一行文字，并且故意留下拼写错误，以便之后修正。

```markdown
# Git教程

- feature-A
- fix-B
- faeture-C
```

提交这部分内容。这个小小的变更就没必要先执行 git add 命令 再执行 git commit 命令了，我们用 git commit -am 命令来一次完成这两步操作。

![image-20230308100018286](https://cos.gump.cloud/uPic/image-20230308100018286.png)

### 修正拼写错误

现在来修正刚才预留的拼写错误。请各位自行修正 README.md 文件的内容，修正后的差别如下所示。

![image-20230308100108926](https://cos.gump.cloud/uPic/image-20230308100108926.png)

然后进行提交。

![image-20230308100120152](https://cos.gump.cloud/uPic/image-20230308100120152.png)

错字漏字等失误称作 typo，所以我们将提交信息记为 " Fix typo " 。

实际上，我们不希望在历史记录中看到这类提交，因为健全的历史记录并不需要它们。如果能在最初提交之前就发现并修正这些错误，也就不会出现这类提交了。

### 更改历史

因此， 我们来更改历史。 将 " Fix typo"修正的内容与之前一次的提交合并， 在历史记录中合并为一次完美的提交。 为此， 我们要用到 git rebase 命令。

![image-20230308100200538](https://cos.gump.cloud/uPic/image-20230308100200538.png)

用上述方式执行 git rebase 命令， 可以选定当前分支中包含 HEAD（最新提交）在内的两个最新历史记录为对象， 并在编辑器中打开。

![image-20230308100223897](https://cos.gump.cloud/uPic/image-20230308100223897.png)

我们将 6fba227 的 Fix typo 的历史记录压缩到 7a34294 的 Add feature-C 里。按照下图所示，将 6fba227 左侧的 pick 部分删除，改写为 fixup。

![image-20230308100301597](https://cos.gump.cloud/uPic/image-20230308100301597.png)

保存编辑器里的内容，关闭编辑器。

![image-20230308100313343](https://cos.gump.cloud/uPic/image-20230308100313343.png)

系统显示 rebase 成功。也就是以下面这两个提交作为对象，将 " Fix typo " 的内容合并到了上一个提交 " Add feature-C " 中，改写成了一个新的提交。

- 7a34294 Add feature-C
-  6fba227 Fix typo

现在再查看提交日志时会发现 Add feature-C 的哈希值已经不是 7a34294 了，这证明提交已经被更改。

![image-20230308100357577](https://cos.gump.cloud/uPic/image-20230308100357577.png)

这样一来，Fix typo 就从历史中被抹去，也就相当于 Add feature-C 中从来没有出现过拼写错误。这算是一种良性的历史改写。

### 合并至 master 分支

feature-C 分支的使命告一段落，我们将它与 master 分支合并。

![image-20230308100422839](https://cos.gump.cloud/uPic/image-20230308100422839.png)

master 分支整合了 feature-C 分支。开发进展顺利。

