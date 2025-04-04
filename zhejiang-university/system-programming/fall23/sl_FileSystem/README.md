# FileSystem

<<<<<<< HEAD
利用 Rust 实现一个小型的文件系统
https://github.com/20001216sl/FileSystem
=======
利用 Rust 实现一个小型的文件系统https://github.com/20001216sl/FileSystem
>>>>>>> e396b926d0db3f22657a6bb2815faf07aa028cd5

# 设计内容

实现一个模拟文件系统，可为该文件系统设计相应的数据结构来管理目录、磁盘空闲空间、已分配空间等。提供文件的创建、删除、移位、改名等功能。提供良好的界面，可以显示磁盘文件系统的状态和空间的使用情况。提供虚拟磁盘转储功能，可将信息存入磁盘，还可从磁盘读入内存。

# 设计准备

## 原理和概念

文件系统，指OS中管理文件的软件。它负责管理文件的存储、检索、更新，提供安全可靠的共享和保护手段，并为用户提供一整套方便有效的文件使用和操作方法。它在OS接口中占比例最大，是I/O系统的上层软件。

要实现一个简单的文件系统，首先就了解操作系统中文件系统所需要使用到相关知识。需要了解的概念：文件目录/目录文件、文件控制块（FCB）/目录项、磁盘簇、文件分配表（FAT）。

在此，选取单用户多级目录进行实现，并视所有文件为流式文件，建立类似于FAT32文件系统中的文件分配表（FAT）进行 文件连接-物理簇 的索引。


<<<<<<< HEAD
=======

>>>>>>> e396b926d0db3f22657a6bb2815faf07aa028cd5
# 设计过程

## 基本数据结构的构建

首先，需要建立数据模型，才能对磁盘和磁盘中的对象进行具体的描述和操作。宏观上的逻辑结构说明可以参照第五章内容。

### 目录数据结构

从最根本的FCB结构开始构建。为了简单，仅仅在FCB中记录了很少的信息：文件名、文件类型（文件或目录）、FCB指向的起始簇号、文件长度。
```rust
enum FileType {
    File,
    Directory,
}

struct Fcb {
    name: String,         // 文件名
    file_type: FileType,  // 文件类型
    first_cluster: usize, // 起始块号
    length: usize,        // 文件大小
}
```

有了目录项FCB，可以开始构建目录文件了，所有的FCB条目都在目录文件中进行储存。目录结构更加简洁，只记录了文件名和一个动态的FCB项列表，可以进行对于FCB项的增加和删除。

```rust
struct Directory {
    name: String,
    files: Vec<Fcb>,
}
```

新建目录时，目录文件的文件列表会被自动加入两个特殊文件：".." 和 "." ，分别是该目录父目录的FCB和该目录自己的FCB。根目录的父目录指向自己。这样不但方便定位，而且目录文件自己就包含了自身和父级的FCB信息，对如同修改目录文件自身信息的操作是至关重要的。

### FAT表数据结构

FAT表即文件分配表(File Allocation Table)，其中每一个表项都对应了相应簇的使用情况。一个簇可能有四种情况：未被使用、指向下一个簇、坏簇和文件结束。就是这种自索引的表结构，让一个文件能够从最开始的簇号通过FAT表一直查找到文件的末尾，而不必在硬盘上物理的相邻，极大的提高了空间利用的效率。

```rust
enum FatItem {
    NotUsed,          // 未使用的簇
    ClusterNo(usize), // 指向下一个的簇号
    BadCluster,       // 坏簇
    EoF,              // 文件结束
}
```

这是一个通过Rust语言中的枚举结构表达的FAT表的表项。

### 虚拟硬盘数据结构

现在来创建一个虚拟磁盘的数据结构。FAT表在磁盘内已经预先分配了固定大小的空间，因此在软件实现上可以从储存数据的区域中独立出来，假设它不占用簇空间，仅仅在保存的时候才连带着整个磁盘一起写入虚拟磁盘文件。

```rust
struct Disk {
    fat: Vec<FatItem>,
    data: Vec<u8>,
}

impl Disk {
  fn new() -> Disk {
    Disk {
      // 创建FAT文件分配表
      fat: vec![FatItem::NotUsed; BLOCK_COUNT],

      // 数据区，初始值为0，块大小为1024.
      // 每一个块都有一个对应的FAT项
      // 所以真实的数据区域需要在总数中减去FAT项的数据大小保持硬盘大小
      data: vec![
        0u8;
        (BLOCK_COUNT - size_of::<FatItem>() * BLOCK_COUNT / BLOCK_SIZE - 1)
         * BLOCK_SIZE
      ],
    }
  }
}
```

Disk数据结构存在一个FAT项列表（`Vec<FatItem>`）和字节列表（`Vec<u8>`）。在新建该数据结构的实例时，首先初始化FAT表的每一项都是未使用状态，然后在数据区内减去FAT表所占用的磁盘空间。

### 单例模式

使用单例模拟单目录，并在里面实现各种文件系统操作所需要的各种功能函数。里面简单的包含了Disk虚拟硬盘对象和一个当前目录的对象，用于保存当前用户的操作状态。

```rust
struct DiskManager {
    disk: Disk,
    cur_dir: Directory,
}
```

## 磁盘簇操作

对上层建筑——文件和目录文件等对象的操作，最终还是要归化到最基础的簇操作上。本程序中使用簇号乘簇大小作为字节列表的索引来模拟簇。因为代码长到不适合打印，所以这里只介绍原理。

### 簇状态

簇是硬盘读写的基本单位。簇状态由FAT表进行标志，和簇的实际状态无关。例如，若FAT表中标识一个簇是未使用状态，那么不论这个簇中是否有实际数据，都被文件系统认为是空簇。这是该文件系统逻辑的基础。

### 写簇

要将一段完整的数据写入到硬盘簇中，首先需要处理FAT表。首先计算根据数据大小计算需要占用的簇数，然后在FAT表中查找到足够的未使用的簇。将簇标记改为到下一个簇的索引，最后一簇组改写为簇结束标记函数，这样一个文件的空间就在FAT表中被分配出来了。

下一个操作是把数据真正的写入到磁盘中。把一个整段的字节序列data分成以常量BLOCK_SIZE（值是1024，代表簇大小：一个簇所能储存的字节数）为大小的数据段，按照簇指针形成的链的顺序对每个簇写入数据，直到最后一个簇。若数据大小并非簇大小的整倍数，将在文件最后加上一个自定义的EoF标志（在程序中为常量EOF_BYTE，值是255），然后用0填充到簇大小的整数倍。

### 读簇

从FCB块中的首簇号开始读取，一直到簇的结束标记。截断最后一个簇中EOF标记和其以后的数据，否则会读入多余的数据。将每个簇的数据相加，就能得到一个完整的文件数据。

## 文件操作

### 文件

创建文件，如同3.2.2中所说的那样，先给文件的数据分配簇，然后写入磁盘。不过之后，还要创建一个新的包含指向该文件的首簇号数据的FCB，并把这个FCB添加到当前的目录文件的FCB列表中。

删除文件，只要找到这个文件所占用的所有簇的簇号，然后在FAT表中标记该簇为空，之后在目录中删除这个文件的FCB条目即可。没有特殊需求的话，不必真正的清空簇数据，否则会极大的降低性能。

重命名文件，改变那个文件的FCB中的文件名即可。

### 目录

目录本质上也是文件，区别只有它保存的是FCB的集合。一个目录文件也需要被一个FCB指向才能被成功的被检测到。对于目录的操作都可以归化到上面对文件的操作上，只不过操作的不是文件的FCB了而已。

值得注意的是，当目录中有文件时，直接删除目录文件会让目录中文件所占用的簇永远无法被文件系统回收，导致严重的"外存泄露"，所以不能直接删除非空的目录文件。


# 结果和分析
```Powershell
[INFO] Do you want to try to load file-sys.vd? [Y/N] n
[INFO] Will not load vd file from disk.

[INFO] Creating new disk...

==================================================
            SHENLANG's Basic File System
==================================================

Help:
    cd <dirname>: Change current dir.
    mkdir <dir name>: Create a new dir.
    ls : List all files and dir in current dir.
    cat <filename>: Show the file content.
    rm <filename>: Delete a file on disk.
    diskinfo : Show some info about disk.
    save : Save this virtual disk to file 'file-sys.vd'.
    exit : Exit the system.

Testing:
    test create: Create a random file to test.

System Inner Function:
    fn create_file_with_data(&amp;mut self, name: &amp;str, data: &amp;[u8])
    fn rename_file(&amp;mut self, old: &amp;str, new: &amp;str)
    fn delete_file_by_name(&amp;mut self, name: &amp;str)
    fn read_file_by_name(&amp;self, name: &amp;str) -> Vec<u8>

> ls

Directroy 'root' Files:
..      Directory   Length: 0
.       Directory   Length: 0

>
```
各组件经测试，在不输入错误的情况下运行良好，功能正常。


<<<<<<< HEAD
=======

>>>>>>> e396b926d0db3f22657a6bb2815faf07aa028cd5
# 使用说明

本程序使用命令行交互界面。目前可以输入以下命令：

- `cd <dirname>`: 更改当前目录。
- `mkdir <dir name>`: 新建目录。
- `ls` : 列出当前目录下所有文件和目录。
- `cat <filename>`: 显示文件内容。
- `rm <filename>`: 删除文件。
- `diskinfo` : 显示磁盘统计信息。
- `save` : 保存当前虚拟磁盘到文件 &#39;file-sys.vd&#39;。
- `exit` : 退出系统。
