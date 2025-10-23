## 常用操作
1. 管理者\
`sudo`
2. host添加raw.git\
`185.199.108.133 raw.githubsercontent.com`
3. 添加源\
`sudo gedit /etc/apt/sources.list`
4. 进入 home 目录：xcc 为用户名\
`cd /home/xcc/xxx`

## 更新软件
1. 更新软件源\
`sudo apt update`
2. 升级已安装软件\
`sudo apt upgrade`

## 权限修改
1. u: 所有者 g: 所有组 o: 其他人 a: 所有人(u、g、o的综合)
2. r = 4, w = 2, x = 1 rwx = 4+2+1 = 7
3. 方式一：+、-、=变更权限\
`chmod u=rwx, g=rx, o=x`\
`chmod o+w 文件目录名`
4. 方式二：通过数字\
```
chmod u = rwx, g = rx, o = x 文件目录名 //等价于
chmod 751 文件目录名
```
5. 修改文件所有者\
`chown newowner file`

## 文件操作  
1. 新建文件\
`touch/gedit/vim 文件名.后缀`
2. 新建文件夹\
`mkdir`
3. 打开root权限文件夹\
`sudo nautilus`
4. 递归复制文件夹\
`cp -r /targetFolder`
5. 删除：-r 递归删除文件夹；-f 强制删除不显示\
`rm`
6. 移动\
`mv /temp/movefiel /targetFolder`
7. 查看：|more 以分页的形式打开\
`cat`

## 搜索
1. 递归搜索：-name；-user；-size\
`find [目录] [选项]`
2. 快速定位：基于数据库，第一次运行前必须使用 updatedb 指令创建 locate 数据库\
`locate`
3. 过滤查找：-n	显示匹配行以及行号；-i	忽略字母大小写\
`grep [选项] 查找文件 源文件`

## 目录操作
1. 显示绝对路径\
`pwd`
2. 显示文件：-a 显示当前目录所有文件（包括隐藏）；-l 以列表显示\
`ls`

## 安装
1. deb格式安装\
`sudo apt install ./文件名`
2. pl格式安装\
`sudo ./文件名`

## 下载
1. 网址形式\
`wget http://XXX`
2. wget -O\
wget 默认以最后一个符合'/'的后面的字符来命名，对于动态链接的下载通常文件名会不正确。\
`wget -O name http://XXX`
3. 多文件，txt中每一行一个链接\
`wget -i filelist.txt`
4. -c：断点续传；-b：后台下载\
`wget http://XXX`

## 进程
1. 查看：-a 显示当前终端所有进程信息；-u 以用户的格式显示；-x 显示后台进程运行的参数
`ps`
2. 终止：-9 表示强迫进程立即停止
`kill [选项] 进程号/killall 进程名称`
3. 查看：-p 显示进程的PID；-u 显示进程的所属用户
`pstree [选项]`

## 界面操作    
1. 修改分辨率\
`xrandr --output Virtual1 --mode "1920x1080"`