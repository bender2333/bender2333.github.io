---
layout: post
title: Cmake build sytems
categories: [cmake,技术翻译]
description: some word here
keywords: cmake, build
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## 介绍
cmake 构建系统能够以高级抽象的方式,构建出可执行、库或者其他定制化目标。

## 二进制目标
```
添加一个静态库
add_library(archive archive.cpp zip.cpp lzma.cpp)
添加一个二进制目标
add_executable(zipapp zipapp.cpp)
链接
target_link_libraries(zipapp archive)
```

## 二进制库
### 一般库

```
#动态库
add_library(archive SHARED archive.cpp zip.cpp lzma.cpp)
#静态库,默认不加STATIC也是静态库
add_library(archive STATIC archive.cpp zip.cpp lzma.cpp)
```

MODULE库
```
add_library(archive MODULE 7z.cpp)
```
与shared/static不同,module库一般不会被链接,即不会被用作target_link_libraies()的参数,它通常用来插件技术上,作为运行的加载。如果其不包含任何非托管符号,则要求不能是shared库,因为cmake要求shared库至少导出一个符号

## object libraries
这种库在之前的sdk项目中经常使用, 它是一种非归档类型的对象文件,由一些给定的源文件称其。通常作为一个中间编译模块,用于其他目标的输入。在msvc中,以static lib的形式实现的。
```
添加一个obj库
add_library(archive OBJECT archive.cpp zip.cpp lzma.cpp)
通过迭代器使用
add_library(archiveExtras STATIC $<TARGET_OBJECTS:archive> extras.cpp)
add_executable(test_exe $<TARGET_OBJECTS:archive> test.cpp)
```
因为其定义为中间目标,因为不能用一些特定阶段,比如add_custom_command、file等


## Pseudo Targets
有时候目标类型并不需要作为构建系统的输出,而是作为编译依赖、别名

### imported targets
这个常用于导入三方库使用,该目标表示一个预先存在的依赖项,通常由上游包定义，并且不太会由变化。
当声明一个导入的目标后,就可以使用target_compile_definitions(), target_include_directories(), target_compile_options() or target_link_libraries()
来调整目标的属性

```
add_library(foo STATIC IMPORTED)
set_property(TARGET foo PROPERTY IMPORTED_LOCATION "/path/to/libfoo.a")
```
windows
```
add_library(bar SHARED IMPORTED)
set_property(TARGET bar PROPERTY
             IMPORTED_LOCATION "c:/path/to/bar.dll")
set_property(TARGET bar PROPERTY
             IMPORTED_IMPLIB "c:/path/to/bar.lib")
add_executable(myexe src1.c src2.c)
target_link_libraries(myexe PRIVATE bar)
```

 multiple configurations 
 ```
find_library(math_REL NAMES m)
find_library(math_DBG NAMES md)
add_library(math STATIC IMPORTED GLOBAL)
set_target_properties(math PROPERTIES
  IMPORTED_LOCATION "${math_REL}"
  IMPORTED_LOCATION_DEBUG "${math_DBG}"
  IMPORTED_CONFIGURATIONS "RELEASE;DEBUG"
)
add_executable(myexe src1.c src2.c)
target_link_libraries(myexe PRIVATE math)
```

## Alias Targets
一个ALIAS目标是一个名称，它可以在只读上下文中与二进制目标名称互换使用。ALIAS目标的一个常见场景是，伴随库的单元测试可执行文件，它们可能是同一构建系统的一部分，或者根据用户配置单独构建。
```
add_library(lib1 lib1.cpp)
install(TARGETS lib1 EXPORT lib1Export ${dest_args})
install(EXPORT lib1Export NAMESPACE Upstream:: ${other_args})
#将lib1作为Upsteam的一部分
add_library(Upstream::lib1 ALIAS lib1)
```

ALIAS 目标不可变、不可安装或不可导出。它们完全局限于构建系统描述内部。通过读取目标的 ALIASED_TARGET 属性，可以测试一个名称是否为 ALIAS 名称：

## Interface Libraries

一个 INTERFACE 库目标不编译源代码，也不会在磁盘上生成库文件，因此它没有位置（LOCATION）。
它可以指定使用需求，例如 INTERFACE_INCLUDE_DIRECTORIES、INTERFACE_COMPILE_DEFINITIONS、INTERFACE_COMPILE_OPTIONS、INTERFACE_LINK_LIBRARIES、INTERFACE_SOURCES 和 INTERFACE_POSITION_INDEPENDENT_CODE。只有 target_include_directories()、target_compile_definitions()、target_compile_options()、target_sources() 和 target_link_libraries() 命令的 INTERFACE 模式可以与 INTERFACE 库一起使用。
自 CMake 3.19 起，INTERFACE 库目标可以选择包含源文件。包含源文件的接口库将在生成的构建系统中作为构建目标包含进来。它不编译源文件，但可以包含自定义命令来生成其他源文件。此外，IDE 会将源文件作为目标的一部分显示，以便进行交互式阅读和编辑。
INTERFACE 库的一个主要用途是仅头文件库。自 CMake 3.23 起，可以通过使用 target_sources() 命令将头文件添加到头文件集中，从而将头文件与库关联起来。

```
#将eigen添加为interface lib
add_library(Eigen INTERFACE)
#添加头文件
target_sources(Eigen PUBLIC
  FILE_SET HEADERS
    BASE_DIRS src
    FILES src/eigen.h src/vector.h src/matrix.h
)

add_executable(exe1 exe1.cpp)
target_link_libraries(exe1 Eigen)
```
当我们在这里指定 FILE_SET 时，我们定义的 BASE_DIRS 会自动成为目标 Eigen 的使用需求中的包含目录。这些来自目标的使用需求在编译时被消耗和使用，但对链接没有影响。
另一个用例是采用完全以目标为中心的设计来处理使用需求：
```
add_library(pic_on INTERFACE)
set_property(TARGET pic_on PROPERTY INTERFACE_POSITION_INDEPENDENT_CODE ON)
add_library(pic_off INTERFACE)
set_property(TARGET pic_off PROPERTY INTERFACE_POSITION_INDEPENDENT_CODE OFF)

add_library(enable_rtti INTERFACE)
target_compile_options(enable_rtti INTERFACE
  $<$<OR:$<COMPILER_ID:GNU>,$<COMPILER_ID:Clang>>:-rtti>
)

add_executable(exe1 exe1.cpp)
target_link_libraries(exe1 pic_on enable_rtti)
```
这样一来,exe1的构建规范完全通过链接目标来表达,而特定于编译器的标志则被放到一个Interface库目标中了,也就是不影响链接的目标


> 疑问？
inferface只是源码,并不会构建,所以上述的INTERFACE_POSITION_INDEPENDENT_CODE配置会是在哪里生效的？按上述说法exe1并不会受INTERFACE_POSITION_INDEPENDENT_CODE的影响。

QA:根据chatgpt的说法:

>当你在一个 INTERFACE 库上设置 INTERFACE_POSITION_INDEPENDENT_CODE 属性时，任何链接到这个接口库的目标都会继承这个属性。因此，exe1 目标将会被编译成位置无关代码（PIC），如果它链接了 pic_on。

interface lib还可以被Install和export

```
add_library(Eigen INTERFACE)

target_sources(Eigen INTERFACE
  FILE_SET HEADERS
    BASE_DIRS src
    FILES src/eigen.h src/vector.h src/matrix.h
)

install(TARGETS Eigen EXPORT eigenExport
  FILE_SET HEADERS DESTINATION include/Eigen)
install(EXPORT eigenExport NAMESPACE Upstream::
  DESTINATION lib/cmake/Eigen
)
```