## 概述
查找包有两种方法
### Config 模式（默认）
1. 会去寻找 XXX.cmake 文件
2. 使用大写的 find_package(MySQL ...) 时，强制 CMake 只使用 Config 模式。
### Module 模式
1. 通常在使用小写的 find_package(aaa ...) 时调用
2. 或者 find_package(MySQL REQUIRED MODULE)

## 核心问题
MySQL 8.4 与 CMake 3.15，会报错 "No "FindMySQL.cmake" found in CMAKE_MODULE_PATH"
### 原因
1. MySQL 8.4 没有提供现代 CMake 所需的 MySQLConfig.cmake 文件，只提供了旧的 mysql_config 帮助程序
2. CMake 3.15+ 已经移除了旧的 FindMySQL.cmake 模块，只支持现代的 MySQLConfig.cmake 文件
### 解决
1. 既然 find_package 无法工作，就完全绕过它。
2. 修改 CMakeLists.txt，让它自己去运行 /usr/bin/mysql_config
3. 将其输出结果直接保存到 CMake 变量（MySQL_INCLUDE_DIRS 和 MySQL_LIBRARIES）中
```
if(USE_MYSQL)
    message(STATUS "MYSQL is ENABLED")
    # 1. 找到已经安装的 mysql_config 程序
    find_program(MYSQL_CONFIG_EXECUTABLE mysql_config)
    if(NOT MYSQL_CONFIG_EXECUTABLE)
        message(FATAL_ERROR "mysql_config executable not found. Cannot configure MySQL.")
    endif()
    # 2. 运行 "mysql_config --variable=pkgincludedir" 来获取头文件目录
    execute_process(
        COMMAND ${MYSQL_CONFIG_EXECUTABLE} --variable=pkgincludedir
        OUTPUT_VARIABLE MySQL_INCLUDE_DIRS
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    # 3. 运行 "mysql_config --libs" 来获取所有链接器标志
    execute_process(
        COMMAND ${MYSQL_CONFIG_EXECUTABLE} --libs
        OUTPUT_VARIABLE MySQL_LIBRARIES
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    message(STATUS "Found MySQL Includes: ${MySQL_INCLUDE_DIRS}")
    message(STATUS "Found MySQL Libs: ${MySQL_LIBRARIES}")
else()
    message(STATUS "MYSQL is DISABLED")
endif()
```