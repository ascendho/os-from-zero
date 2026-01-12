# 第 1 章：计算机工作原理和 Hello World

## 一、核心目标
不依赖现有操作系统，通过**二进制编辑器直接编写**或**C语言编译链接**的方式创建程序，在屏幕上显示“Hello, World!”，同时理解计算机底层工作原理、UEFI/BIOS启动流程及操作系统开发的基础工具链。

## 二、关键技术基础
### 1. 数据表示与编码
#### （1）进制转换（核心对应关系）
| 十进制数 | 十六进制数（前缀0x） | 二进制数（前缀0b） | 十进制数 | 十六进制数（前缀0x） | 二进制数（前缀0b） |
|----------|----------------------|--------------------|----------|----------------------|--------------------|
| 0        | 0x0                  | 0b0000             | 16       | 0x10                 | 0b10000            |
| 1        | 0x1                  | 0b0001             | 17       | 0x11                 | 0b10001            |
| 2        | 0x2                  | 0b0010             | 18       | 0x12                 | 0b10010            |
| 3        | 0x3                  | 0b0011             | 19       | 0x13                 | 0b10011            |
| 4        | 0x4                  | 0b0100             | 20       | 0x14                 | 0b10100            |
| 5        | 0x5                  | 0b0101             | 21       | 0x15                 | 0b10101            |
| 6        | 0x6                  | 0b0110             | 22       | 0x16                 | 0b10110            |
| 7        | 0x7                  | 0b0111             | 23       | 0x17                 | 0b10111            |
| 8        | 0x8                  | 0b1000             | 24       | 0x18                 | 0b11000            |
| 9        | 0x9                  | 0b1001             | 25       | 0x19                 | 0b11001            |
| 10       | 0xA                  | 0b1010             | 26       | 0x1A                 | 0b11010            |
| 11       | 0xB                  | 0b1011             | 27       | 0x1B                 | 0b11011            |
| 12       | 0xC                  | 0b1100             | 28       | 0x1C                 | 0b11100            |
| 13       | 0xD                  | 0b1101             | 29       | 0x1D                 | 0b11101            |
| 14       | 0xE                  | 0b1110             | 30       | 0x1E                 | 0b11110            |
| 15       | 0xF                  | 0b1111             | 31       | 0x1F                 | 0b11111            |
- 核心用途：计算机仅识别二进制，十六进制用于简化二进制书写（1个十六进制数对应4个二进制数）。

#### （2）字符编码
- **ASCII**：英文字母/符号与字节的映射（如'H'=0x48，'o'=0x6F），1字节表示。
- **UCS-2**：UEFI应用强制使用的编码（如BOOTX64.EFI），2字节表示1个字符，且x86-64架构采用**小端序**（低字节在前，如'H'=0x0048，存储为0x48 0x00）。

### 2. 可执行文件格式
| 格式   | 用途                                  | 关联场景                          |
|--------|---------------------------------------|-----------------------------------|
| PE     | Windows/UEFI应用的可执行文件格式      | 最终生成的hello.efi、BOOTX64.EFI |
| COFF   | 编译后的中间对象文件格式              | Clang生成的hello.o（目标文件）    |
| ELF    | Linux的可执行文件/对象文件格式        | 传统Linux程序，本章暂不使用        |
- 关键逻辑：C语言源码→编译器（Clang）生成COFF格式对象文件→链接器（lld-link）合并为PE格式UEFI应用。

## 三、实现“Hello, World!”的两种方法
### 方法1：二进制编辑器直接编写（无编程语言依赖）
#### （1）工具准备
- 编辑器：Okteta（Linux原生）或Windows Binary Editor（WSL环境）。
- 安装Okteta（Ubuntu）：`sudo apt install okteta`。

#### （2）操作步骤
1. 启动Okteta，按文档指定的十六进制序列输入数据（核心是UEFI应用的引导指令+“Hello, World!”的UCS-2编码）。
2. 保存文件为`BOOTX64.EFI`（UEFI启动的默认引导文件名）。
3. 校验文件正确性：执行`sum BOOTX64.EFI`，校验和需为`12430 2`（错误则需手动排查输入）。

### 方法2：C语言编写+编译链接（推荐，易扩展）
#### （1）C语言源码（hello.c）
```c
EFI_STATUS EfiMain(EFI_HANDLE ImageHandle, EFI_SYSTEM_TABLE *SystemTable) {
    // 调用UEFI的输出接口显示字符串（L表示UCS-2编码）
    SystemTable->ConOut->OutputString(SystemTable->ConOut, L"Hello, world!\n");
    while(1); // 防止程序退出
    return 0;
}
```
- 关键说明：
  - `EfiMain`：UEFI应用的入口函数（替代普通C程序的`main`）。
  - `SystemTable->ConOut`：UEFI提供的控制台输出接口，用于屏幕显示。
  - 入参`ImageHandle`和`SystemTable`由UEFI BIOS自动传入。

#### （2）编译+链接命令（Ubuntu/WSL环境）
1. 编译（生成COFF格式对象文件hello.o）：
   ```bash
   clang -target x86_64-pc-win32-coff -mno-red-zone -fno-stack-protector -fshort-wchar -Wall -c hello.c -o hello.o
   ```
2. 链接（生成PE格式UEFI应用hello.efi）：
   ```bash
   lld-link /subsystem:efi_application /entry:EfiMain /out:hello.efi hello.o
   ```
- 参数解释：
  - `-target x86_64-pc-win32-coff`：指定输出COFF格式（适配UEFI）。
  - `/subsystem:efi_application`：声明目标是UEFI应用。
  - `/entry:EfiMain`：指定入口函数为`EfiMain`。

## 四、程序运行方式（3种场景）
### 场景1：真机运行（物理计算机）
#### （1）操作步骤
1. 格式化U盘：使用`mkfs.fat /dev/sdb1`（`/dev/sdb1`为U盘设备名），格式化为FAT32（UEFI仅支持FAT文件系统）。
2. 创建目录结构：`sudo mkdir -p /mnt/usbmem/EFI/BOOT`。
3. 复制文件：将`BOOTX64.EFI`（或hello.efi重命名为BOOTX64.EFI）复制到`/mnt/usbmem/EFI/BOOT`。
4. 卸载U盘：`sudo umount /mnt/usbmem`。
5. 启动真机：插入U盘，开机按Delete/F2进入BIOS，**禁用Secure Boot**（否则会阻止自制UEFI应用启动），设置U盘为第一启动项。

#### （2）关键问题：如何查找U盘设备名？
- 执行`dmesg`命令，查看输出中“USB Flash Disk”对应的设备（如`/dev/sda`或`/dev/sdb`，分区为`sda1`/`sdb1`）。

### 场景2：WSL（Windows Subsystem for Linux）运行
#### （1）特殊注意事项
- WSL无法直接访问U盘设备，需先在Windows中操作，再通过WSL挂载复制。

#### （2）操作步骤
1. Windows中格式化U盘：右键U盘→格式化→文件系统选`exFAT`（兼容Windows和WSL）。
2. WSL中挂载U盘：
   ```bash
   sudo mkdir -p /mnt/usbmem
   sudo mount -t drvfs F: /mnt/usbmem  # F:为Windows中U盘的盘符
   ```
3. 复制文件：同场景1的步骤2-3。
4. 卸载：`sudo umount /mnt/usbmem`。

### 场景3：模拟器运行（QEMU，推荐新手）
#### （1）操作步骤
1. 创建FAT32磁盘镜像（200MB）：
   ```bash
   qemu-img create -f raw disk.img 200M
   mkfs.fat -n 'MIKAN OS' -s 2 -f 2 -R 32 -F 32 disk.img
   ```
2. 挂载镜像并复制文件：
   ```bash
   mkdir -p mnt
   sudo mount -o loop disk.img mnt
   sudo mkdir -p mnt/EFI/BOOT
   sudo cp BOOTX64.EFI（或hello.efi） mnt/EFI/BOOT/BOOTX64.EFI
   sudo umount mnt
   ```
3. 启动QEMU（UEFI模式）：
   ```bash
   qemu-system-x86_64 \
     -drive if=pflash,file=$HOME/osbook/devenv/OVMF_CODE.fd \
     -drive if=pflash,file=$HOME/osbook/devenv/OVMF_VARS.fd \
     -hda disk.img
   ```
- 简化命令（使用脚本）：`$HOME/osbook/devenv/run_qemu.sh BOOTX64.EFI`。
- 关键依赖：`OVMF_CODE.fd`和`OVMF_VARS.fd`是UEFI固件镜像，需提前安装。

## 五、计算机底层工作原理
### 1. CPU核心工作流程
CPU（数字电路）仅处理二进制数据，流程为：
1. 从主存储器读取指令和数据；
2. 执行指令（如计算、调用接口）；
3. 将执行结果写回主存储器；
4. 循环上述步骤。

### 2. 硬件协作逻辑
- 核心组件：CPU（计算核心）、主存储器（存储指令/数据）、芯片组（连接CPU与外部设备）、外部设备（屏幕、U盘等）。
- 数据流向：外部设备→芯片组→主存储器→CPU（执行）→主存储器→芯片组→外部设备（输出，如屏幕显示）。

## 六、UEFI/BIOS启动核心流程
### 1. BIOS与UEFI的区别
| 类型       | 特点                                  | 适用场景                          |
|------------|---------------------------------------|-----------------------------------|
| 传统BIOS   | 功能简单，兼容性旧设备                | 早期计算机                        |
| UEFI BIOS  | 统一规范、可扩展，支持现代硬件/文件系统 | 2006年后计算机，本章开发首选      |

### 2. 启动步骤（UEFI模式）
1. 开机后CPU自动执行ROM中的UEFI BIOS程序；
2. UEFI BIOS初始化硬件（CPU、内存、磁盘等）；
3. BIOS扫描存储设备（U盘/硬盘）的`/EFI/BOOT/BOOTX64.EFI`文件（UEFI应用）；
4. 将`BOOTX64.EFI`加载到主存储器；
5. CPU执行主存储器中的`BOOTX64.EFI`指令，屏幕显示“Hello, World!”。

## 七、开发工具链汇总
| 工具类型       | 具体工具                          | 核心用途                                  | 关键命令/操作                          |
|----------------|-----------------------------------|-------------------------------------------|---------------------------------------|
| 二进制编辑器   | Okteta、Windows Binary Editor     | 直接编写十六进制指令/数据，生成UEFI应用    | 输入指定十六进制序列，保存为BOOTX64.EFI |
| 编译器         | Clang                             | 将C源码编译为COFF格式对象文件              | `clang -target x86_64-pc-win32-coff ...` |
| 链接器         | lld-link                          | 将对象文件合并为PE格式UEFI应用             | `lld-link /subsystem:efi_application ...` |
| 模拟器         | QEMU                              | 虚拟硬件环境，安全测试自制程序             | `qemu-system-x86_64 -drive ...`       |
| 命令行工具     | dmesg、sum、mkfs.fat、mount       | 查找设备、校验文件、格式化、挂载存储介质   | `dmesg`（查U盘）、`sum 文件名`（校验） |

## 八、学习建议与注意事项
### 1. 高效学习方法
- “抄经”：手动输入代码/十六进制序列（而非复制粘贴），关注细节（如字符编码的字节顺序）。
- 实践优先：即使暂时不理解理论，先动手运行程序，后续回头复盘会更易理解。
- 主动拓展：修改源码（如修改显示字符串），排查问题（如校验和错误、启动失败），深化理解。

### 2. 常见问题排查
- 真机启动失败：大概率是Secure Boot未禁用，进入BIOS关闭即可。
- WSL挂载U盘失败：确认Windows中U盘盘符正确（如F:），命令为`mount -t drvfs 盘符 /mnt/路径`。
- QEMU启动无显示：需确认使用UEFI模式（加载OVMF镜像），BIOS模式无法运行UEFI应用。
- 字符显示乱码：检查字符编码是否为UCS-2，字节顺序是否为小端序。