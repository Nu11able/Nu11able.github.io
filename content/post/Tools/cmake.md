---
title: cmake使用记录
date: 2022-03-03
tags:
 - tools
 - cmake
categories:
 - tools
---

# cmake使用记录

## cmake使用模板
```cmake
#CMake的最低版本要求
cmake_minimum_required(VERSION 3.0)
 
#项目名称
project(tingx)
 
#设置版本是 Debug 还是 Release
set(CMAKE_BUILD_TYPE "Debug")
 
file(GLOB_RECURSE SRC_FILES ${PROJECT_SOURCE_DIR}/src/**.cc ${PROJECT_SOURCE_DIR}/src/**.hpp ${PROJECT_SOURCE_DIR}/src/**.h)

# 移除某些文件
file(GLOB_RECURSE MAINS src/*main.cc)
list(REMOVE_ITEM LIBRARY ${MAINS})

# msvc 按文件夹分组
macro(source_group_by_dir source_files)
    if(MSVC)
        set(sgbd_cur_dir ${CMAKE_CURRENT_SOURCE_DIR})
        foreach(sgbd_file ${${source_files}})
            string(REGEX REPLACE ${sgbd_cur_dir}/\(.*\) \\1 sgbd_fpath ${sgbd_file})
            string(REGEX REPLACE "\(.*\)/.*" \\1 sgbd_group_name ${sgbd_fpath})
            string(COMPARE EQUAL ${sgbd_fpath} ${sgbd_group_name} sgbd_nogroup)
            string(REPLACE "/" "\\" sgbd_group_name ${sgbd_group_name})
            if(sgbd_nogroup)
                set(sgbd_group_name "\\")
            endif(sgbd_nogroup)
            source_group(${sgbd_group_name} FILES ${sgbd_file})
        endforeach(sgbd_file)
    endif(MSVC)
endmacro(source_group_by_dir)

#设置包含目录
include_directories(${PROJECT_SOURCE_DIR}/src)

#设置输出目录
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/build)
 
#添加编译参数,比如add_definitions(-DDEBUG)将在gcc命令行添加DEBUG宏定义
add_definitions(-g)
 
#添加可执行文件
add_executable(tingx ${SRC_FILES})
add_executable(client ${PROJECT_SOURCE_DIR}/test/client.cc)
```


## cmake预定义变量
`以下假设项目文件夹路径为/home/name/project`
|变量|值|
|-|-|
|PROJECT_SOURCE_DIR|/home/name/project|

## 问题汇总

### cmake+vscode配置后源代码中显示头文件找不到
在项目文件夹下.vscode中settings.json中添加行"C_Cpp.default.configurationProvider": "vector-of-bool.cmake-tools"