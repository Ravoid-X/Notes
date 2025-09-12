## 指令
1. ``cmake``\
（1）-G：用于指定生成器（generator），windows 下 cmake 默认使用 msvc，若要使用 mingw 需加上``-G "MinGW Makefiles"``\
（2）-B: 用于指定构建目录，``cmake -B path_to_build_directory``\
（3）-S：用于指定源代码目录，``cmake -S path_to_source_directory``
2. ``make``\
（1）-j：指定同时执行的命令数目，默认是系统允许的最大可能数目，有时会卡死\
（2）-s：取消命令执行过程中的打印\
（3）-k：执行命令错误时不终止 make 的执行，make 尽最大可能执行所有的命令，直至出现知名的错误才终止。
## MinGW编译
1. 在 mingw64/bin 目录下，将 mingw32-make.exe 复制一份重命名为 make
2. 在目录下新建一个 build 文件夹，并进入
3. 指令如下
```
cmake -G "MinGW Makefiles"
make
./xxx
```
## MVSC编译
1. 在目录下新建一个 build 文件夹，并进入
2. 指令如下
```
cmake ..
cmake --build .
./Debug/xxx
```
## CMakeLists 文件
1. ``cmake_minimum_required(VERSION 3.10)``
2. ``PROJECT(projectname [CXX] [C] [Java])``\
定义工程名字，并指定了工程的语言，支持语言的列表可以省略，默认情况下表示支持所有语言
3. ``SET(VAR [VALUE] [CACHE TYPE DOCSTRING [FORCE]])``\
显式的定义变量，如``SET(SRC_LIST main.c t1.c t2.c)``
4. ``MESSAGE([SEND_ERROR | STATUS | FATAL_ERROR] "message to display" ...)``\
向终端输出用户定义的信息，包含三种类型 SEND_ERROR（产生错误，生成过程被跳过）、SATUS（输出前缀为--的信息）和 FATAL_ERROR（立即终止所有cmake 过程）
5. ``ADD_EXECUTABLE(hello ${SRC_LIST})``\
定义了这个工程会生成一个文件名为 hello 的可执行文件，相关的源文件是 SRC_LIST 中定义的源文件列表
6. ``include_directories(xxx)``\
会为当前CMakeLists.txt的所有目标，以及之后添加的所有子目录的目标添加头文件搜索路径
7. ``target_include_directories(test xxx ${PROJECT_SOURCE_DIR}/include)``\
只会为指定目标包含头文件搜索路径。如果想为不同目标设置不同的搜索路径，target_include_directories 更合适。\
PRIVATE 则表示头文件只能由 test 使用，INTERFACE 则表示头文件只能由调用 test 的文件使用，PUBLIC 则 test 和任何调用 test 的文件都能使用头文件。
8. ``add_library(生成的库名称 STATIC/SHARED 源文件.cpp)``\
将源文件生成 静态/动态 库文件
9. ``target_link_libraries (库/可执行文件 library1 library2 ...)``\
为库或者可执行文件添加需要链接的库，需要放在add_executable之后
10.  ``add_subdirectory(source_dir)``\
向当前工程添加存放其他源文件的子目录，用于多目录下都有CMakeLists.txt文件的情况
11.  ``find_package(<NAME> 版本号 EXACT/QUIET/REQUIRED)``\
EXACT：要求该版本号必须精确匹配；QUIET：禁掉没有找到时的警告信息，如果没找到则忽略这一问题，继续执行；REQUIRED：如果包没有找到的话，CMake的过程会终止，并输出警告信息