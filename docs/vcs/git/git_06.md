# 第六章 从远程仓库获取

上一节中我们把在 GitHub 上新建的仓库设置成了远程仓库，并向这个仓库 push 了 feature-D 分支。现在，所有能够访问这个远程仓库的人都可以获取 feature-D 分支并加以修改。

本节中我们从实际开发者的角度出发，在另一个目录下新建一个本地仓库，学习从远程仓库获取内容的相关操作。这就相当于我们刚刚执 行过 push 操作的目标仓库又有了另一名新开发者来共同开发。

## git clone —— 获取远程仓库

### 获取远程仓库

首先我们换到其他目录下，将 GitHub 上的仓库 clone 到本地。注意不要与之前操作的仓库在同一目录下。

![image-20230308101033138](https://cos.gump.cloud/uPic/image-20230308101033138.png)

执行 git clone 命令后我们会默认处于 master 分支下，同时系统会自动将 origin 设置成该远程仓库的标识符。也就是说，当前本地仓库 的 master 分支与 GitHub 端远程仓库（origin）的 master 分支在内容上是完全相同的。

![image-20230308101056798](https://cos.gump.cloud/uPic/image-20230308101056798.png)

我们用 git branch -a 命令查看当前分支的相关信息。添加参数可以同时显示本地仓库和远程仓库的分支信息。

结果中显示了 remotes/origin/feature-D，证明我们的远程仓库中已经有了 feature-D 分支。

### 获取远程的 feature-D 分支

我们试着将 feature-D 分支获取至本地仓库。

![image-20230308101131308](https://cos.gump.cloud/uPic/image-20230308101131308.png)

-b 参数的后面是本地仓库中新建分支的名称。 为了便于理解， 我们仍将其命名为 feature-D，让它与远程仓库的对应分支保持同名。新建分支名称后面是获取来源的分支名称。例子中指定了 origin/feature-D， 就是说以名为 origin 的仓库（这里指 GitHub 端的仓库）的 feature-D 分支为来源，在本地仓库中创建 feature-D 分支。

### 向本地的 feature-D 分支提交更改

现在假定我们是另一名开发者，要做一个新的提交。在 README. md 文件中添加一行文字，查看更改。

![image-20230308101225167](https://cos.gump.cloud/uPic/image-20230308101225167.png)

按照之前学过的方式提交即可。

![image-20230308101239983](https://cos.gump.cloud/uPic/image-20230308101239983.png)

### 推送 feature-D 分支

现在来推送 feature-D 分支。

![image-20230308101256567](https://cos.gump.cloud/uPic/image-20230308101256567.png)

从远程仓库获取 feature-D 分支， 在本地仓库中提交更改， 再将 feature-D 分支推送回远程仓库，通过这一系列操作，就可以与其他开发者相互合作，共同培育 feature-D 分支，实现某些功能。

## git pull —— 获取最新的远程仓库分支

现在我们放下刚刚操作的目录，回到原先的那个目录下。这边的本地仓库中只创建了 feature-D 分支，并没有在 feature-D 分支中进行任何提交。然而远程仓库的 feature-D 分支中已经有了我们刚刚推送的提交。 这时我们就可以使用 git pull 命令，将本地的 feature-D 分支更新到最新状态。当前分支为 feature-D 分支。

![image-20230308101354585](https://cos.gump.cloud/uPic/image-20230308101354585.png)

GitHub 端远程仓库中的 feature-D 分支是最新状态，所以本地仓库中的 feature-D 分支就得到了更新。今后只需要像平常一样在本地进行提交再 push 给远程仓库， 就可以与其他开发者同时在同一个分支中进行作业，不断给 feature-D 增加新功能。

如果两人同时修改了同一部分的源代码，push 时就很容易发生冲突。所以多名开发者在同一个分支中进行作业时，为减少冲突情况的发 生，建议更频繁地进行 push 和 pull 操作。

