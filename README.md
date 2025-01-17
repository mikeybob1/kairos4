# FAT32下的文件检索

## 一、实验介绍

当我们使用`fopen`函数打开一个文件时，文件系统是如何通过传入的文件名来进行检索定位的呢？在本次实验中，你将体验到FAT32中的文件查找的流程！

> 不可否认，FAT32是一项陈旧的文件系统，因此，我们通过提供框架代码帮助你隐藏了大部分FAT32的文件名相关细节问题，因此你大可不必担心对于FAT32文件系统规格本身不熟悉而导致的困扰。

## 二、实验目的

1. 了解FAT32文件系统的文件命名

## 三、实验要求

1. 完成实验要求函数
2. 提交代码

## 四、实验步骤

### 1. 逻辑目录遍历

在`src/fs/fat32.c`中，我们已经为你实现了一个FAT32的目录遍历框架，其接口定义：

```c
FR_t fat_travs_logical_dir(fat32_t *fat, uint32_t dir_clus, uint32_t dir_offset, travs_handler_t handler, void *state)
```

其中一个最为重要的参数为`handler`，这个接口函数将会遍历FAT32实际的目录项，屏蔽掉FAT32具体目录项的具体细节，并集中整合为一个逻辑上的目录项（如果你了解过[FAT32](https://academy.cba.mit.edu/classes/networking_communications/SD/FAT.pdf)，应当知道其目录项实际分为长/短目录），将其传递给我们的`handler`函数进行处理。`travs_handler_t`的定义如下：

```c
/**
 * @brief The handler used to traverse a directory.
 *        It will receive 4 params: 
 *          1. item    - current dir entry
 *          2. name    - current dir entry name
 *          3. offset  - current dir entry offset in directory
 *          4. __state - used to save state in this round of traversal
 *        Returning FR_OK/FR_ERR means end, or FR_CONTINUE to continue.
 */
typedef FR_t (*travs_handler_t)(dir_item_t *item, const char *name, off_t offset, void *__state);
```

此外，我们传递的`handler`还应该有自己的状态信息，这将被存储在`state`参数中，一并传递给`handler`。

### 2. 实现查找

接下来的问题就转嫁到`handler`的函数实现上，我们已经为你包裹好了一个入口函数：

```c
FR_t fat_dirlookup(fat32_t *fat, uint32_t dir_clus, const char *cname, dir_item_t *ret, uint32_t *offset) {
    DEFINE_LOOKUP_HELPER(helper, cname, 0, ret, offset);
    return fat_travs_logical_dir(fat, dir_clus, 0, lookup_handler, &helper);
}
```

`DEFINE_LOOKUP_HELPER`定义了一个名为`helper`的辅助结构体，它就是我们上节提到的`handler`所需的状态信息，其定义如下：

```c
typedef struct lookup_helper {
    /// @brief target name
    const char *name;
    /// @brief name checksum
    uint8_t checksum;
    /// @brief dir item
    dir_item_t *item;
    /// @brief dir item offset
    uint32_t *offset;
} lookup_helper_t;
```

你可以暂时忽略`checksum`字段，只需关注`name: 文件名`，`item: 目标目录项`，`offset: 目标目录项在块中的偏移量`，注意后两个参数是用于输出的，输出的意思是指当你在`hanlder`中匹配到了对应的目录项后，*你需要将相关的目录项信息赋值给这些字段中指针所指向的位置*。

接下来，`fat_dirlookup`将`helper`状态信息与我们将要去实现的`lookup_hanlder`等信息一并传递给`fat_travs_logical_dir`，用于遍历。

```c
// src/fs/fat32.c:917
static FR_t lookup_handler(dir_item_t *item, const char *name, off_t offset, void *__helper) {
    lookup_helper_t *helper = (lookup_helper_t *)__helper;
    // TODO:
    // compare `name` and `helper->name`
    // and decide if continue
}
```

这就是我们最终要去实现的函数了，它每次将会接受一个目录项`item`，以及该目录项的名字`name`，目录项偏移`offset`，还有我们传入的`helper`。

接下里你要做的便是比较(`strncmp`)文件名然后传递结果。该函数的返回值将会决定外部`fat_travs_logical_dir`的遍历行为：当返回`FR_CONTINUE`时，遍历将会继续，返回`FR_OK`或者`FR_ERR`将会终止外部的遍历，你应当根据搜索结果来决定返回值。
