## TortoiseSVN 安装及简单使用教程

### 1.安装 TortoiseSVN

打开安装程序，基本上一路next就可以。但是有两个地方需要注意：

- 若需要集成到 IDEA 中，command line client tools 要勾选上。
- 安装路径可以自行指定。

![](E:\学习资料\笔记\images\20190809144704.png)

安装完成后，在桌面点击右键，应该会出现 SVN 的菜单。

![](E:\学习资料\笔记\images\20190809155630.png)



### 2.使用 TortoiseSVN

#### （1）从服务器上 check out 文件

右键选择 SVN Checkout，填写相关信息：

![](E:\学习资料\笔记\images\20190809145738.png)

URL of repository：版本库的位置，即项目文件位置。

Checkout directory：检出目录，文件存放在本地的位置。

Revision：HEAD revision 指的是服务器上的最新版本，下面可点击Show log选择版本。

点击 OK 后，输入用户名和密码并确定，即可开始下载文件。

![](E:\学习资料\笔记\images\20190809150944.png)

check out 完成后，打开文件存放的本地目录，可以看到一个隐藏文件 .svn 和从服务器下载的文件。

![](E:\学习资料\笔记\images\20190809151217.png)

**注意：不同用户有着不同的权限，请确保输入的文件路径的正确性。若提示无权限，请联系管理员。**

#### （2）commit 文件到服务器

选择要上传的文件，右键 TortoiseSVN -> Add：

![](E:\学习资料\笔记\images\20190809152321.png)

点击后，文件呈现如下状态，这个 Add 的动作并未真正的将档案放到 repository 中。仅仅是告知 SVN 准备       要在 Repository 中放入这些档案。

![](E:\学习资料\笔记\images\20190809152650.png)

空白处右键选择 SVN commit，在这里可以清楚地了解到哪些文件要被 commit 到 repository（版本库）中。同样的，如果有档案不想在这个时候 commit 到 repository ，可以取消选取的档案。在 Message 文本框中可以写入对本次 commit 的说明（建议规范化）。

![](E:\学习资料\笔记\images\20190809153138.png)

点击 OK 后完成 commit 动作，然后可以到本地文件目录中，确定是否所有的档案 icon 都有如下的绿色勾勾在上面，这代表文件都正确无误的提交到 repository 中。

![](E:\学习资料\笔记\images\20190809153608.png)

#### （3）从服务器上 update 文件

由于版本控制系统是由许多人共同使用。所以，同样的档案可能还有人会去进行编辑。为了确保本地工作目录中的档案与 repository 中的档案是同步的，在编辑前都应该先进行更新的操作。

在本地工作目录中空白处右键选择 SVN Update，即可更新到最新版本。

若需要更新到指定版本，右键 TortoiseSVN -> Update to revision，在 show log 中选择相应的版本更新。

#### （4）删除文件

在本地删除已上传到服务器上的文件后，需要白处右键选择 SVN commit，同步删除服务器上的文件。