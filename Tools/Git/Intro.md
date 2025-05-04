# win10配置
1. 配置\
    ``git config --global user.name "Ravoid-X"``\
    ``git config --global user.email "ravoid.xcc@gmail.com"``
    ``git config --list``
2. 设置ssh key\
    ``ssh-keygen -t rsa -C "ravoid.xcc@gmail.com"``
3. 将id_rsa.pub(~/.ssh/)拷贝到github设置中的SSH and GPG keys中new SSH Key
4. C:\Windows\System32\drivers\etc中host添加git域名
5. 测试\
   ``ssh git@github.com``\
   ``ssh -T git@github.com``

# 关系
1. 工作区:电脑里能看到的目录
2. 暂存区：一般存放在.git 目录下的index 文件（.git/index）中
3. 版本库:即工作区的隐藏目录 .git\
<img src="../../Pic/Tools/Git/git-command.jpg">
# 基本操作
1. 当前目录初始化为 Git 仓库\
    ``git init``
2. 克隆远程仓库到本地,包括版本变化\
    ``git clone <url> <directory>``
3. 添加文件到暂存区\
    ``git add <file>/[directory]``\
     ``git add .``添加当前目录下的所有文件到暂存区
4. 将暂存区内容添加到仓库中\
    ``git commit [file1] [file2] ... -m [message]``\
    ``git commit -a``不需要执行 git add 命令，直接提交\
    Linux中使用单引号'，Windows双引号"，[file]可省略
5. 将文件从暂存区和工作区中删除\
   ``git rm <file>``\
   ``git rm -f <file>``删除之前修改过并且已经放到暂存区域\
   ``git rm --cached <file>``从暂存区域移除，保留在当前工作目录中
6. 显示所有远程仓库\
   ``git remote -v``
7. 显示某个远程仓库的信息\
   ``git remote show [url]``
8. 提取远程仓库更新的数据\
   ``git fetch [alias]``
9. 将更新与本地当前分支合并\
   ``git merge [alias]/[branch]``
10. git pull 命令用于从远程获取代码并合并本地的版本,=git fetch+git merge\
    ``git pull [alias]/[remote branch]:[local branch]``\
    如果远程分支是与本地当前分支合并，则冒号后面的部分可以省略
11. 本地的分支版本上传到远程并合并\
    ``git push [alias]/[local branch]:[remote branch]``\
    本地分支名与远程分支名相同，则冒号后面的部分可以省略\
    如果本地版本与远程版本有差异，但又要强制推送可以使用 --force 参数\
    ``git push origin --delete master``\
    删除远程的分支可以使用 --delete 参数
12. 创建分支\
    ``git branch [branchname]``
13. 切换分支\
    ``git checkout [branchname]``
14. 删除分支\
    ``git branch -d [branchname]``
15. 合并到主分支\
    ``git merge [branchname]``
16. 合并时可能有冲突，用 git add 要告诉 Git 文件冲突已经解决
17. git log - 查看历史提交记录\
    ``--reverse``逆向显示\
    ``--author=xxx``查找指定用户\
    ``--no-merges``隐藏合并提交\
    ``--before={3.weeks.ago} --after={2010-04-18}``限定时间
18. git blame <file> - 以列表形式查看指定文件的历史修改记录
19. 给commit打标签,后面加上提交编号提交追加标签\
    ``git tag -a v1.0``