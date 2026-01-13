# 02 - EDK II和内存映射

## 一、核心目标
1. 掌握**EDK II（UEFI开发工具包）** 的使用，重构第1章的Hello World程序，理解UEFI应用的标准化开发流程。
2. 理解**内存映射**的概念与作用，通过EDK II提供的引导服务获取计算机内存映射，并将其以CSV格式保存到文件中。
3. 夯实C语言指针基础（含结构体指针、指针的指针），为后续操作系统内存管理开发铺垫。

## 二、关键技术基础
### 1. EDK II简介与核心构成
#### （1）EDK II的定位
- 全称：EFI Development Kit II，Intel主导的开源UEFI开发工具包，是制作UEFI应用/引导加载器的标准化工具，兼容各类UEFI BIOS。
- 核心用途：提供UEFI相关的头文件、库函数、编译工具链，简化UEFI应用开发（无需手动编写二进制指令）。

#### （2）EDK II目录结构（核心文件/文件夹）
| 路径/文件                | 类型         | 核心作用                                  |
|--------------------------|--------------|-------------------------------------------|
| `edk2/edksetup.sh`       | 脚本文件     | 初始化EDK II编译环境，生成配置文件        |
| `edk2/Conf/`             | 配置文件夹   | 含`target.txt`（构建目标配置）、`tools_def.txt`（工具链配置） |
| `edk2/MdePkg/`           | 核心包       | 提供UEFI基础头文件（如`Uefi.h`）和库函数  |
| `edk2/AppPkg/`           | 应用包       | UEFI应用示例，可作为开发参考              |
| `edk2/OvmfPkg/`          | 固件包       | 开源UEFI BIOS实现（OVMF），用于模拟器测试  |
| 自定义包（如MikanLoaderPkg） | 开发包       | 存放自制UEFI应用的源码、配置文件          |

#### （3）自定义包（MikanLoaderPkg）的核心文件
| 文件名               | 类型         | 核心作用                                  |
|----------------------|--------------|-------------------------------------------|
| `MikanLoader.dec`    | 包声明文件   | 定义包的版本、依赖等元信息                |
| `MikanLoader.dsc`    | 包描述文件   | 指定编译目标、工具链、包含的组件          |
| `Loader.inf`         | 组件定义文件 | 声明UEFI应用的入口点（`ENTRY_POINT`）、依赖库 |
| `Main.c`             | 源码文件     | 应用核心逻辑（如Hello World、内存映射获取） |

### 2. 主存储器与内存映射核心概念
#### （1）主存储器的软硬件视角
- 硬件视角：由多块DDR-SDRAM内存芯片组成，物理上是独立的硬件模块。
- 软件视角：被CPU抽象为**连续的字节数组**，每个字节对应唯一的物理地址（如0x00000000、0x00000001），CPU通过地址读写数据（1字节=8比特）。

#### （2）内存映射的定义与作用
- 定义：描述主内存“哪个区域用于何种用途”的一维“地图”，核心字段包括物理起始地址、区域类型、大小（以页面为单位）。
- 作用：操作系统启动前需通过内存映射识别**空闲内存区域**（用于加载系统内核/应用）、已占用区域（如UEFI服务代码/数据），避免内存冲突。

#### （3）内存映射关键字段与类型说明
| 核心字段          | 含义                                  | 示例值       |
|-------------------|---------------------------------------|--------------|
| `PhysicalStart`   | 内存区域的物理起始地址                | 0x00001000   |
| `Type`            | 内存区域类型（用数字标识）            | 0x2（空闲区域） |
| `NumberOfPages`   | 区域大小（以4KiB为1页）               | 0x9F（约253KB） |

| `Type`值 | 类型名称               | 含义                                  |
|----------|------------------------|---------------------------------------|
| 1        | `EfiLoaderCode`        | UEFI应用的执行代码区域                |
| 2        | `EfiLoaderData`        | UEFI应用使用的数据区域                |
| 3        | `EfiBootServicesCode`  | UEFI引导服务驱动程序的执行代码        |
| 4        | `EfiBootServicesData`  | UEFI引导服务驱动程序的数据区域        |
| 7        | `EfiConventionalMemory`| 空闲内存区域（操作系统可使用）        |

- 注意：内存区域可能存在“孔洞”（非连续），需通过`NumberOfPages×4KiB`计算实际大小，不可直接通过起始地址累加推导下一个区域地址。

### 3. 指针基础（UEFI开发必备）
#### （1）核心概念
- 指针：存储变量内存地址的变量，本质是“带类型信息的地址编号”（如`int*`指向整数变量，`struct*`指向结构体变量）。
- 关键操作：
  - `&变量`：取变量的内存地址（赋值给指针）。
  - `*指针`：解引用（通过指针访问指向的变量内容）。

#### （2）结构体指针与箭头运算符
- 场景：UEFI开发中大量使用结构体（如`EFI_SYSTEM_TABLE`、`MemoryMap`），需通过指针访问成员。
- 语法对比（以`MemoryMap`结构体为例）：
  ```c
  struct MemoryMap m;
  struct MemoryMap* pm = &m;
  pm->buffer_size = 4096;  // 箭头运算符（推荐，简洁）
  (*pm).buffer_size = 4096; // 解引用+点运算符（等价，繁琐）
  ```
- 核心逻辑：指针变量仅存储地址，无论指向何种类型（int/结构体），指针本身占用的内存大小固定（x86-64架构下通常为8字节）。

#### （3）指针的指针（UEFI编程高频场景）
- 定义：指向指针变量的指针（如`EFI_FILE_PROTOCOL** ptr_ptr`），用于函数中返回指针类型的结果。
- 典型用途：UEFI函数需返回“操作状态（EFI_STATUS）”和“指针结果”，因C语言函数仅能返回一个值，故通过指针的指针传递结果：
  ```c
  EFI_FILE_PROTOCOL* memmap_file;
  EFI_FILE_PROTOCOL** ptr_ptr = &memmap_file;
  // 通过ptr_ptr改写memmap_file的值，返回文件指针
  root_dir->Open(root_dir, ptr_ptr, L"\\memmap", ...);
  ```

## 三、基于EDK II的实践操作
### 实践1：用EDK II重构Hello World程序
#### （1）核心源码（Main.c）
```c
#include <Uefi.h>          // EDK II核心头文件（含EFI_STATUS等类型）
#include <Library/Uefilib.h> // 含Print函数声明

// 入口点（由Loader.inf的ENTRY_POINT指定）
EFI_STATUS EFIAPI UefiMain(EFI_HANDLE image_handle, EFI_SYSTEM_TABLE *system_table) {
    Print(L"Hello, Mikan World!\n"); // EDK II的打印函数（类似printf，支持UCS-2编码）
    while(1); // 防止程序退出
    return EFI_SUCCESS; // EDK II标准返回值（成功）
}
```
- 关键说明：
  - `Uefi.h`：包含UEFI基础类型（`EFI_STATUS`、`EFI_SYSTEM_TABLE`），通过`#include <Uefi/UefiBaseType.h>`间接定义`EFI_STATUS`为无符号整数。
  - `Print()`：EDK II提供的格式化打印函数，支持`%d`、`%s`等占位符，字符串前缀`L`表示UCS-2编码（UEFI强制要求）。
  - `include guard`：头文件中`#ifndef __PI_UEFI_H_`、`#define __PI_UEFI_H_`、`#endif`，用于防止头文件被重复包含导致编译错误。

#### （2）构建配置与步骤
1. 切换源码版本：
   ```bash
   cd $HOME/workspace/mikanos
   git checkout osbook_day02a # 切换到day02a版本（EDK II Hello World）
   ```
2. 关联自定义包到EDK II：
   ```bash
   cd $HOME/edk2
   ln -s $HOME/workspace/mikanos/MikanLoaderPkg ./ # 创建符号链接，让EDK II识别自定义包
   ```
3. 初始化编译环境：
   ```bash
   source edksetup.sh # 生成Conf/target.txt配置文件
   ```
4. 配置`Conf/target.txt`（关键设置）：
   | 设置项           | 设置值                          | 含义                                  |
   |------------------|---------------------------------|---------------------------------------|
   | `ACTIVE_PLATFORM` | MikanLoaderPkg/MikanLoaderPkg.dsc | 指定要构建的自定义包                  |
   | `TARGET`         | DEBUG                           | 构建模式（DEBUG/RELEASE）             |
   | `TARGET_ARCH`    | X64                             | 目标架构（64位）                      |
   | `TOOL_CHAIN_TAG` | CLANG38                         | 工具链（使用Clang）                   |
5. 编译生成UEFI应用：
   ```bash
   build # 编译结果输出到$HOME/edk2/Build/MikanLoaderX64/DEBUG_CLANG38/X64/Loader.efi
   ```
6. 运行测试：
   - 复制`Loader.efi`到U盘`/EFI/BOOT/BOOTX64.EFI`，真机启动（禁用Secure Boot）。
   - 或使用QEMU：`$HOME/osbook/devenv/run_qemu.sh Build/MikanLoaderX64/DEBUG_CLANG38/X64/Loader.efi`，屏幕显示“Hello, Mikan World!”。

### 实践2：获取并保存内存映射
#### （1）核心逻辑
1. 调用EDK II引导服务`gBS->GetMemoryMap()`获取内存映射数据。
2. 通过UEFI文件协议创建文件，将内存映射以CSV格式写入文件。

#### （2）关键代码实现
##### ① 定义内存映射结构体
```c
struct MemoryMap {
    UINTN buffer_size;        // 缓冲区大小
    VOID* buffer;             // 存储内存映射数据的缓冲区
    UINTN map_key;            // 内存映射关键字（用于后续退出引导服务）
    UINTN map_size;           // 实际内存映射数据大小
    UINTN descriptor_size;    // 每个内存描述符的大小
    UINT32 descriptor_version;// 内存描述符版本
};
```

##### ② 调用`GetMemoryMap()`获取内存映射
```c
EFI_STATUS GetMemoryMap(struct MemoryMap* map) {
    if (map->buffer == NULL) {
        return EFI_BUFFER_TOO_SMALL; // 缓冲区为空，返回错误
    }
    map->map_size = map->buffer_size;
    // 调用UEFI引导服务获取内存映射
    return gBS->GetMemoryMap(
        &map->map_size,        // 输入：缓冲区大小；输出：实际需要的大小
        (EFI_MEMORY_DESCRIPTOR*)map->buffer, // 存储内存描述符的缓冲区
        &map->map_key,         // 输出：内存映射关键字
        &map->descriptor_size, // 输出：每个描述符的大小
        &map->descriptor_version // 输出：描述符版本
    );
}
```
- `gBS`：EDK II提供的引导服务全局指针，包含内存管理、文件操作等核心功能。
- 函数参数类型：`IN`（输入参数，需提前赋值）、`OUT`（输出参数，函数改写）、`IN OUT`（既是输入也是输出）。

##### ③ 保存内存映射到CSV文件
```c
// 主函数中初始化缓冲区并调用保存逻辑
CHAR8 memmap_buf[4096*4]; // 16KB缓冲区（存储内存映射数据）
struct MemoryMap memmap = {sizeof(memmap_buf), memmap_buf, 0, 0, 0, 0};
GetMemoryMap(&memmap); // 获取内存映射

// 打开根目录，创建memmap文件
EFI_FILE_PROTOCOL* root_dir;
OpenRootDir(image_handle, &root_dir);
EFI_FILE_PROTOCOL* memmap_file;
root_dir->Open(
    root_dir,
    &memmap_file,
    L"\\memmap", // 文件名（UCS-2编码）
    EFI_FILE_MODE_READ | EFI_FILE_MODE_WRITE | EFI_FILE_MODE_CREATE, // 读写+创建模式
    0
);

// 保存内存映射到文件（CSV格式）
SaveMemoryMap(&memmap, memmap_file);
memmap_file->Close(memmap_file);
```

##### ④ CSV格式写入逻辑（SaveMemoryMap函数）
```c
EFI_STATUS SaveMemoryMap(struct MemoryMap* map, EFI_FILE_PROTOCOL* file) {
    CHAR8 buf[256];
    UINTN len;
    // 写入CSV表头
    CHAR8* header = "Index,Type,Type(name),PhysicalStart,NumberOfPages,Attribute\n";
    len = AsciiStrLen(header);
    file->Write(file, &len, header);

    // 遍历所有内存描述符，写入CSV行
    EFI_PHYSICAL_ADDRESS iter = (EFI_PHYSICAL_ADDRESS)map->buffer;
    for (int i = 0; iter < (EFI_PHYSICAL_ADDRESS)map->buffer + map->map_size; 
         iter += map->descriptor_size, i++) {
        EFI_MEMORY_DESCRIPTOR* desc = (EFI_MEMORY_DESCRIPTOR*)iter;
        // 格式化CSV行（AsciiSPrint类似sprintf）
        len = AsciiSPrint(
            buf, sizeof(buf),
            "%u,%x,%-Ls,%08Lx,%Lx,%Lx\n",
            i, desc->Type, GetMemoryTypeUnicode(desc->Type), // 类型编号+名称
            desc->PhysicalStart, // 物理起始地址
            desc->NumberOfPages, // 页面数
            desc->Attribute & 0xffffflu // 属性（仅保留低20位）
        );
        file->Write(file, &len, buf); // 写入文件
    }
    return EFI_SUCCESS;
}
```
- `EFI_MEMORY_DESCRIPTOR`：UEFI定义的内存描述符结构体，核心字段见下表：
  | 字段名称         | 类型                  | 含义                                  |
  |------------------|-----------------------|---------------------------------------|
  | `Type`           | `UINT32`              | 内存区域类型（对应表2.3中的数值）     |
  | `PhysicalStart`  | `EFI_PHYSICAL_ADDRESS`| 物理起始地址                          |
  | `VirtualStart`   | `EFI_VIRTUAL_ADDRESS` | 虚拟起始地址（暂未使用）              |
  | `NumberOfPages`  | `UINT64`              | 区域大小（4KiB/页）                   |
  | `Attribute`      | `UINT64`              | 内存属性（如读写权限）                |

#### （3）运行与验证步骤
1. 切换源码版本：
   ```bash
   cd $HOME/workspace/mikanos
   git checkout osbook_day02b # 切换到支持内存映射的版本
   ```
2. 重复实践1的构建步骤（`source edksetup.sh` → `build`）。
3. 用QEMU运行：
   ```bash
   $HOME/osbook/devenv/run_qemu.sh Build/MikanLoaderX64/DEBUG_CLANG38/X64/Loader.efi
   ```
4. 查看内存映射文件：
   ```bash
   mkdir -p mnt
   sudo mount -o loop disk.img mnt # 挂载QEMU磁盘镜像
   cat mnt/memmap # 查看CSV格式的内存映射
   sudo umount mnt
   ```
- 预期输出：CSV文件包含内存区域的索引、类型、物理地址、大小等信息。

## 四、开发工具链与关键命令汇总
| 工具/命令                | 核心用途                                  | 关键场景                          |
|--------------------------|-------------------------------------------|-----------------------------------|
| `edksetup.sh`            | 初始化EDK II编译环境                      | 构建前执行                        |
| `build`                  | 编译EDK II项目（生成UEFI应用）            | 配置`target.txt`后执行            |
| `git checkout 版本号`    | 切换源码版本（如osbook_day02a/day02b）    | 切换不同实践场景                  |
| `ln -s 源路径 目标路径`  | 创建符号链接（关联自定义包到EDK II）      | 让EDK II识别MikanLoaderPkg        |
| `qemu-system-x86_64`     | 启动QEMU模拟器测试UEFI应用                | 快速验证程序功能                  |
| `mount -o loop 镜像文件 挂载点` | 挂载磁盘镜像                              | 查看保存的内存映射文件            |

## 五、注意事项与学习建议
### 1. 常见问题排查
- 构建失败：检查`Conf/target.txt`的配置项（如`ACTIVE_PLATFORM`路径是否正确）、自定义包的文件是否完整（.inf/.dsc/.dec）。
- 内存映射获取失败：确保缓冲区大小足够（建议≥16KB），`gBS`指针未被改写（EDK II环境初始化正常）。
- 文件创建失败：确认U盘/磁盘镜像为FAT32格式（UEFI仅支持FAT文件系统），路径拼写正确（如`L"\\memmap"`中的反斜杠）。

### 2. 学习建议
- 深入理解`Uefi.h`的包含链：跟踪`EFI_STATUS`、`EFI_MEMORY_DESCRIPTOR`等类型的定义来源（如`UefiBaseType.h`、`UefiSpec.h`），夯实UEFI类型体系。
- 熟练掌握指针操作：通过改写内存映射缓冲区大小、修改CSV输出格式等练习，巩固结构体指针、指针的指针的使用（UEFI开发核心技能）。
- 验证内存映射的“孔洞”：对比CSV文件中相邻区域的`PhysicalStart + NumberOfPages×4KiB`与下一个区域的`PhysicalStart`，观察是否存在非连续情况。

### 3. 关键知识点延伸
- 内存页面大小：UEFI默认1页=4KiB（1KiB=1024字节），后续内存管理（如分页机制）需基于页面单位分配内存。
- 引导服务与运行时服务：`gBS`（引导服务）仅在UEFI启动阶段可用（如获取内存映射、文件操作），`gRT`（运行时服务）在操作系统启动后仍可用（如时间管理）。
- CSV格式设计：便于后续用Excel、Python等工具分析内存映射，实际开发中可扩展为更易解析的格式（如JSON）。