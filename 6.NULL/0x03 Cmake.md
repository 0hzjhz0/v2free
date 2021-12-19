# CMake By Example

## 0x01_helloworld

`Makefile` 基本格式：

```makefile
name: dependencies
		commands
```

例如：

```maefile
hello: main.cpp
		$(CXX) -o hello main.cpp # CXX 是 Make 的内置变量
		echo "OK"
```

构建 & 运行命令：

```shell
$ make hello
$ ./hello
```

## 0x01_helloworld

变量赋值：

```shell
CC := clang
CXX := clang++ # 可通过 make CXX=g++ 形式覆盖

objects := main.o
```

使用变量：

```makefile
hello: $(objects)
	$(CXX) -o $@ $(objects) # $@ 是自动变量，表示 target 名

main.o: main.cpp
	$(CXX) -c main.cpp
```

## 0x02_ask_for_answer

Make 可以自动推断 `.o` 文件需要用什么命令从什么文件编译：

```makefile
objects := main.o answer.o

answer: $(objects)
	$(CXX) -o $@ $(objects)

main.o: answer.hpp
answer.o: answer.hpp
```

## 0x03_switch_to_cmake

`CMakeLists.txt` 基本格式：

```
cmake_minimum_required(VERSION 3.9)
project(answer)

add_executable(answer main.cpp answer.cpp)
```

生成 & 构建 & 运行命令：

```
cmake -B build      # 生成构建目录，-B 指定生成的构建系统代码放在 build 目录
cmake --build build # 执行构建
./build/answer      # 运行 answer 程序
```

## 0x04_split_library

项目中可以复用的部分可以拆成 library：

```
add_library(libanswer STATIC answer.cpp)
```

`STATIC` 表示 `libanswer` 是个静态库。

使用（链接）library：

```
add_executable(answer main.cpp)
target_link_libraries(answer libanswer)
```

## 0x05_subdirectory

功能独立的模块可以放到单独的子目录：

```
.
├── answer
│  ├── answer.cpp
│  ├── CMakeLists.txt
│  └── include
│     └── answer
│        └── answer.hpp
├── CMakeLists.txt
└── main.cpp
# CMakeLists.txt
add_subdirectory(answer)

add_executable(answer_app main.cpp)
target_link_libraries(answer_app libanswer) # libanswer 在 answer 子目录中定义
# answer/CMakeLists.txt
add_library(libanswer STATIC answer.cpp)
target_include_directories(libanswer PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
```

`CMAKE_CURRENT_SOURCE_DIR` 是 CMake 内置变量，表示当前 `CMakeLists.txt` 文件所在目录，此处其实可以省略。

`target_include_directories` 的 `PUBLIC` 参数表示这个包含目录是 `libanswer` 的公开接口一部分，链接 `libanswer` 的 target 可以 `#include` 该目录中的文件。

## 0x06_use_libcurl

系统中安装的第三方库可以通过 `find_package` 找到，像之前的 `libanswer` 一样链接：

```
find_package(CURL REQUIRED)
target_link_libraries(libanswer PRIVATE CURL::libcurl)
```

`REQUIRED` 表示 `CURL` 是必须的依赖，如果没有找到，会报错。

`PRIVATE` 表示“链接 `CURL::libcurl`”是 `libanswer` 的私有内容，不应对使用 `libanswer` 的 target 产生影响，注意和 `PUBLIC` 的区别。

`CURL` 和 `CURL::libcurl` 是约定的名字，其它第三方库的包名和 library 名可在网上查

## 0x07_link_libs_in_same_root

可以链接同一项目中其它子目录中定义的 library：

```
# CMakeLists.txt
add_subdirectory(answer)
add_subdirectory(curl_wrapper)
add_subdirectory(wolfram)
# answer/CMakeLists.txt
add_library(libanswer STATIC answer.cpp)
target_include_directories(libanswer PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
target_link_libraries(libanswer PRIVATE wolfram)
# wolfram/CMakeLists.txt
add_library(wolfram STATIC alpha.cpp)
target_include_directories(wolfram PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
target_link_libraries(wolfram PRIVATE curl_wrapper)
# curl_wrapper/CMakeLists.txt
find_package(CURL REQUIRED)
add_library(curl_wrapper STATIC curl_wrapper.cpp)
target_include_directories(curl_wrapper
                           PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
target_link_libraries(curl_wrapper PRIVATE CURL::libcurl)
```

搞清楚项目各模块间的依赖关系很重要：



## 0x08_cache_string



