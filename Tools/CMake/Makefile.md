## 概述
1. 一个文本文件，基于依赖关系和文件时间戳，包含了一系列的规则，用于告诉 make 工具如何编译和链接一个程序
2. make 工具通过比较文件（目标文件和依赖文件）的最后修改时间，来决定哪些文件需要重新生成
# 规则
## 基本结构
```
目标 (target): 依赖 (prerequisites)
	[Tab键] 命令 (command)
```
### 目标
通常是要生成的文件名，比如可执行文件或目标文件（.o）
1. 也可以是伪目标，不是一个真实的文件名，而是一个动作的标签，声明方式：`.PHONY: 目标名`，告诉 make 跳过文件系统检查
2. 是为了避免冲突，假设有一个 clean 目标用来删除文件但忘记写`.PHONY: clean`。当目录里出现叫 clean 的文件，运行 make clean 时。\
3. make 会检查 clean 文件的依赖（没有依赖），发现 clean 文件已经存在，于是认为目标已是最新，什么也不做（rm 命令不会被执行）\
4. 常见伪目标： all（默认目标）, clean（清理）, install（安装）
### 依赖
1. 构建目标所需要的文件或其他目标。
2. make 会在执行“命令”之前，先会遍历所有依赖。\
（1）如果任何一个依赖文件不存在，会查找其他规则来先构建那个依赖\
（2）如果所有依赖都存在，会比较“目标”文件的时间戳和每一个“依赖”文件的时间戳\
（3）如果任何一个依赖比目标更新（时间戳更晚），或者目标文件不存在，会执行该规则的“命令”
### 命令
是生成目标所执行的 Shell 命令。每一行命令都必须以一个 Tab 键开头，空格会报错
1. make 会为每一行命令启动一个全新的、独立的子 Shell 来执行
```makefile
test:
    cd build # 在一个子 Shell 中执行，该 Shell 退出后状态全部丢失
    pwd # 运行 make test，输出仍然是当前目录，而不是 build 目录
```
2. 修正方法
```makefile
test:
    cd build; pwd 或者
    cd build && \
    pwd

```
### 命令修饰符
1. @：放在命令行的最前面，make 在执行时不会把这行命令本身打印到屏幕上
2. `-` (忽略错误)：放在命令行的最前面。即使这条命令执行失败返回非零退出码），也会继续执行

# 变量
## 语法
### `=`
1. make 在使用这个变量时（在规则的命令中，或在其他变量定义中）才对其进行展开
2. 如果它包含了对其他变量的引用，会一层层递归地展开
```makefile
X = $(Y)
Y = Hello

all:
    @echo $(X)  # 即使 X 定义时 Y 还没值，但在 all 规则中展开 $(X) 时，make 会去找 $(Y)
```
3. 无限循环：`A = $(B)$ 和 $B = $(A)` 会导致 make 致命错误
4. 意外追加：`CFLAGS = $(CFLAGS) -g` 这样的写法会陷入无限递归
5. 性能问题：
```makefile
VERSION = $(shell git rev-parse HEAD)
# 每次使用 $(VERSION) 都会重新执行一次 'git rev-parse HEAD'
```
### `:=`
1. make 在定义这一行时，就立刻计算出变量的值，并将其固定下来
```makefile
X := $(Y)
Y := Hello

all:
    @echo $(X)  # 输出 "" (空字符串)
```
2. 性能高：`VERSION := $(shell git rev-parse HEAD)` 只会执行一次 Shell 命令
3. 安全追加：`CFLAGS := $(CFLAGS) -g` 是合法的（虽然 += 更好）
### `?=`
1. 只有当这个变量在之前没有被定义过时，才给它赋值
2. 可以设置默认值，并允许用户从命令行覆盖
```makefile
# Makefile 内部
CC ?= gcc

all:
    @echo "编译器: $(CC)"
```
### `+=`
1. 向变量的末尾追加字符串（自动加一个空格）
2. 会继承变量赋值和扩展的方式
```makefile
OBJS := main.o
OBJS += utils.o # $(OBJS) 的值现在是 "main.o utils.o"
# 如果 OBJS 是用 := 定义的，+= 也会立即展开
# 如果用 = 定义，+= 也会被推迟展开
```
## 变量优先级
从高到低
1. 命令行参数： make CC=clang
2. Makefile 内部定义： CC := gcc (除非内部用的是 ?=)
3. 环境变量： export CC=tcc; make
4. make 的内置变量： make 默认 CC 是 cc。
## 自动变量
自动变量是 make 提供的“上下文变量”。值在每一个规则中都会根据该规则的“目标”和“依赖”自动变化。
### `$@`
1. 含义：规则的完整目标名称
2. 示例：在 build/main.o: src/main.c 规则中，$@ 的值是 build/main.o
### `$<`
1. 含义：规则的第一个依赖的名称
2. 示例：在 my_program: main.o utils.o 规则中，$< 的值是 main.o
### `$^`
1. 含义：规则的所有依赖的完整列表，以空格分隔，并自动去除重复项
2. 示例：在 my_program: main.o utils.o main.o 规则中，$^ 的值是 main.o utils.o
3. 用途：链接时使用，不需要关心依赖写了几遍
### `$?`
1. 含义：所有比目标更新的依赖的列表
2. 用途：构建静态库（.a 文件）。只想把新编译的 .o 文件添加到库中，而不是每一次都把所有 .o 文件加进去
```makefile
libutils.a: utils.o string.o
    # ar 是归档工具。r(替换), c(创建), s(创建索引)
    # $? 只包含被更新的 .o 文件
    ar rcs $@ $?
```
### `$+`
1. 含义：规则的所有依赖的完整列表，保留原始顺序和重复项
2. 用途：链接器对库的顺序和重复很敏感
```makefile
# 假设你需要链接 a 库，然后是 b 库，然后再链接一次 a 库
my_program: liba.a libb.a liba.a
    # 如果用 $^, 会变成 'liba.a libb.a'，链接失败
    # 必须用 $+
    gcc -o $@ $^
    gcc -o $@ $+  # 正确
```
### `$*`
1. 含义：在“模式规则”中，代表“主干”的部分（即 % 匹配到的部分）。
2. 示例：在 %.o: %.c 规则中，如果目标是 main.o，$* 的值就是 main。
3. 用途：生成相关文件，比如 %.s: %.c (生成汇编) -> gcc -S $< -o $*.s
### 修饰符
最强大的功能，用于处理复杂的目录结构（如“源码外构建”）
1. D (Directory)：提取目录部分。
2. F (File)：提取文件名部分。
3. 假设 $@ 的值是 $build/obj/main.o：\$(@D)$ 的值是 build/obj；$(@F)$的值是 main.o
# 模式与函数
## 模式规则
不需要为每个文件都写规则，而是定义一个“模板”。% 是通配符
### 示例
```makefile
# 模板：如何从一个同名的 .c 文件构建一个 .o 文件
%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@
```
### 源码外构建
假设源文件在 src/，目标文件放在 build/
```makefile
BUILD_DIR := build
SRC_DIR := src

# 目标：build/main.o
# 依赖：src/main.c
$(BUILD_DIR)/%.o: $(SRC_DIR)/%.c
	# 目标目录 build/ 可能不存在，先创建它
	# $(@D) 在这里展开为 "build"
	@mkdir -p $(@D)
	
	# $< 展开为 "src/main.c"
	# $@ 展开为 "build/main.o"
	$(CC) $(CFLAGS) -c $< -o $@
```
## vpath 和 VPATH
### VPATH
1. 一个全局变量，设置所有依赖的搜索路径，用冒号或空格分隔。
2. 如 VPATH = src:include
### vpath
1. vpath %.c src (只在 src 目录找 .c 文件)
2. vpath %.h include (只在 include 目录找 .h 文件)
## 常用函数
### $(wildcard 模式)
搜索文件，会扫描硬盘，返回匹配到的文件列表
```makefile
# 找到 src/ 下所有的 .c 文件
SRCS := $(wildcard src/*.c)
# SRCS 的值可能是 "src/main.c src/utils.c"
```
### $(patsubst 模式, 替换, 文本)
```makefile
# SRCS = "src/main.c src/utils.c"
# 我们想得到 "build/main.o build/utils.o"
OBJS := $(patsubst src/%.c, build/%.o, $(SRCS))
```
### $(shell 命令)
1. 执行 Shell 命令并将其标准输出作为函数的值
2. 尽量只在 := 赋值时使用，否则会有性能问题
```makefile
KERNEL := $(shell uname -r)
```

# 示例
```makefile
# 1. 变量定义 (使用 := 和 ?=)
# -----------------------------------------------------

# 编译器和标志
CC ?= gcc
# CFLAGS: C 编译选项
# LDFLAGS: 链接选项
# LIBS: 链接时要包含的库 (例如 -lm 表示链接数学库)
CFLAGS := -g -Wall -Iinclude
LDFLAGS :=
LIBS := -lm

# 目录
SRC_DIR := src
BUILD_DIR := build
TARGET_DIR := bin

# 最终目标
TARGET := $(TARGET_DIR)/my_program

# 2. 自动文件搜寻 (使用函数)
# -----------------------------------------------------
# 查找所有 .c 源文件
# SRCS = "src/main.c src/utils.c src/math/vector.c"
SRCS := $(wildcard $(SRC_DIR)/*.c $(SRC_DIR)/*/*.c)

# 自动推导 .o 目标文件列表
# OBJS = "build/main.o build/utils.o build/math/vector.o"
OBJS := $(patsubst $(SRC_DIR)/%.c, $(BUILD_DIR)/%.o, $(SRCS))

# 自动推导 .d 依赖文件列表 (用于头文件自动依赖)
DEPS := $(OBJS:%.o=%.d)

# 3. 核心规则
# -----------------------------------------------------

# 伪目标：all, clean
.PHONY: all clean

# 默认目标：第一个规则是 'make' 的默认目标
all: $(TARGET)

# 链接规则：
# 目标：bin/my_program
# 依赖：build/main.o build/utils.o ...
$(TARGET): $(OBJS)
	@echo "--- 正在链接: $@ ---"
	# 创建目标目录
	@mkdir -p $(@D)
	# $^ 包含所有 .o 文件
	$(CC) $(LDFLAGS) $^ -o $@ $(LIBS)
	@echo "--- 构建完成 ---"

# 模式规则：编译 .c -> .o
# 目标：build/math/vector.o
# 依赖：src/math/vector.c
$(BUILD_DIR)/%.o: $(SRC_DIR)/%.c
	@echo "--- 正在编译: $< ---"
	# 创建 .o 文件的目录
	@mkdir -p $(@D)
	# $< 是第一个依赖 (src/math/vector.c)
	# $@ 是目标 (build/math/vector.o)
	$(CC) $(CFLAGS) -c $< -o $@

# 4. 高级：自动头文件依赖
# -----------------------------------------------------
# 这个规则告诉 make，当 .o 文件不存在时，
# 它也依赖于 .d 文件 (依赖描述文件)
$(BUILD_DIR)/%.d: $(SRC_DIR)/%.c
	@echo "--- 生成依赖: $< ---"
	@mkdir -p $(@D)
	# -MMD: 生成依赖规则，但不包括系统头文件
	# -MF $@: 将生成的规则输出到 $@ (即 .d 文件)
	$(CC) $(CFLAGS) -MMD -MF $@ -c $<

# 如果 .d 文件存在，就把它们包含进来
# 'include' 指令会把这些 .d 文件的内容当作 Makefile 的一部分
# .d 文件的内容可能是：
# build/main.o: src/main.c include/utils.h
# 这样 make 就知道，如果 utils.h 变了，main.o 也要重新编译
-include $(DEPS)

# 5. 清理规则
# -----------------------------------------------------
clean:
	@echo "--- 正在清理 ---"
	# - (忽略错误) r(递归) f(强制)
	-rm -rf $(BUILD_DIR) $(TARGET_DIR)
```