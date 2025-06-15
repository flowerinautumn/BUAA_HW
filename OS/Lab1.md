### Excercise 1.1

**函数2：`readelf`**

```c
int readelf(const void *binary, size_t size) {
    // 将二进制数据强制转换为ELF文件头结构体指针
    Elf32_Ehdr *ehdr = (Elf32_Ehdr *)binary;

    // 检查是否是合法的ELF文件
    if (!is_elf_format(binary, size)) {
        fputs("not an elf file\n", stderr); // 输出错误信息
        return -1;
    }

    // 获取节头表（Section Header Table）的信息
    const void *sh_table;          // 节头表起始地址
    Elf32_Half sh_entry_count;     // 节头数量
    Elf32_Half sh_entry_size;      // 每个节头的大小（字节）

    // 从ELF头中提取信息：
    sh_table = binary + ehdr->e_shoff;      // 节头表偏移 = 文件起始地址 + e_shoff
    sh_entry_count = ehdr->e_shnum;         // 节头数量
    sh_entry_size = ehdr->e_shentsize;      // 单个节头大小

    // 遍历所有节头并输出地址
    for (int i = 0; i < sh_entry_count; i++) {
        const Elf32_Shdr *shdr;   // 当前节头的指针
        unsigned int addr;        // 节在内存中的虚拟地址（sh_addr）

        // 计算当前节头的位置：节头表起始地址 + 索引 * 单个节头大小
        shdr = (const Elf32_Shdr *)((const char *)sh_table + i * sh_entry_size);
        addr = shdr->sh_addr;    // 获取节的内存地址

        // 输出节索引和地址（如 "0:0x8048000"）
        printf("%d:0x%x\n", i, addr);
    }

    return 0;
}
```

- *在C语言中，将指针与整数相加时，实际上是将该整数乘以指针所指向类型的大小。*

- 实际代码中可能使用了GCC等编译器，它们允许void指针的算术运算，将其视为char*类型，即每次加减1移动一个字节。这种情况下，sh_table + i \* sh_entry_size会被当作(char*)sh_table + i * sh_entry_size，从而正确计算偏移量。

- 指针运算的底层逻辑：C 语言中，指针的算术运算是 **基于指针类型大小** 的。例如：

  - `int *p = ...; p + i`：实际偏移为 `i * sizeof(int)` 字节。
  - `char *p = ...; p + i`：实际偏移为 `i * 1` 字节（因为 `sizeof(char) = 1`）。

  **但 `void \*` 指针的算术运算在标准 C 中是未定义的**，因此必须显式转换为具体类型的指针（如 `char *`）才能进行字节级偏移计算。