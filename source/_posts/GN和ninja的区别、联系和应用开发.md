---
title: GN和Ninja构建工具的基础入门、进阶和应用开发
date: 2023-12-24
tags:
categories:
- [LiteOS]
description: |
  &emsp;&emsp;GN是一个用于生成工程构建配置文件的工具，而Ninja工具用于替代Make、执行实际构建过程，其构建过程快速、高效，二者通常一起使用。<br>
  &emsp;&emsp;要移植鸿蒙LiteOS-M、了解其源码工程之构建，那么熟悉GN和Ninja的编译构建方式是必要的，本文介绍个人对GN与Ninja的学习与理解，主要目的为辅助理解OpenHarmony-LiteOS-M工程。
---

- 本文是个人对GN文件的理解，可能知识体系上有些凌乱分散，与官方表述上有些许出入
- 文章主要为谈及GN文件的常见重要概念，并基于此引申出代码示例及解释

## 基本介绍

GN文件使用GN语言编写，其是一种基于脚本的语言，用于定义构建系统的规则和目标
- gn即Generate ninja，gn与ninja的关系 类似于 cmake与make
- GN文件可以用于构建各种类型的项目，包括C++、Java、Python项目
<br>

ninja用于执行GN文件定义的规则和目标，其是一个C++编写的构建工具，可以生成高效的构建命令，提高构建速度
<br>


## 基本概念与语法

.gn是源文件，.gni是头文件，源文件可通过import引用头文件，也可使用import导入源文件
  > 如：`import("//build/config/c++/c++.gni")`
<br>

GN的基本概念包括：**变量、函数、目标、规则、配置项、模块**
- 变量：用于存储值
- 函数：用于执行操作
- 目标：用于表示构建过程中的某个对象，例如文件、库或可执行文件；**目标是GN构建系统的基本单位**
- 规则：用于定义目标的构建方式
- 配置项：用于控制构建过程的行为
- 模块：一种组织和管理代码的方式，可以包含变量、函数、目标、规则等，便于模块化地管理和复用构建配置

### 变量

GN中的变量可以使用set_default_variable()函数进行定义。变量的值可以是字符串、数字、布尔值或列表
- 变量可以存储构建过程中使用的常量值，如 参数、目标类型、选项
- 如`set_default_variable("target_os", "linux")`，设置操作系统为linux

<br>

**内置变量：**
```
target_os：当前目标的操作系统
target_cpu：当前目标的处理器架构
target_toolchain：当前目标的工具链
target_out_dir：当前目标的输出目录，默认值是//out/target_name/
target_gen_dir：当前目标的生成目录
root_build_dir：构建系统的根目录
buildconfig：构建系统的配置文件
```
<br>

**变量类型：数字、字符串、布尔值、列表、映射**
```
x = "Hello, world!"
y = 10
z = true

// 定义一个名为 list 的列表变量
list = [ "a", "b", "c" ]

// 定义一个名为 map 的映射变量
map = {
"key1" = "value1",
"key2" = "value2"
}
```
<br>

**`${}`--字符串插值操作符用于在字符串中插入变量、表达式或函数调用的值，如：**
```
x = 10
println("The value of x is ${x}")
```

### 函数

GN的函数形式与C语言的类似，由函数名、参数列表和函数体组成
```
function_name(parameter_list) {
// 函数体
}
```
用户可以使用内置函数或自定义函数

内置函数包括有如下：
```
assert()：断言函数，如果条件不成立，则抛出异常
concat()：字符串拼接函数
defined()：判断变量是否定义了的函数
dirname()：获取目录名的函数
file()：获取文件名的函数
join()：字符串连接函数
path()：获取路径名的函数
read_file()：读取文件的函数
write_file()：写入文件的函数
```
<br>

**其中，read_file()示例如下：**
```
# config.json
{
"product_name": "My Product",
"product_version": "1.0.0"
}

---
# 读取product_config_path目录下的config.json文件，并将其内容以JSON格式返回
product_cfg = read_file("${product_config_path}/config.json", "json")

# 将product_name属性的值赋值给变量product_name
product_name = product_cfg["product_name"]
```
<br>

用户自定义函数以及调用，示例如下：
```
// a.gn
function add(x, y) {
return x + y;
}
---
// b.gn
import("a.gn")

println("The sum of 1 and 2 is ${add(1, 2)}")
```


### 目标

目标是构建系统要生成的文件或目录，其可以是可执行文件、库、源代码文件、文档等
- BUILD.gn是GN构建系统默认使用的文件名
- 目标使用`target()`语句定义，语法如下：
  ```
  target(target_name) {
  // 目标配置
  }
  ```
<br>

目标名称必须唯一，可以有以下属性配置项定义：deps-依赖关系、type-目标类型、output-目标输出、public-目标是否公开、configs-目标配置
- 子目标可以使用deps属性指定
```
target("my_executable") {
type = "executable"       // 指定目标类型为可执行文件
sources = ["main.cc"]     // 指定目标所需要的源文件列表
deps = ["my_library"]     // 依赖关系，系统先构建此子目标
}
```
<br>

GN目标可以使用`build`命令构建
- 如：在当前路径下有BUILD.gn文件，其定义了名为`my_executable`的目标
- 则在命令行输入`gn build my_executable`表示构建该目标
- `gn build`表示构建所有目标
- `gn build my_executable:my_sub`表示构建目标的子目标my_sub


### 配置项

配置项用于配置GN构建系统的行为，其可以控制：构建目标的类型和输出、构建规则的行为、构建系统的环境
- 其可以分为：**目标配置项、规则配置项、环境配置项**
- 其与目标的联系为：配置项用于定义目标的行为，**目标的配置项可以定义在目标的gn配置文件中**
<br>

常用的配置项如下：
```
target_type：定义目标的类型
output：定义目标的输出
deps：定义目标的依赖关系
public：定义目标是否公开
configs：定义目标的配置
rule_type：定义规则的类型
deps：定义规则的依赖关系
inputs：定义规则的输入
outputs：定义规则的输出
action：定义规则的操作
script：定义规则的脚本
args：定义规则的参数
vars：定义规则的变量
```

`config()`函数用于创建配置，其可以指定**配置名称、编译选项、链接选项、依赖关系、宏定义、头文件目录、公开的配置、可见性**等<br>

代码应用示例如下：
```
config(name) {
  options = []
}

config("my_config") {
  cflags = ["-Wall"]
}

# 将 my_config 配置应用于 my_target 目标
executable("my_target") {
  configs = ["my_config"]
}
```


### 规则

GN中的规则可以使用rule()函数进行定义。规则可以指定以下属性：
- inputs：规则需要的输入文件。
- outputs：规则的输出文件。
- action：规则执行的操作。
```
rule("my_rule") {
inputs = ["input.txt"]
outputs = ["output.txt"]
action = "echo 'Hello, world!' > $out"
}
```

### 模块

模块可以是任何类型的代码，其可以包含源文件、头文件、库文件等，GN使用模块来组织代码和构建文件。
- 如：一个模块可以包含一个应用程序的所有源文件，另一个模块可以包含一个库的所有源文件
<br>

GN可使用模块来表示构建过程中的依赖关系
- 如：一个模块可以依赖另一个模块的源文件或者库文件
<br>


GN中的模块可以分为几种类型：**executable(可执行文件)、library(库文件)、test(测试文件)、group(组模块)、action(动作模块)**
- 模块可以应用在GN构建系统的任何地方，包括但不限于组织代码、表示依赖关系、执行操作等
- 以下示例创建了一个名为my_module的模块，其包含源文件my_module.c，而后创建了一个名为my_executable的可执行文件，其依赖my_module模块
```
module("my_module") {
  sources = ["my_module.c"]
}

executable("my_executable") {
  sources = ["my_executable.c"]
  deps = [":my_module"]
}
```
<br>

kernel_module() 函数用于创建内核模块。内核模块是运行在操作系统内核中的模块。
module() 函数用于创建普通模块。普通模块可以运行在操作系统内核中，也可以运行在操作系统用户空间中
```
kernel_module(name) {
  sources = []
  deps = []
  ldflags = []
  cflags = []
}

module(name) {
  sources = []
  deps = []
  ldflags = []
  cflags = []
}
```


### 基本语法

- 注释：在注释句子的行首加上`#`即可

- 条件语句

- 循环语句

## 应用实践范例

在构建项目时，GN会首先解析GN文件，并将工程的根目录作为起点，开始构建项目
- 目录路径中`//xxx/xxx/xxx`中双斜杠表示工程源码的根

```
  rule my_rule {
  # 定义规则的输入
  input: "input.cpp"
  # 定义规则的输出
  output: "output.o"

  # 定义规则的执行逻辑
  action {
      command: "gcc -c input.cpp -o output.o"
  }
  }
```


## GN进阶操作


### 自定义模板

- 模板是用于重复使用代码的工具。模板可以用于创建目标、配置、或其他 GN 对象
- 自定义模板使用`template()`语句
  ```
  template(name) {
    options = []
    output = ""
  }
  ```
- template() 函数的输出是模板的配置
<br>

代码应用示例如下：
- 名为 my_executable 的可执行文件，其使用 my_template 模板，包含其模板的所有配置（源文件、依赖）
```
template("my_template") {
  sources = ["my_template.c"]
  deps = [":my_module"]
}

executable("my_executable") {
  sources = ["my_executable.c"]
  template = ":my_template"
}
```

### group元目标

group不是目标，目标是指可以生成可执行文件、库文件或其它产物的构建单元。<br>
group是gn工具提供的一种元目标，其可以用于组织其它目标，但它本身并不生成任何产物。<br>


**其用途包括：**
- 组织具有相同功能或目的的目标：例如，可以将所有驱动程序目标组织到一个group中，将所有库文件目标组织到另一个group中
- 定义依赖关系：例如，可以将一个group定义为另一个group的依赖，这样，如果第一个group中的任何一个目标被编译，则第二个group中的所有目标也会被编译
- 提供文档：group可以包含文档，这些文档将在gn工具生成的输出中显示。

**一个应用示例如下：**
```
group("drivers") {
  deps = [
    "//drivers/usb/host",
    "//drivers/ethernet",
    "//drivers/storage",
  ]
}

group("libraries") {
  deps = [
    "//libraries/lib1",
    "//libraries/lib2",
    "//libraries/lib3",
  ]
}
```
- 以上代码定义了两个group，drivers目标依赖于//drivers/usb/host、//drivers/ethernet和//drivers/storage三个目标。libraries目标依赖于//libraries/lib1、//libraries/lib2和//libraries/lib3三个目标
- 在gn工具生成的输出中，drivers和libraries两个group将显示为一个单独的目标

## 补充说明以及疑惑解答

### config("public")的应用

在gn文件中，如果多个文件都定义了config("public")的include_dirs属性，那么它们的这个属性值将会叠加，而不是互相覆盖
<br>

config("public")是一个用来定义公共配置的语法。公共配置可以被其他模块引用，以便共享相同的设置
- 当然，config("public")并不表示其一定是整个工程公开的，其需要配置，可以通过configs属性来引用
- 如果一个模块的configs属性中包含了config("public")，则其会继承config("public")的所有设置，并且该模块的设置将会公开可见
- 通常做法为：在GN文件中包含统一的头文件，而后在此头文件中包含config("public")属性

### GN文件中如何包含工程源码的头文件路径？

可以使用`include_dirs`属性指定C头文件的路径，此属性是一个列表，代码示例如下：
```
target("my_executable") {
type = "executable"
sources = ["main.c"]
include_dirs = ["/usr/include/my_headers"]
}
```

也可使用`public_configs`属性，此属性列表包含了`include_dirs`属性的配置
- 以下代码示例，指示构建系统在`//configs:my_config`配置中搜索`include_dirs`配置
  ```
  target("my_executable") {
  type = "executable"
  sources = ["main.c"]
  public_configs = ["//configs:my_config"]
  }
  ``` 
- `include_dirs`属性应用示例如下：
  ```
  // 指定一个目录：
  include_dirs = ["/usr/include/my_headers"]
  // 指定多个目录：
  include_dirs = ["/usr/include/my_headers", "/usr/include/other_headers"]
  // 指定相对路径：
  include_dirs = ["../include"]
  // 指定通配符：
  include_dirs = ["**/*.h"]
  ```

