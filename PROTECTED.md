#### 用于记录进入保护模式之后的学习过程

1. 保护模式总结

    > On the x86-64 architecture, page-level protection now completely supersedes Segmentation as the memory protection mechanism. On the IA-32 architecture, both paging and segmentation exist, but segmentation is now considered 'legacy'.
    
    所以说我一直使用的是I-32不是x86-64，查看[osdev](https://wiki.osdev.org/Bochs):

    > The default compile does not support x86-64, --enable-x86-64 will turn it on.

    所以如果配置的时候没有加这个修饰的话，应该就是32位的了，在配置文件里应该能看出来，但是没找到具体在哪。


    还是要总结一下 segment-selector 和 segment-descriptor的结构和内容说明:

    ```cpp
    typedef struct segment_selector{
        RPL_2,
        TI_1,
        Index_13
    } segment_selector;
    ```
    
    segment_selector是一个16位的选择器
      + 高13位是用来作为序号选择gdt的segment_descriptor
      + TI位 0表示GDT， 1表示LDT
      + RPL-Requested Privilege Level位用来表示优先级与segment_descriptor的DPL结合使用

    ```cpp
    typedef struct segment_descriptor
    {
        Segment_Limit_16[0 : 15],
        Base_Address_24[0 : 23],
        Type_4,
        S_1,
        DPL_2,
        P_1,
        Segment_Limit_4[16 : 19],
        AVL_1,
        L_1,
        DB_1,
        G_1,
        Base_Address_8[24 : 31]
    } segment_descriptor;
    ```
    
    > A segment descriptor is a data structure in a GDT or LDT that provides the processor with the size and location of a segment, as well as access control and status information. 

    segment_descriptor 是一个64位的描述符数据结构, GDT的Table表指的就是具有很多 segment_descriptor 组成的一个大的数据结构，而ldgt加载的也就是其中的 第0个 segment_descriptor的首地址

      + 低16位表示限制大小limit[0:15], 这里要注意的是
        > the limit value is added to the base address to get the address of the last valid byte.
        
        所以 这个 limit = 最大的字节数-1
      + 紧接着的23位是 基地值 base[0:23], 就是这个segment_descriptor的首地址
      + 4位的type 表示的范围是0-15, 其中[0,7]表示数据段， [8,15]表示代码段， 具体的区别在于访问A，读写R/W和扩展方向E， 除了访问位作为一种标记手段，其余两个bit的作用会根据代码段和数据段而有所区别
      + S 位 1表示 代码段和数据段， 为0表示系统段，暂时不理解
      + DPL-descriptor privilege level 与上面的RPL结合使用
      + P 位表示 段是否被加加载到内存中
      + D_B 位对于32位需要置1
      + G 位 用来表示limit 的粒度，0代表字节位单位，1代表4kB为单位
      + 最高8为位也是基地址base[24:31]

    > For a program to access a segment, the segment selector for the segment must have been loaded in one of the segment registers.

    最后结合上面这段话，总结一下在总体的访问逻辑，首先将一个 segment_selector 加载到段寄存器里， 也就是 CS、DS、SS、ES、FS、GS这6个当中， 然后直接 jmp X:B 应该就可以进入保护模式了，手册里也有提到，这里需要是一个长跳转。

2. paging 内存映射

    要用到很多控制寄存器的内容。见intel手册[卷3-2.5节Control Register]()。

    如果cr4_PAE = 0, 也就是进入保护模式后默认情况，给cr0_PG 位置1后进入的就是32-bits的paging模式。

    > Every paging structure is 4096 Bytes in size and comprises a number of individual entries. With 32-bit paging,each entry is 32 bits (4 bytes); there are thus 1024 entries in each structure. 

    因此每个页表2^12 = 4096个字节，每一个页表项4个字节，就有1024个页表项。

    > Contains the physical address of the base of the paging-structure hierarchy and two flags (PCD and PWT). Only the most-significant bits (less the lower 12 bits) of the base address are specified; the lower 12 bits of the address are assumed to be 0. The first paging structure must thus be aligned to a page (4-KByte) boundary. 

    > When using the physical address extension, the CR3 register contains the base address of the page-directory-pointer table.

    cr3[12:31]的20位用来表述页目录的基地址。外加上还有两个符号位PCD、PWT用来cache，暂不考虑。
    
    >  Only the most-significant bits (less the lower 12 bits) of the base address are specified; the lower 12 bits of the address are assumed to be 0. The first paging structure must thus be aligned to a page (4-KByte) boundary. 

    这里指出， cr3只是给出了页目录基地址的高20位，低12位全部是0， 因此具体在设置页目录的位置的时候，要求地址的后12位全部是0。同样的，因为页表中的有效地址也都是20位，因此页表也要同样的对齐。


    一个页是4KB=4096B， 那么总共就有 4GB / 4KB = 2^20 个页表项。如果想要统计这2^20 * 4B 也就是 4MB的空间，那么任何一个程序的都要4MB的空间显然是浪费了，进而有了页目录的出现。相当于是两级索引：

            31:22 -> Directory      在页目录中查
            21:12 -> Table          在页表中查
            11: 0 -> Offset         偏移

    进而对应的页目录PDE(Page-Directory Entry)、页表( Page-Table Entry)也有自己相应的结构，具体在[卷3-4.3 : 图4-4、表4-5 表4-6]()：
    
    ### PDE:

    + 0 (P)
        
    > Present; must be 1 to reference a page table

    + 1 (R/W)
    > Read/write; if 0, writes may not be allowed to the 4-MByte region controlled by this entry.

    + 2 (U/S)
    > User/supervisor; if 0, user-mode accesses are not allowed to the 4-MByte region controlled by this entry
     
    + 3 (PWT)
    > Page-level write-through; indirectly determines the memory type used to access the page table referenced by this entry 

    + 4 (PCD)
    > Page-level cache disable; indirectly determines the memory type used to access the page table referenced by this entry

    + 5 (A)
    > Accessed; indicates whether this entry has been used for linear-address translation 

    + 6
    > Ignored

    + 7 (PS)
    > If CR4.PSE = 1, must be 0 (otherwise, this entry maps a 4-MByte page

    + 11:8
    > Ignored

    + 31:12
    > Physical address of 4-KByte aligned page table referenced by this entry

    ### PTE 


    + 0 (P)
    > Present; must be 1 to map a 4-KByte page

    + 1 (R/W)
    > Read/write; if 0, writes may not be allowed to the 4-KByte page referenced by this entry

    + 2 (U/S)
    > User/supervisor; if 0, user-mode accesses are not allowed to the 4-KByte page referenced by this entry

    + 3 (PWT)
    > Page-level write-through; indirectly determines the memory type used to access the 4-KByte page referenced by this entry 

    + 4 (PCD)
    > Page-level cache disable; indirectly determines the memory type used to access the 4-KByte page referenced by this entry

    + 5 (A)
    > Accessed; indicates whether software has accessed the 4-KByte page referenced by this entry

    + 6 (D)
    > Dirty; indicates whether software has written to the 4-KByte page referenced by this entry

    + 7 (PAT)
    > If the PAT is supported, indirectly determines the memory type used to access the 4-KByte page referenced by this entry (see Section 4.9.2); otherwise, reserved (must be 0)1

    + 8 (G)
    > Global; if CR4.PGE = 1, determines whether the translation is global (see Section 4.10); ignored otherwise

    + 11:9
    > Ignored

    + 31:12
    > Physical address of the 4-KByte page referenced by this entry


    现在将PDE放在0x2000, PTE放在0x3000, 都满足低20位全是0的条件

    因此对于一个逻辑地址比如：0xc000_0000 到 0xc010_0000 这1MB的地址，分为10+10+12就是：

        0-10  : 0b_11_0000_0000
        11-20 : 0b_00_0000_0000
        21-32 : 0b_0000_0000_0000
    对于一个逻辑地址，分别去PDE、PTE寻找，并结合最后的12位 得到真实的物理地址。

    所以主要就是两次查找，对应的也就有两次配置，配置PDE和配置PTE：
    + 配置PDE

          mov eax, PTE        ; 把PTE的地址放入eax
          or eax, ATTR        ; 配置控制位
          mov [PDE + 0x3ff * 4], eax  
          这里是把 PTE地址的页表首地址放在了第0x3ff个页目录的页表项中，
          所以，当逻辑地址的前10位是0x3ff的时候，会访问这个页目录的页表项，并得到相应的页表首地址
    + 配置PTE

          mov esi, 0      ; 把实际物理地址放到PTE中， 这里是从0开始，每次增加2^12 = 4KB
          mov eax, esi    
          shl eax, 12
          or eax, ATTR    ; 配置控制位
          mov [PTE + esi * 4], eax     ; 把物理页的地址放到PTE开始的页表项中
          ; 这里就把 0x0000_0000 到0x0010_0000 这1MB的物理地址按顺序写进PTE开始的256个页表项内，
          ; eax每次偏移4Kb，页表项每次偏移4个字节
    
    - 总结:
      - cr3存的是1K个PDE的首地址
      - PDE的页表项中存的是1K个PTE的首地址
      - PTE的页表项中存的是一个4KB大小的物理页的首地址

    还有一个很特殊的地方就是，0x3ff 的页目录被指向了页目录自己，所以对于前20位都是1的1KB：

          0xffff_f000 - 0xffff_ffff

    指向的就是PDE页目录的物理地址, 通过 info tab 就可以查看