# Ghidra指南

### 一、Ghidra简介

- Ghidra是由NSA创建和维护的开源逆向工具，支持Windows/Linux/macOS等不同平台，支持x84、ARM、RISC-V等不同架构下ELF文件的汇编与反汇编。
- Ghidra所在的GitHub链接：[Ghidra Software Reverse Engineering Framework](https://github.com/NationalSecurityAgency/ghidra)

### 二、安装

1. **安装Open JDK LTS**：[open jdk 21](https://adoptium.net/zh-CN/temurin/releases)

2. **将Open JDK添加到环境变量**：由于Open JDK的Jlink会和SEGGER的JLink冲突，因此Open JDK的优先级要高于SEGGER/JLink。

![environment_var](.image\environment_var.png)

![jlink_version](.image\jlink_version.png)

3. **安装Ghidra**：在GitHub仓库中下载最新的Release。

![ghidra_release](.image\ghidra_release.png)

### 三、运行Ghidra

- Ghidra直接通过压缩包即可使用，不需要修改各种系统配置，便于删除，但是不能设置快捷启动方式，需要运行`ghidraRun.bat`文件启动Ghidra。

![ghidra_run](.image\ghidra_run.png)

### 四、工程创建

- Ghidra在逆向ELF文件之前需要创建工程，一个工程中可以导入多个ELF文件。

![ghidra_create_pro](.image\ghidra_create_pro.png)

![ghidra_import_file](.image\ghidra_import_file.png)

![ghidra_open_elf](.image\ghidra_open_elf.png)

### 五、工程解析

- Ghidra打开ELF文件后，首先需要对ELF文件进行解析。

![ghidra_analyze](.image\ghidra_analyze.png)

- 解析选项如下表：

| 解析选项                             | 解释                                                         |
| ------------------------------------ | ------------------------------------------------------------ |
| Aggressive Instruction Finder        | 在尚未反汇编的未定义字节中查找有效代码，建议不勾选该选项。   |
| Apply Data Archives                  | 根据程序信息应用已知的数据类型存档，默认勾选。               |
| ARM Aggressive Instruction Finder    | 积极尝试对ARM/Thumb混合代码进行反汇编。                      |
| ARM Constant Reference Analyzer      | ARM常量传播分析器，用于分析通过多条指令计算得到的常量引用，默认勾选。 |
| ARM Symbol                           | 分析字节以识别Thumb符号，并根据需要进行-1位移调整，默认勾选。 |
| ASCII Strings                        | 搜索有效的ASCII字符串，并在二进制文件中自动创建它们，默认勾选。 |
| Call Convention ID                   | 使用反编译器来识别未知的调用约定，默认勾选。                 |
| Call-Fixup Installer                 | 调用修正，编译器在生成代码时，为了处理某些特殊函数插入的调用修正逻辑，默认勾选。 |
| Condense Filler Bytes                | 查找函数之间的填充字节并将其合并。                           |
| Create Address Tables                | 分析未定义的数据以识别地址表，默认勾选。                     |
| Data Reference                       | 分析被数据引用的数据，默认勾选。                             |
| Decompiler Parameter ID              | 反编译器为函数创建参数和局部变量，该选项会消耗大量时间，如果需要可以勾选！！！ |
| Decompiler Switch Analysis           | 使用反编译器为动态指令创建switch语句，默认勾选。             |
| Demandler GNU                        | 函数创建后，对函数名进行解混淆，并为参数设置数据类型，默认勾选。 |
| Disassemble Enter Pointer            | 对入口点函数进行反汇编，默认勾选。                           |
| DWARF                                | 自动从ELF文件中提取DWARF信息，将外部调试文件中的符号复制到程序中，默认勾选。 |
| ELF Scalar Operand References        | 存在动态加载库文件时建议勾选该选项，用于消除错误引用。       |
| Embedded Media                       | 查找嵌入的媒体数据类型，例如PNG、GIF、JPEG、WAV等，默认勾选。 |
| External Enter References            | 为已存在指令的外部入口点创建函数定义，默认勾选。             |
| External Symbol Resolver             | 将未解析的外部符号链接到程序所需库列表中找到的第一个符号，默认勾选。 |
| Function Start Pre Search            | 在反汇编任何代码之前，搜索特定于体系结构/编译器的模式，例如已知的ARM函数模式，这些函数用于处理跳转表且不返回，默认勾选。 |
| Function Start Search                | 搜索特定于体系结构的字节模式：通常是函数的起始部分，默认勾选。 |
| Function Start Search After Code     | 默认勾选。                                                   |
| Function Start Search After Data     | 默认勾选。                                                   |
| GCC Exception Handlers               | 定位并注释由GCC编译器安装的异常处理基础结构，默认勾选。      |
| Non-Returning Functions - Discovered | 在反汇编代码的过程中，检测到函数不会返回的迹象。当证据超过一定阈值时，这些函数会被标记为“无返回函数”，默认勾选。 |
| Non-Returning Functions - Known      | 通过名称定位已知的一般不会返回的函数（如 exit、abort 等），并设置“无返回”（No Return）标志，默认勾选。 |
| Reference                            | 分析指令所引用的数据，默认勾选。                             |
| Shared Return Calls                  | 当分支目标是函数时，将分支转换为调用（call），并紧接一个立即返回（return），默认勾选。 |
| Stack                                | 为函数创建栈变量，默认勾选。                                 |
| Subroutine References                | 为被调用的代码创建函数定义，默认勾选。                       |
| Variadic Function Signature Override | 检测可变参数函数调用，并解析它们的格式字符串参数以推断正确的函数签名。目前，此分析器仅支持 printf、scanf 及其变体（如 snprintf、fscanf）。 |

### 六、ELF解析

#### 6.1 ELF头

- Ghidra导航栏中的Help下拉可以查看当前ELF文件的ELF头信息。

![ghidra_help_elfheader](.image\ghidra_help_elfheader.png)

![ghidra_help_elf_header_msg](.image\ghidra_help_elf_header_msg.png)

1. **Language ID**：描述了当前ELF文件的语言规则。ARM:LE:32:v8，表示基于ARMv8内核的32位小端序。
2. **Bytes**：ELF文件字节数。
3. **Memory Blocks**：对应ELF文件中的节数量(ARM ELF文件表示ER + Section)。
4. **Instructions**：ELF文件包含的指令数。
5. **Functions**：ELF文件包含的函数数量。
6. **Data Types**：ELF文件中定义的数据类型数量。
7. **Data Type Categories**：ELF文件中数据类型的命名空间数量。
8. **ELF Original Image Base**：内存加载基地址。

#### 6.2 ELF符号表

- Ghidra导航栏中的Window下拉Symbol Table可以查看符号表。
- 在符号表窗口可以查看符号之间的引用关系，右上角Display Symbol References。

![ghidra_symbol_ref](.image\ghidra_symbol_ref.png)

- Ghidra导航栏中的Window下拉Functions可以查看函数列表。

![ghidra_functions](.image\ghidra_functions.png)

- Functions窗口支持升序、降序排列函数大小。

![ghidra_function_sj](.image\ghidra_function_sj.png)

#### 6.3 ELF内存区域

- ARM Complier引入了**Region**的概念，是链接器对程序内存的一种抽象，分为**Load Region**(LR)和**Executable Region**(ER)，其中**Region**具体的定义在scatter链接文件(.scf/.sct)中。

- ARM ELF文件中并不像ELF文件一样强调Section和Segment的概念，而是重点关注LR和ER。

- Ghidra导航栏中的Window下拉Memory Map可以查看内存区域。

![ghidra_memory_map](.image\ghidra_memory_map.png)

- ER部分对应的Start和End分别是加载到内存中的起始地址和结束地址，**R**表示可读、**W**表示可写、**X**表示可执行。

- ER里面包含众多Section，上面没有框起来的部分是debug相关的Section，不需要加载到内存中，因此也没有对应的内存起始地址和内存结束地址。

### 七、常规操作

#### 7.1 反汇编

- Ghidra支持以函数为单位将汇编指令进行反汇编，并且能够最大程度还原高级语言程序。

![ghidra_disass_bar](.image\ghidra_disass_bar.png)

![ghidra_disass](.image\ghidra_disass.png)

#### 7.2 十六进制转换

- Ghidra支持以16进制的方式显示，并且能够和汇编指令相对应。

![ghidra_hex_bar](.image\ghidra_hex_bar.png)

![ghidra_hex_display](.image\ghidra_hex_display.png)

#### 7.3 导航

- Ghidra支持快速定位，只需要提供符号名、指令地址等，Ghidra导航栏中的Navigation下拉Go to支持快速定位。

![ghidra_nav](.image\ghidra_nav.png)

![ghidra_nav_main](.image\ghidra_nav_main.png)

#### 7.4 函数调用关系

- Ghidra支持查看函数的调用关系，包含调用者和被调用者，支持图形方式显示和文字方式显示两种类型。
- 定位到相关函数，Ghidra导航栏中的Window下拉Function Call Graph可以查看图形调用关系。

![ghdira_function_use_graph](.image\ghdira_function_use_graph.png)

- 定位到相关函数，Ghidra导航栏中的Window下拉Function Call Trees可以查看文字调用关系。

![ghidra_function_use_tree](.image\ghidra_function_use_tree.png)
