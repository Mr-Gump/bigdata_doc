# 第五章 推送至远程仓库

Git 是分散型版本管理系统，但我们前面所学习的， 都是针对单一 本地仓库的操作。 下面，我们将开始接触远在网络另一头的远程仓 库。 远程仓库顾名思义， 是与我们本地仓库相对独立的另一个仓库。 让我们先在 GitHub 上创建一个仓库， 并将其设置为本地仓库的远程仓库。

## git remote add —— 添加远程仓库

在 GitHub 上创建的仓库路径为 “ git@github.com: 用户名 / git-tutorial.git ”。现在我们用 git remote add 命令将它设置成本地仓库的远程仓库。

![image-20230308100651648](https://cos.gump.cloud/uPic/image-20230308100651648.png)

按照上述格式执行 git remote add 命令之后，Git 会自动将 git@github.com:github-book/git-tutorial.git 远程仓库的名称设置为 origin（标识符）。

## git push —— 推送至远程仓库

### 推送至 master 分支

如果想将当前分支下本地仓库中的内容推送给远程仓库，需要用到 git push 命令。现在假定我们在 master 分支下进行操作。

![image-20230308100737767](https://cos.gump.cloud/uPic/image-20230308100737767.png)

像这样执行 git push 命令，当前分支的内容就会被推送给远程仓库 origin 的 master 分支。 -u 参数可以在推送的同时，将 origin 仓库的 master 分支设置为本地仓库当前分支的 upstream（上游）。添加了这个参数，将来运行 git pull 命令从远程仓库获取内容时，本地仓库的这个分支就可以直接从 origin 的 master 分支获取内容，省去了另外添加参数的麻烦。

执行该操作后， 当前本地仓库 master 分支的内容将会被推送到 GitHub 的远程仓库中。在 GitHub 上也可以确认远程 master 分支的内容和本地 master 分支相同。

### 推送至 master 以外的分支

除了 master 分支之外，远程仓库也可以创建其他分支。举个例子，我们在本地仓库中创建 feature-D 分支，并将它以同名形式 push 至远程仓库。

![image-20230308100851034](https://cos.gump.cloud/uPic/image-20230308100851034.png)

我们在本地仓库中创建了 feature-D 分支，现在将它 push 给远程仓库并保持分支名称不变。

![image-20230308100908936](https://cos.gump.cloud/uPic/image-20230308100908936.png)

现在，在远程仓库的 GitHub 页面就可以查看到 feature-D 分支了。

