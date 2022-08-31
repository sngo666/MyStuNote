## 间接寻址、JMP与LOOP

##### 间接寻址

直接寻址很少用于数组的处理，直接寻址在使用常量偏移寻址多个数组元素时很不实用，当使用寄存器的值作为指针寻址时，称为间接寻址。

&emsp;

* 间接操作数
保护模式： 任何一个32位通用寄存器加上括号就能构成一个间接操作数。寄存器中存放的是数据的地址示例如下：

>```asm
> .data
> val BYTE 20h
> .code 
> mov esi, OFFSET val
> mov al, [esi]            ;al = 20h
> ```

如果目的操作数也是间接操作数，那么新值将存入由寄存器提供的地址的内存位置。如下：
>`mov [esi], bl`  
>此时有将BL寄存器内容复制到ESI寻址的内容中去。  

当一个操作数的大小可能无法从指令中直接看出来时，这是可以用PTR运算符确定操作数的大小：
>`inc BYTE PTR [esi]`

&emsp;

* 数组

使用间接操作数遍历数组，如：

> ```asm
> .data
> array DWORD 1000h, 2000h, 3000h
> .code
> mov esi, OFFSET array
> mov eax, [esi]                 ;eax = 1000h
> add esi, 4
> mov eax, [esi]                 ;eax = 2000h
> add esi, 4
> mov eax, [esi]                 ;eax = 3000h
> ```

&emsp;

* 变址操作数
  
变址操作数指的是，在寄存器上加上常数产生一个有效地址。每个32位通用寄存器都可以用作变址寄存器，格式如下：
>
> ```asm
> constant[reg]
> [constant + reg]
> ```

对于第二种的使用在`数组`中已经稍有提及，现在举例说明第一种变址的使用：

> ```asm
> .data
> array DWORD 1000h, 2000h, 3000h
> .code
> mov esi, 0
> mov eax, array[esi]                 ;eax = 1000h
> mov eax, array[esi + 4]             ;eax = 2000h
> ```
>

将ESI和array的偏移量相加，并将响应内容复制进eax

_当我们在实地址模式中，一般使用16位寄存器作为变址操作数，那么能使用的寄存器只能有SI、DI、BX和BP。而且如果有间接操作数，尽量避免使用BP寄存器，除非是寻址堆栈数据。_

也可以通过使用比例因子，使代码更具可读性：
>
> ```asm
> .data
> array DWORD 1000h, 2000h, 3000h
> .code
> mov esi, 0
> mov eax, array[esi]                    ;eax = 1000h
> mov eax, array[esi + 2 * TYPE DWORD]   ;eax = 3000h
> ```

&emsp;

* 指针

如果一个变量包含另一个变量的地址，则该变量被称为指针。使用方法如下：

> ```asm
> .data
> array BYTE 10h, 20h, 30h
> ptr DWORD array
> ```

也可以使用OFFSET运算符定义ptr，使得关系更加明确：
>`ptr DWORD OFFSET array`

&emsp;

* 使用TYPEDEF运算符

TYPEDEF运算符可以用以创建用户定义类型。如：
>`PBYTE TYPEDEF PTR BYTE`

> ```asm
> array BYTE 10h, 20h, 30h
> ptr1 PBYTE ?
> ptr2 PBYTE array
> ```

&emsp;

* JMP指令

使用JMP指令无条件跳转到目标地址，该地址使用代码标号来标识，并被汇编器转换为偏移量。格式如下：
> `JMP destination`

使用JMP指令可以创造很多种程序结构，如循环：
>
> ```asm
> top:
>  ...
>  JMP top
> ```

&emsp;
