## 概述
1. Shell 是一个命令解释器，架设在用户和内核之间
2. 既是用户与系统交互的交互式界面，也是一个强大的脚本编程环境
## 类型
### bash (Bourne-Again SHell)
1. 目前最主流、最通用的 Shell，是绝大多数 Linux 发行版（如 Ubuntu, CentOS, Red Hat）和 macOS（早期） 的默认 Shell
2. 是 sh的增强版，功能丰富，易于使用
### zsh (Z Shell)
1. 日益流行的 Shell，被认为是 bash 的终极魔改版
2. 最大的特点是极其强大的自动补全、拼写纠错和高度可定制性
3. 现在是 macOS 的默认 Shell
### sh (Bourne Shell)
1. 经典、古老的 Unix Shell，它的语法是所有现代 Shell 的基础
2. 在写一些需要高度兼容的脚本时，会使用 sh 语法。
3. 在 bash 环境中，sh 通常只是一个指向 bash 的符号链接（或 bash 运行在 sh 兼容模式下）
### 查看
```Bash
echo $SHELL 或 echo $0
```
## 命令基本结构
```Bash
command -options arguments
```
### 示例
```Bash
ls -l /home
ls: 命令，要执行的程序。
-l: 选项，用来调整命令的行为（这里是“长列表”格式）。
/home: 参数，命令操作的对象（这里是要查看的目录）。
```
## I/O 重定向
在 Linux 中，一切皆文件，每个进程默认打开三个文件描述符。重定向就是改变这些默认的流向
### 文件描述符
1. 0 (stdin): 标准输入（默认来自键盘）
2. 1 (stdout): 标准输出（默认输出到屏幕）
3. 2 (stderr): 标准错误（默认输出到屏幕）
### 重定向符号
1. `>` (覆盖输出): ls -l > file.txt。将 ls -l 的标准输出重定向到 file.txt，如果文件已存在，它将被覆盖。
2. `>>` (追加输出): echo "Hello" >> logs.txt。将 "Hello" 的标准输出追加到 logs.txt 的末尾，不覆盖原有内容。
3. `<` (输入重定向): grep "error" < logs.txt。grep 命令通常从 stdin 读取数据。这里将 logs.txt 文件的内容作为 grep 的标准输入
4. `2>` (错误重定向): find / -name "secret" 2> errors.txt\
（1）find 命令在没有权限访问某些目录时会产生标准错误\
（2）此命令将所有错误信息重定向到 errors.txt，而正常找到的结果仍然会显示在屏幕上
5. `&>` (合并重定向): make &> build.log。常用技巧，等同于 make > build.log 2>&1。将标准输出和标准错误合并，并一起重定向到 build.log
## 管道 (Pipes |)
管道 | 将前一个命令的 stdout 连接到后一个命令的 stdin
### 示例
```Bash
ps aux | grep "nginx"
```
1. ps aux: 列出系统上所有正在运行的进程（产生大量输出到 stdout）。
2. | (管道): 将 ps aux 的全部输出，作为 grep 命令的输入。
3. grep "nginx": 从接收到的输入中，只过滤并显示包含 "nginx" 的行。
4. 剖析: 这两个命令是同时运行的。ps 一边产生输出，grep 就一边读取并过滤，非常高效。
## 变量
### 环境变量
1. 特性: 大写，全局范围，可被子进程继承。
2. 示例:$PATH: Shell 搜索命令的路径列表；$HOME: 当前用户的家目录；$USER: 当前用户名
3. 示例: export MY_API_KEY="xyz123" (使用 export 使其成为环境变量)
### 本地变量
1. 特性: 通常小写，只在当前 Shell 实例中有效
2. 示例 1: name="Alice" (注意：= 两边不能有空格)
3. 示例 2: echo "Hello, $name" 或 echo "Hello, ${name}"
4. ${} 可以清晰地界定变量名，推荐使用
## 引号
决定了 Shell 是否以及如何进行展开
### `"`(双引号 - "弱"引用)
会进行变量展开和命令替换，但会阻止通配符展开和单词分割
```Bash
name="Alice"
echo "Hello, $name. Today is $(date +%F)"
# 输出: Hello, Alice. Today is 2025-10-27
```
### `'`(单引号 - "强"引用)
内部的一切都视为字面量 (Literal)，不进行任何展开
```Bash
name="Alice"
echo 'Hello, $name. Today is $(date +%F)'
# 输出: Hello, $name. Today is $(date +%F)
```
### `$(...)`(命令替换)
1. Shell 会先执行 (...) 内部的命令，然后将其标准输出替换到当前位置
2. 示例: filename="backup-$(date +%Y%m%d).tar.gz"
## 通配符(Wildcards / Globbing)
1. 由 Shell 负责展开，在命令执行之前就完成了
2. 与正则表达式不同，正则由 grep, sed 等命令自己解析
### `*`
1. 匹配 0 个或多个任意字符
2. ls *.txt: 匹配所有以 .txt 结尾的文件
### `?`
1. 匹配 1 个任意字符
2. ls file?.log: 匹配 file1.log, fileA.log，但不匹配 file10.log
### `[]`
1. 匹配方括号内的任意一个字符
2. ls [abc].jpg: 匹配 a.jpg, b.jpg, c.jpg
3. ls [0-9].txt: 匹配 1.txt, 2.txt ... 9.txt
## 进程控制
### & (后台运行)
sleep 60 &: sleep 60 命令会在后台运行，Shell 会立即返回提示符，继续输入其他命令
### Ctrl+C
发送 SIGINT (中断) 信号，强制终止前台正在运行的命令。
### Ctrl+Z
发送 SIGTSTP (暂停) 信号，将前台命令暂停并放入后台。
### jobs
查看所有在后台暂停或运行的任务。
### fg
将后台任务（如 fg %1）调回前台继续运行。
### bg
让一个已暂停的后台任务（如 bg %1）在后台继续运行
## 脚本编程
### #! (Shebang)
1. 第一行通常是 #!/bin/bash (或 #!/bin/sh)
2. 不是注释，其告诉操作系统，当执行这个文件时，应该使用 /bin/bash 解释器来运行
### 条件判断 (if)
1. `[` 是 test 命令的简写，用于进行测试
2. if [ -f "file.txt" ]; then ... fi\
（1）-f: 测试文件是否存在且为普通文件\
（2）-d: 测试是否为目录\
（3）-z "$VAR": 测试变量是否为空\
（4）"$A" == "$B": 测试字符串是否相等 (在 `[` 中建议用 == 或 =，在 `[[` 中用 ==)。
### 循环 (for / while)
1. for
```Bash
for i in 1 2 3 4 5; do
  echo "Number: $i"
done
```
2. while (常用于逐行读取文件)
```Bash
while read -r line; do
  echo "Read line: $line"
done < file.txt
```
### 函数
```Bash
greet() {
  local name="$1"  # $1 是第一个参数, local使其成为局部变量
  echo "Hello, $name"
}
# 调用函数
greet "Alice"
```
## 示例
一个简单的备份脚本
```Bash
#!/bin/bash
# 备份脚本：将指定目录压缩并加上时间戳
# --- 变量定义 ---
SOURCE_DIR="/var/log"            # 要备份的目录
BACKUP_DIR="/mnt/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
ARCHIVE_FILE="log-backup-${TIMESTAMP}.tar.gz"
# --- 逻辑执行 ---
echo "Starting backup of ${SOURCE_DIR}..."
# 1. 检查备份目录是否存在
if [ ! -d "$BACKUP_DIR" ]; then
  echo "Backup directory $BACKUP_DIR does not exist. Creating it."
  mkdir -p "$BACKUP_DIR"
  if [ $? -ne 0 ]; then
    echo "Failed to create backup directory. Exiting." >&2 # 输出到 stderr
    exit 1 # 异常退出
  fi
fi
# 2. 执行压缩
# -c: create (创建)
# -z: gzip (压缩)
# -f: file (指定文件名)
tar -czf "${BACKUP_DIR}/${ARCHIVE_FILE}" -C "${SOURCE_DIR}" .
# 3. 检查上一个命令是否成功
# $? 存储了上一个命令的退出状态码。0 代表成功。
if [ $? -eq 0 ]; then
  echo "Backup successful: ${BACKUP_DIR}/${ARCHIVE_FILE}"
else
  echo "Backup failed." >&2
  exit 1
fi

echo "Backup complete."
exit 0 # 正常退出
```