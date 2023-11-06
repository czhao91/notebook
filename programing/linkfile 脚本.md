# Linkfile

## 1. 介绍

当编写多个C文件，需要将它们编译链接成一个可执行文件，此时需要用到链接脚本文件 (ld)

ld 脚本的功能是将多个目标文件 (.o) 和库文件 (.a) 链接成一个可执行文件

### 1.1 基本构成

**链接配置**：包括符号变量定义，入口地址，输出格式等

```makefile
STACK_SIZE=0x200
OUTPUT_FORMAT("elf32-littlearm")
OUTPUT_ARCH(arm)
ENTRY(_start)
```

**内存布局**：以 MEMORY命名定义存储空间，其中以 ORIGIN 定义地址空间起始位置，LENGTH 定义地址空间长度

```makefile
MEMORY
{
	FLASH(rx) : ORIGIN=0x80000000, LENGTH=0x20000
}
```

**段链接定义**

```makefile
SECTIONS
{
	.text :				// 代码段
	{
		*(.text*)		// 工程中所有目标文件的.text段链接到FLASH中
	} > FLASH
}
```

## 2. 常用关键字、命令

### 2.1 MEMORY 命令

```kotlin
MEMORY {
NAME1 [(ATTR)] : ORIGIN = ORIGIN1, LENGTH = LEN2
NAME2 [(ATTR)] : ORIGIN = ORIGIN2, LENGTH = LEN2
…
}
```

- NAME : 存储区域名字
- ATTR：定义该存储区域的属性
  - R：只读
  - W：读/写
  - X：可执行
  - A：可分配
  - I：初始化
  - !：不满足该字符之后任何一个属性
- ORIGIN：区域开始地址
- LENGTH：区域大小

### 2.2 定位符号 `.`

`.` 表示当前地址，可被赋值也可以赋值给某个变量

```kotlin
SECTIONS
{
    . = 0×10000;
    .text : 				// 所有目标文件的text段从0x10000开始存放
    { 
        *(.text)
    }
    
    . = 0×8000000;
    .data : 
    { 
        *(.data) 
    }
}
```

### 2.3 SECTION 命令

- secname：输出文件
- contents：输入文件
- start ：将某个段强制链接到该地址
- region : MEMORY 命令定义的位置信息
- AT（addr）：实现存放地址和加载地址不一致的功能，AT 表示在文件中存放的位置

```kotlin
SECTIONS
{
       ...
      secname start BLOCK(align) (NOLOAD) : AT ( ldadr )
      { 
        contents 
      } >region :phdr =fill
      ...
}
```



```kotlin
SECTIONS
{
       ...
      .data :
      { 
        main.o (.data）
        *(.data)	//所有目标的 .data 链接到输出文件 .data 段
      } 
      ...
}
```

### 2.4 PROVIDE 关键字

该关键字用于定义：在目标文件内被引用，但没有在任何目标文件内被定义的符号。

```c++
SECTIONS
{
    .text :
    {
        *(.text)
        _etext = .;
        PROVIDE(_etext = .);
    }
}
```



```c++
int main()
{
    extern char _etext;  // reference
}
```



## 3. KL26 芯片链接脚本示例

```
/*
 * In this linker script there is no heap available.
 * The stack start at the end of the ram segment.
 */
STACK_SIZE = 0x2000;             /*  stack size config   8k        */

/*
 * Take a look in the "The GNU linker" manual, here you get
 * the following information about the "MEMORY":
 *
 * "The MEMORY command describes the location and size of 
 * blocks of memory in the target."
 */
MEMORY
{
   FLASH_INT     (rx)  : ORIGIN = 0x00000000, LENGTH = 0x00000100
   FLASH_CONFIG  (rx)  : ORIGIN = 0x00000400, LENGTH = 0x00000010
   FLASH_TEXT    (rx)  : ORIGIN = 0x00000410, LENGTH = 0x0001F7F0
   RAM           (rwx) : ORIGIN = 0x1FFFF000, LENGTH = 16K
}

/*
 * And the "SECTION" is used for:
 *
 * "The SECTIONS command tells the linker how to map input
 * sections into output sections, and how to place the output
 * sections in memory.
 */
SECTIONS
{
   /* The startup code goes first into internal flash */
    .interrupts :
    {
      __VECTOR_TABLE = .;
      . = ALIGN(4);
      KEEP(*(.vectors))         /* Startup code */
      . = ALIGN(4);
    } > FLASH_INT

    .flash_config :
    {
      . = ALIGN(4);
      KEEP(*(.FlashConfig))    /* Flash Configuration Field (FCF) */
      . = ALIGN(4);
    } > FLASH_CONFIG

     .text :
    {
        _stext = .;           /* Provide the name for the start of this section */
  
        *(.text)
        *(.text.*)              /*  cpp namespace function      */
        *(.romrun)              /*  rom中必须的函数             */
        
        . = ALIGN(4);           /* Align the start of the rodata part */
        *(.rodata)              /*  read-only data (constants)  */
        *(.rodata*)
        *(.glue_7)
        *(.glue_7t)
    } > FLASH_TEXT

    /* section information for simple shell symbols */
    .text :
    {
        . = ALIGN(4);
        __shellsym_tab_start = .;
        KEEP(*(.shellsymbol))
        __shellsym_tab_end = .;
    } >FLASH_TEXT

    /* .ARM.exidx is sorted, so has to go in its own output section */
    . = ALIGN(4);
     __exidx_start = .;
     PROVIDE(__exidx_start = __exidx_start);
    .ARM.exidx :
    {
        /* __exidx_start = .; */
        *(.ARM.exidx* .gnu.linkonce.armexidx.*)
        /* __exidx_end = .;   */
    } > FLASH_TEXT
    . = ALIGN(4);
     __exidx_end = .;
     PROVIDE(__exidx_end = __exidx_end);

    /* .data 段数据初始化内容放在这里 */
    . = ALIGN(16);
    _etext = . ;
    PROVIDE (etext = .);
    
   /*
    * The ".data" section is used for initialized data
    * and for functions (.fastrun) which should be copied 
    * from flash to ram. This functions will later be
    * executed from ram instead of flash.
    */
   .data : AT (_etext)
   {
      . = ALIGN(4);        /* Align the start of the section */
      _sdata = .;          /* Provide the name for the start of this section */
      
      *(.data)
      *(.data.*)
      
      . = ALIGN(4);        /* Align the start of the fastrun part */
      *(.fastrun)
      *(.fastrun.*)
      
      . = ALIGN(4);        /* Align the end of the section */
   } > 
   
   _edata = .;             /* Provide the name for the end of this section */
   
   USB_RAM_GAP = DEFINED(__usb_ram_size__) ? __usb_ram_size__ : 0x800;
   /*
    * The ".bss" section is used for uninitialized data.
    * This section will be cleared by the startup code.
    */
   .bss :
   {
      . = ALIGN(4);        /* Align the start of the section */
      _sbss = .;           /* Provide the name for the start of this section */
      
      *(.bss)
      *(.bss.*)
      . = ALIGN(512);
      USB_RAM_START = .;
    . += USB_RAM_GAP;
      
      . = ALIGN(4);        /* Align the end of the section */
   } > RAM
   _ebss = .;              /* Provide the name for the end of this section */
   
    /* 系统堆 */
    . = ALIGN(4);
    PROVIDE (__heap_start__ = .);
    .heap (NOLOAD) :
    {

    } > RAM
    . = ORIGIN(RAM) + LENGTH(RAM) - STACK_SIZE;
    . = ALIGN(4);
    PROVIDE (__heap_end__ = .);
   
   /* 
    * The ".stack" section is our stack.
    * Here this section starts at the end of the ram segment.
    */
   _estack = ORIGIN(RAM) + LENGTH(RAM);

   m_usb_bdt USB_RAM_START (NOLOAD) :
   {
     *(m_usb_bdt)
     USB_RAM_BDT_END = .;
   }

   m_usb_global USB_RAM_BDT_END (NOLOAD) :
   {
     *(m_usb_global)
   }
}

/*** EOF **/
```