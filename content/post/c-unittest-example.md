---
title: "C语言项目单元测试实践"
author: WingLim
date: 2020-12-22T21:50:57+08:00
slug: c-unittest-example
tags: 
- C
- CMocka
- CodeCov
- CMake
categories:
- Tech
typora-root-url: ../../static
---



## 前言

最近一段时间在写C语言的课程设计，之前在用 Golang 的时候，Golang 自带单元测试，用起来非常舒服，而C语言不使用框架写测试则会很麻烦，下面通过一个简单的项目来实践在C语言中进行单元测试。

项目中使用 [CMocka](https://cmocka.org) 作为单元测试框架，使用 [CodeCov](https://about.codecov.io) 检查代码覆盖率。

完整项目代码可以在 GitHub 上查看：[c-unittest-example](https://github.com/WingLim/c-unittest-example)



## 项目目录

```
.
├── CMakeLists.txt
├── Makefile
├── README.md
├── cmake
│   ├── CMocka.cmake
│   └── CodeCov.cmake
├── include
│   └── add.h
├── src
│   └── add.c
└── test
    ├── CMakeLists.txt
    ├── add_tests.c
    └── test.h

4 directories, 10 files
```

## 目录说明 

`cmake`: 存放 CMake 的模块文件，包括 CMocka 和 CodeCov。

`include`: 项目头文件
`src`: 项目源代码
`test`: 单元测试代码



## 项目设置文件

### `Makefile`

用于便携执行单元测试和构建程序

```makefile
.PHONY: cmake test

BUILD_TYPE ?= Debug
BUILD_DIR ?= cmake-build-$(shell echo $(BUILD_TYPE) | tr '[:upper:]' '[:lower:]')
CODECOV ?= OFF
IWYU ?= ON

TEST_SUITES = add_tests

# 清理文件
clean:
	@rm -rf $(BUILD_DIR)

# 创建 cmake-build-debug，并在里面执行 cmake
cmake:
	@mkdir -p $(BUILD_DIR) && cd $(BUILD_DIR) && cmake -DCODE_COVERAGE=$(CODECOV) -DIWYU=$(IWYU) -DCMAKE_BUILD_TYPE=$(BUILD_TYPE) -j 4 ..

# 构建文件
build: cmake
	@cd $(BUILD_DIR) && make project -j 4

# 进行单元测试
test:
	@cd $(BUILD_DIR) && make $(TEST_SUITES) test CTEST_OUTPUT_ON_FAILURE=TRUE

# 测试代码覆盖率
coverage: test
	@cd $(BUILD_DIR) && make codecov CMAKE_BUILD_TYPE=$(BUILD_TYPE)
```



### `cmake/CMocka.cmake`

添加 CMocka 到项目中

```cmake
include(ExternalProject)

# 添加额外的项目，即 CMocka
ExternalProject_Add(cmocka_ep
        URL https://git.cryptomilk.org/projects/cmocka.git/snapshot/cmocka-1.1.5.tar.gz

        CMAKE_ARGS -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
        -DCMAKE_SYSTEM_NAME=${CMAKE_SYSTEM_NAME}
        -DBUILD_STATIC_LIB=ON
        -DCMAKE_ARCHIVE_OUTPUT_DIRECTORY_DEBUG:PATH=Debug
        -DCMAKE_ARCHIVE_OUTPUT_DIRECTORY_RELEASE:PATH=Release
        -DUNIT_TESTING=OFF

        BUILD_COMMAND $(MAKE) cmocka-static
        INSTALL_COMMAND "")

# 全局作用域下添加 cmocka 静态库
add_library(cmocka STATIC IMPORTED GLOBAL)
# 获取二进制文件夹路径
ExternalProject_Get_Property(cmocka_ep binary_dir)

# 分别设置正常、Debug、Release的导入路径
set_property(TARGET cmocka PROPERTY IMPORTED_LOCATION "${binary_dir}/src/libcmocka-static.a")
set_property(TARGET cmocka PROPERTY IMPORTED_LOCATION_DEBUG "${binary_dir}/src/Debug/libcmocka-static.a")
set_property(TARGET cmocka PROPERTY IMPORTED_LOCATION_RELEASE "${binary_dir}/src/Release/libcmocka-static.a")

# 将 cmocka_ep 依赖添加到 cmocka
add_dependencies(cmocka cmocka_ep)

# 获取 cmocka_ep 的源文件路径
ExternalProject_Get_Property(cmocka_ep source_dir)
# 全局作用域下设置头文件引入路径
set(CMOCKA_INCLUDE_DIR ${source_dir}/include GLOBAL)
```



### `cmake/CodeCov.cmake`

添加 CodeCov 到项目中

```cmake
# 寻找 gcovr 
find_program(GCOVR_PATH gcovr PATHS ${CMAKE_SOURCE_DIR}/scripts/test)

# 设置 CMake 时的参数
set(CMAKE_C_FLAGS_CODECOV "-O0 -g --coverage" CACHE INTERNAL "")
# 设为高级变量
mark_as_advanced(CMAKE_C_FLAGS_CODECOV)

# 确保是在使用 Debug 来构建 
string(TOLOWER ${CMAKE_BUILD_TYPE} current_build_type)
if (NOT current_build_type STREQUAL "debug")
    message(WARNING "Code coverage results with an optimised (non-Debug) build may be misleading")
endif ()

# 如果使用的是 GNU 则链接 gcov 库文件
if (CMAKE_C_COMPILER_ID STREQUAL "GNU")
    link_libraries(gcov)
endif ()

# 添加 CodeCov 编译参数到 CMake
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${CMAKE_C_FLAGS_CODECOV}")
message(STATUS "Appending code coverage compiler flags: ${CMAKE_C_FLAGS_CODECOV}")

# 添加自定义 target
add_custom_target(codecov
        WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
        COMMENT "Generating code cov report at ${PROJECT_BINARY_DIR}/codecov.xml"
        # 在 SHELL 中展示代码覆盖率总结
        COMMAND ${GCOVR_PATH} --exclude-throw-branches -r .. --object-directory "${PROJECT_BINARY_DIR}" -e ".*/test/.*" -e ".*/usr/.*" --print-summary
        # 输出到 codecov.xml
        COMMAND ${GCOVR_PATH} --xml --exclude-throw-branches -r .. --object-directory "${PROJECT_BINARY_DIR}" -e ".*/test/.*" -e ".*/usr/.*" -o codecov.xml)
```



### `CMakeLists.txt`

设置当前项目

```cmake
# CMake 最低版本要求
cmake_minimum_required(VERSION 3.17)
# 设置 CMake 模块目录
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# 项目名
project(project)

# 设置 CodeCov
option(CODE_COVERAGE "Enable coverage reporting" OFF)
if (CODE_COVERAGE AND CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
    include(CodeCov)
endif ()

# 设置 include-what-you-use
option(IWYU "Run include-what-you-use with the compiler" OFF)
if (IWYU)
		# 寻找 iwyu
    find_program(IWYU_COMMAND NAMES include-what-you-use iwyu)
    if (NOT IWYU_COMMAND)
        message(FATAL_ERROR "CMAKE_IWYU is ON but include-what-you-use is not found!")
    endif ()
    # 添加饮用
    set(CMAKE_C_INCLUDE_WHAT_YOU_USE "${IWYU_COMMAND};-Xiwyu")
endif ()

# 启用测试
enable_testing()

# 将源码添加到项目
add_library(project
        src/add.c)

# 设置头文件目录
target_include_directories(project
        PUBLIC
        $<INSTALL_INTERFACE:include>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/src)

# 添加单元测试子文件
add_subdirectory(test)
```



### `test/CMakeLists.txt`

设置项目的单元测试

```cmake
# 引入 CMocka.cmake
include(CMocka)

# 引入 Cmocka 头文件
include_directories(${CMOCKA_INCLUDE_DIR})

# 用于快捷添加测试，_testName 为单元测试的文件名，不带后缀
function(add_test_suite _testName)
    add_executable(${_testName} ${_testName}.c)
    target_link_libraries(${_testName} project cmocka)
    add_test(${_testName} ${CMAKE_CURRENT_BINARY_DIR}/${_testName})
endfunction()

add_test_suite(add_tests)
```



## 项目源码

这里实现一个可以传可变参数的加法函数。

### `add.h`

```c
#ifndef PROJECT_ADD_H
#define PROJECT_ADD_H

int add(int count, ...);

#endif //PROJECT_TEST_H
```



### `add.c`

```c
#include "add.h"
#include <stdarg.h>

int add(int count, ...) {
    va_list arg_ptr;
    va_start(arg_ptr, count);
    int sum = 0;
    for (int i = 0; i < count; ++i) {
        int tmp = va_arg(arg_ptr, int);
        sum += tmp;
    }
    va_end(arg_ptr);
    return sum;
}
```



### `test/test.h`

引入 CMocka 所需的头文件

```c
#ifndef PROJECT_TEST_H
#define PROJECT_TEST_H

#include <stdarg.h>
#include <stddef.h>
#include <setjmp.h>
#include <cmocka.h>

int run_all_tests(void);


int main() {
    return run_all_tests();
}

#endif //PROJECT_TEST_H
```



### `test/add_tests.c`

```c
#include "test.h"
#include "add.h"

static void test_add_1(void **state) {
    int sum = add(1, 1);
    assert_int_equal(1, sum);
}

static void test_add_2(void **state) {
    int sum = add(2, 1, 2);
    assert_int_equal(3, sum);
}

static void test_add_more(void **state) {
    int sum = add(6, 1, 2, 3, 4, 5, 6);
    assert_int_equal(21, sum);
}

int run_all_tests() {
    const struct CMUnitTest tests[] = {
            cmocka_unit_test(test_add_1),
            cmocka_unit_test(test_add_2),
            cmocka_unit_test(test_add_more),
    };

    return cmocka_run_group_tests(tests, NULL, NULL);
}
```



## 进行测试

项目根目录下执行

```shell
CODECOV=ON IWYU=OFF make cmake coverage
```

执行效果如下![image-20201222220536632](/images/image-20201222220536632.png)



到 [CodeCov](https://about.codecov.io) 上创建项目，获取 TOKEN，执行如下命令上传测试报告

```shell
echo "YOUR TOKEN" > .cc_token
bash <(curl -s https://codecov.io/bash) -f cmake-build-debug/codecov.xml -t @.cc_token
```



项目也可以使用 GitHub Actions 进行自动化构建和上传

在项目中设置 `secrets.CODECOV_TOKEN` 即可，详细设置可以查看 `.github/workflows/workflow.yml`