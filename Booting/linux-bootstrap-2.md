Processo de Booting do Kernel. Parte 2.
================================================================================

Primeiros passos na inicialização do kernel
--------------------------------------------------------------------------------

Nós iniciamos nossa jornada pelo kernel Linux na [parte 1](linux-bootstrap-1.md) e vimos o início do código de inicialização do kernel. Nós paramos na primeira chamada para a função `main` (que é a propósito a primeira função escrita em C) em [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c).

Nesta parte continuaremos nossa jornada pelo kernel e veremos:
* o que é o `modo protegido`,
* a preparação e transição para ele,
* a inicialização da heap e console,
* detecção de memória, validação de CPU e inicialização do teclado
* e muito mais.

Então, vamos em frente!

Modo protegido
--------------------------------------------------------------------------------

Antes de entrarmos no nativo Intel64 [Long Mode](http://en.wikipedia.org/wiki/Long_mode), o kernel precisa colocar o CPU em modo protegido.

O que é [modo protegido](https://pt.wikipedia.org/wiki/Modo_protegido)? O modo protegido foi inicialmente adicionado na arquitetura x86 em 1982 e é o modo de operação principal de processadores Intel desde o [80286](https://pt.wikipedia.org/wiki/Intel_80286) até a chegada do Intel 64 e long mode.

A razão principal para sair do [Modo Real](http://wiki.osdev.org/Real_Mode) é que o accesso a memória RAM é muito limitado. Como você deve lembrar da parte 1, estão disponíveis somente 2<sup>20</sup> bytes ou 1 Megabyte, algumas vezes somente 640 Kilobytes de RAM estão disponíveis no Modo Real.

O Modo Protegido traz muitas mudanças, porém a principal é a diferença no gerenciamento de memória. O barramento de endereços de 20-bit é substituído por um de 32-bit. Isso possibilita acesso a 4 Gigabytes de memória contra o 1 Megabyte disponível em Modo Real. Além disso, suporte a [paginação de memória](https://pt.wikipedia.org/wiki/Memória_paginada) é adicionado, que você pôde conferir na próxima seção.

O gerenciamento de memória no Modo Protegido é separado em duas etapas praticamente independentes:

* Segmentação
* Paginação

Vamos ver primeiro sobre segmentação, paginação será abordada na seção seguinte.

Como você pode ler na parte anterior, endereços são compostos por dois componentes no modo real:

* Endereço base do segmento
* Deslocamento/Offset a partir do segmento base

E nós podemos computar o endereço físico se tivermos conhecimento desses dois valores acima:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

Segmentação de memória foi completamente removida no modo protegido. Não existem segmentos de tamanho fixo em 64 Kilobytes, o tamanho e localização de cada segmento é definido por uma estrutura de dados chamada de _Segment Descriptor/Descritores de Segmento_. Estes descritores são armazenados em uma estrutura de dados chamada `Global Descriptor Table` (GDT).

A GDT é uma estrutura que é mantida em memória. Ela não tem um endereço físico, portanto existe um registrador especial chamado `GDTR` responsável por armazenar esse endereço. Nós veremos mais adiante a GDT sendo carregada pelo kernel Linux. Encontraremos uma operação para carregá-la em memória, algo como:

```assembly
lgdt gdt
```

onde a instrução `lgdt` carrega o endereço base e o limite(tamanho) da GDT no registrador `GDTR`. O `GDTR` é um registrador de 48-bit que é composto por duas partes:

 * tamanho(16-bit) da GDT;
 * endereço(32-bit) da GDT

Como dito acima a GDT contém descritores de segmento que representam segmentos de memória.  Cada descritor possui 64-bits de tamanho. O esquema padrão de um descritor é o seguinte:

```
 63         56         51   48    45           39        32 
------------------------------------------------------------
|             | |B| |A|       | |   | |0|E|W|A|            |
| BASE 31:24  |G|/|L|V| LIMIT |P|DPL|S|  TYPE | BASE 23:16 |
|             | |D| |L| 19:16 | |   | |1|C|R|A|            |
------------------------------------------------------------

 31                         16 15                         0 
------------------------------------------------------------
|                             |                            |
|        BASE 15:0            |       LIMIT 15:0           |
|                             |                            |
------------------------------------------------------------
```

Não se preocupe. Eu sei que parece um pouco assustador depois do Modo Real, mas na verdade é simples. Por exemplo LIMIT 15:0 significa que os bits 0-15 de LIMIT estão localizados no início do descritor. O restante está em LIMIT 19:16, que está localizado entre os bits 48-51 do descritor. O tamanho de LIMIT então é 0-19 ou 20-bits. Vamos ver isso com mais detalhes:

1. Limit[20-bits] está entre 0-15, 48-51 bits. Ele define `tamanho_do_segmento - 1`. Ele depende do bit `G`(Granularidade).

  * se `G` (bit 55) e o limite de segmento forem 0, o tamanho do segmento é 1 Byte
  * se `G` for 1 e o limite de segmento for 0, o tamanho do segmento é 4096 Bytes
  * se `G` for 0 e o limite de segmento for 0xfffff, o tamanho do segmento é 1 Megabyte
  * se `G` for 1 e o limite de segmento for 0xfffff, o tamanho do segmento é 4 Gigabytes

  Resumidamente:
  * se G for 0, Limit é interpretado com uma escala de 1 Byte e o tamanho máximo do segmento é 1 Megabyte.
  * se G for 1, Limit é interpretado com uma escala de 4096 Bytes = 4 KBytes = 1 Página/Page e o tamanho máximo do segmento pode ser 4 Gigabytes. Na prática, quando G é igual a 1, o valor de Limit é deslocado 12 bits para a esquerda. Então, 20 bits + 12 bits = 32 bits e 2<sup>32</sup> = 4 Gigabytes.

2. Base[32-bits] está entre 16-31, 32-39 e 56-63 bits. Ele define o endereço físico da localização de início do segmento.

3. Type/Attribute[5-bits] está entre 40-44 bits. Ele define o tipo de segmento e as formas de acesso a ele.
  * A flag `S` no bit 44 especifica o tipo de descritor. Se `S` for 0 então esse segmento é um segmento de sistema, caso contrário se `S` for 1 então este é um segmento de código ou dados (segmentos de Stack são segmentos de dados que precisam de acesso leitura/escrita).

Para determinar se o segmento é de código ou dados nós podemos checar o atributo (bit 43) marcado como 0 no diagrama acima. Se ele for igual a 0, então o segmento é de dados caso contrário ele é um segmento de código.

Um segmento pode ser um dos tipos abaixo:

```
|           Type Field        | Descriptor Type | Description
|-----------------------------|-----------------|------------------
| Decimal                     |                 |
|             0    E    W   A |                 |
| 0           0    0    0   0 | Data            | Read-Only
| 1           0    0    0   1 | Data            | Read-Only, accessed
| 2           0    0    1   0 | Data            | Read/Write
| 3           0    0    1   1 | Data            | Read/Write, accessed
| 4           0    1    0   0 | Data            | Read-Only, expand-down
| 5           0    1    0   1 | Data            | Read-Only, expand-down, accessed
| 6           0    1    1   0 | Data            | Read/Write, expand-down
| 7           0    1    1   1 | Data            | Read/Write, expand-down, accessed
|                  C    R   A |                 |
| 8           1    0    0   0 | Code            | Execute-Only
| 9           1    0    0   1 | Code            | Execute-Only, accessed
| 10          1    0    1   0 | Code            | Execute/Read
| 11          1    0    1   1 | Code            | Execute/Read, accessed
| 12          1    1    0   0 | Code            | Execute-Only, conforming
| 14          1    1    0   1 | Code            | Execute-Only, conforming, accessed
| 13          1    1    1   0 | Code            | Execute/Read, conforming
| 15          1    1    1   1 | Code            | Execute/Read, conforming, accessed
```

Como podemos ver o primeiro bit (bit 43) é `0` para um segmento _data_ e `1` para um segmento _code_. Os próximos bits (40, 41, 42) podem ser `EWA`(*E*xpansion *W*ritable *A*ccessible) ou CRA(*C*onforming *R*eadable *A*ccessible).
  * se E(bit 42) for igual a 0, ele é incrementado, caso contrário é reduzido. Leia mais [aqui](http://www.sudleyplace.com/dpmione/expanddown.html).
  * se W(bit 41)(para segmentos de dados) for 1, o acesso de escrita é permitido caso contrário não. Vale ressaltar que o acesso a leitura é sempre permitido em segmentos de dados.
  * A(bit 40) - indica se o segmento é acessado pelo processador ou não.
  * C(bit 43) é o conforming bit/bit de conformidade(para segmentos de código). Se C for 1, o segmento de código pode ser executado com um nível inferior de privilégio, como por exemplo em nível de usuário. Se C for 0, ele pode somente ser executado com o mesmo nível de privilégio.
  * R(bit 41)(para segmentos de código). Se for 1 o acesso a leitura no segmento é permitido caso contrário não. Acesso a escrita nunca é permitido em segmentos de código.

4. DPL[2-bits] (Descriptor Privilege Level/Descritor de Nível de Privilégio) está entre os bits 45-46. Ele define o nível de privilégio do segmento. O valor pode ser 0-3 onde 0 é o nível mais privilegiado.

5. Flag P(bit 47) - indica se o segmento está presente em memória ou não. Se P for 0, o segmento será apresentado como _invalid_ e o processador se recusará a ler este segmento.

6. Flag AVL (bit 52) - Disponível e reservada. Ela é ignorada no Linux.

7. Flag L(bit 53) - indica se o segmento de código contém ou não código 64-bit nativo. Se seu valor for 1 então o segmento executa em modo 64-bit.

8. Flag D/B(bit 54) - chamada de Default/Big ela representa o tamanho do operando, 16/32 bits. Se o bit estiver setado então o tamanho é 32 caso contrário 16.

Registradores de Segmento contém seletores de segmento assim como no modo real. Entretanto, no modo protegido, o seletor de segmento é manuseado de forma diferente. Cada Descritor de Segmento tem um Seletor de Segmento associado que é uma estrutura de 16-bit:

```
 15             3 2  1     0
-----------------------------
|      Index     | TI | RPL |
-----------------------------
```

Onde,
* **Index** mostra o número de indexação do descritor na GDT.
* **TI**(Table Indicator) mostra onde procurar pelo descritor. Se ela for zero 0  então deve-se pesquisar na Global Descriptor Table(GDT), caso contrário na Local Descriptor Table(LDT).
* E **RPL** é o Requester's Privilege Level/Nível de Privilégio do Requerente.

Cada registrador de segmento tem uma parte visível e outra não.
* Visível - o Seletor de Segmento é armazenado aqui
* Oculto - Descritor de segmento(base, limit, attributes, flags)

Os seguintes passos são necessários para recuperarmos o endereço físico no modo protegido:

* The segment selector must be loaded in one of the segment registers
* The CPU tries to find a segment descriptor by GDT address + Index from selector and load the descriptor into the *hidden* part of the segment register
* Base address (from segment descriptor) + offset will be the linear address of the segment which is the physical address (if paging is disabled).

Schematically it will look like this:

![linear address](http://oi62.tinypic.com/2yo369v.jpg)

The algorithm for the transition from real mode into protected mode is:

* Disable interrupts
* Describe and load GDT with `lgdt` instruction
* Set PE (Protection Enable) bit in CR0 (Control Register 0)
* Jump to protected mode code

We will see the complete transition to protected mode in the linux kernel in the next part, but before we can move to protected mode, we need to do some more preparations.

Let's look at [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c). We can see some routines there which perform keyboard initialization, heap initialization, etc... Let's take a look.

Copying boot parameters into the "zeropage"
--------------------------------------------------------------------------------

We will start from the `main` routine in "main.c". First function which is called in `main` is [`copy_boot_params(void)`](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c#L30). It copies the kernel setup header into the field of the `boot_params` structure which is defined in the [arch/x86/include/uapi/asm/bootparam.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/uapi/asm/bootparam.h#L113).

The `boot_params` structure contains the `struct setup_header hdr` field. This structure contains the same fields as defined in [linux boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt) and is filled by the boot loader and also at kernel compile/build time. `copy_boot_params` does two things:

1. Copies `hdr` from [header.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L281) to the `boot_params` structure in `setup_header` field

2. Updates pointer to the kernel command line if the kernel was loaded with the old command line protocol.

Note that it copies `hdr` with `memcpy` function which is defined in the [copy.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/copy.S) source file. Let's have a look inside:

```assembly
GLOBAL(memcpy)
    pushw   %si
    pushw   %di
    movw    %ax, %di
    movw    %dx, %si
    pushw   %cx
    shrw    $2, %cx
    rep; movsl
    popw    %cx
    andw    $3, %cx
    rep; movsb
    popw    %di
    popw    %si
    retl
ENDPROC(memcpy)
```

Yeah, we just moved to C code and now assembly again :) First of all, we can see that `memcpy` and other routines which are defined here, start and end with the two macros: `GLOBAL` and `ENDPROC`. `GLOBAL` is described in [arch/x86/include/asm/linkage.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/linkage.h) which defines `globl` directive and the label for it. `ENDPROC` is described in [include/linux/linkage.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/linkage.h) which marks the `name` symbol as a function name and ends with the size of the `name` symbol.

Implementation of `memcpy` is easy. At first, it pushes values from the `si` and `di` registers to the stack to preserve their values because they will change during the `memcpy`. `memcpy` (and other functions in copy.S) use `fastcall` calling conventions. So it gets its incoming parameters from the `ax`, `dx` and `cx` registers.  Calling `memcpy` looks like this:

```c
memcpy(&boot_params.hdr, &hdr, sizeof hdr);
```

So,
* `ax` will contain the address of the `boot_params.hdr`
* `dx` will contain the address of `hdr`
* `cx` will contain the size of `hdr` in bytes.

`memcpy` puts the address of `boot_params.hdr` into `di` and saves the size on the stack. After this it shifts to the right on 2 size (or divide on 4) and copies from `si` to `di` by 4 bytes. After this, we restore the size of `hdr` again, align it by 4 bytes and copy the rest of the bytes from `si` to `di` byte by byte (if there is more). Restore `si` and `di` values from the stack in the end and after this copying is finished.

Console initialization
--------------------------------------------------------------------------------

After `hdr` is copied into `boot_params.hdr`, the next step is console initialization by calling the `console_init` function which is defined in [arch/x86/boot/early_serial_console.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/early_serial_console.c).

It tries to find the `earlyprintk` option in the command line and if the search was successful, it parses the port address and baud rate of the serial port and initializes the serial port. The value of `earlyprintk` command line option can be one of these:

* serial,0x3f8,115200
* serial,ttyS0,115200
* ttyS0,115200

After serial port initialization we can see the first output:

```C
if (cmdline_find_option_bool("debug"))
    puts("early console in setup code\n");
```

The definition of `puts` is in [tty.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/tty.c). As we can see it prints character by character in a loop by calling the `putchar` function. Let's look into the `putchar` implementation:

```C
void __attribute__((section(".inittext"))) putchar(int ch)
{
    if (ch == '\n')
        putchar('\r');

    bios_putchar(ch);

    if (early_serial_base != 0)
        serial_putchar(ch);
}
```

`__attribute__((section(".inittext")))` means that this code will be in the `.inittext` section. We can find it in the linker file [setup.ld](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L19).

First of all, `putchar` checks for the `\n` symbol and if it is found, prints `\r` before. After that it outputs the character on the VGA screen by calling the BIOS with the `0x10` interrupt call:

```C
static void __attribute__((section(".inittext"))) bios_putchar(int ch)
{
    struct biosregs ireg;

    initregs(&ireg);
    ireg.bx = 0x0007;
    ireg.cx = 0x0001;
    ireg.ah = 0x0e;
    ireg.al = ch;
    intcall(0x10, &ireg, NULL);
}
```

Here `initregs` takes the `biosregs` structure and first fills `biosregs` with zeros using the `memset` function and then fills it with register values.

```C
    memset(reg, 0, sizeof *reg);
    reg->eflags |= X86_EFLAGS_CF;
    reg->ds = ds();
    reg->es = ds();
    reg->fs = fs();
    reg->gs = gs();
```

Let's look at the [memset](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/copy.S#L36) implementation:

```assembly
GLOBAL(memset)
    pushw   %di
    movw    %ax, %di
    movzbl  %dl, %eax
    imull   $0x01010101,%eax
    pushw   %cx
    shrw    $2, %cx
    rep; stosl
    popw    %cx
    andw    $3, %cx
    rep; stosb
    popw    %di
    retl
ENDPROC(memset)
```

As you can read above, it uses the `fastcall` calling conventions like the `memcpy` function, which means that the function gets parameters from `ax`, `dx` and `cx` registers.

Generally `memset` is like a memcpy implementation. It saves the value of the `di` register on the stack and puts the `ax` value into `di` which is the address of the `biosregs` structure. Next is the `movzbl` instruction, which copies the `dl` value to the low 2 bytes of the `eax` register. The remaining 2 high bytes  of `eax` will be filled with zeros.

The next instruction multiplies `eax` with `0x01010101`. It needs to because `memset` will copy 4 bytes at the same time. For example, we need to fill a structure with `0x7` with memset. `eax` will contain `0x00000007` value in this case. So if we multiply `eax` with `0x01010101`, we will get `0x07070707` and now we can copy these 4 bytes into the structure. `memset` uses `rep; stosl` instructions for copying `eax` into `es:di`.

The rest of the `memset` function does almost the same as `memcpy`.

After the `biosregs` structure is filled with `memset`, `bios_putchar` calls the [0x10](http://www.ctyme.com/intr/rb-0106.htm) interrupt which prints a character. Afterwards it checks if the serial port was initialized or not and writes a character there with [serial_putchar](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/tty.c#L30) and `inb/outb` instructions if it was set.

Heap initialization
--------------------------------------------------------------------------------

After the stack and bss section were prepared in [header.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S) (see previous [part](linux-bootstrap-1.md)), the kernel needs to initialize the [heap](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c#L116) with the [`init_heap`](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c#L116) function.

First of all `init_heap` checks the [`CAN_USE_HEAP`](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/uapi/asm/bootparam.h#L22) flag from the [`loadflags`](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L321) in the kernel setup header and calculates the end of the stack if this flag was set:

```C
    char *stack_end;

    if (boot_params.hdr.loadflags & CAN_USE_HEAP) {
        asm("leal %P1(%%esp),%0"
            : "=r" (stack_end) : "i" (-STACK_SIZE));
```

or in other words `stack_end = esp - STACK_SIZE`.

Then there is the `heap_end` calculation:

```C
     heap_end = (char *)((size_t)boot_params.hdr.heap_end_ptr + 0x200);
```

which means `heap_end_ptr` or `_end` + `512` (`0x200h`). The last check is whether `heap_end` is greater than `stack_end`. If it is then `stack_end` is assigned to `heap_end` to make them equal.

Now the heap is initialized and we can use it using the `GET_HEAP` method. We will see how it is used, how to use it and how it is implemented in the next posts.

CPU validation
--------------------------------------------------------------------------------

The next step as we can see is cpu validation by `validate_cpu` from [arch/x86/boot/cpu.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/cpu.c).

It calls the [`check_cpu`](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/cpucheck.c#L112) function and passes cpu level and required cpu level to it and checks that the kernel launches on the right cpu level.
```c
check_cpu(&cpu_level, &req_level, &err_flags);
if (cpu_level < req_level) {
    ...
    return -1;
}
```
`check_cpu` checks the CPU's flags, the presence of [long mode](http://en.wikipedia.org/wiki/Long_mode) in the case of x86_64(64-bit) CPU, checks the processor's vendor and makes preparation for certain vendors like turning off SSE+SSE2 for AMD if they are missing, etc.

Memory detection
--------------------------------------------------------------------------------

The next step is memory detection by the `detect_memory` function. `detect_memory` basically provides a map of available RAM to the CPU. It uses different programming interfaces for memory detection like `0xe820`, `0xe801` and `0x88`. We will see only the implementation of **0xE820** here.

Let's look into the `detect_memory_e820` implementation from the [arch/x86/boot/memory.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/memory.c) source file. First of all, the `detect_memory_e820` function initializes the `biosregs` structure as we saw above and fills registers with special values for the `0xe820` call:

```assembly
    initregs(&ireg);
    ireg.ax  = 0xe820;
    ireg.cx  = sizeof buf;
    ireg.edx = SMAP;
    ireg.di  = (size_t)&buf;
```

* `ax` contains the number of the function (0xe820 in our case)
* `cx` register contains size of the buffer which will contain data about memory
* `edx` must contain the `SMAP` magic number
* `es:di` must contain the address of the buffer which will contain memory data
* `ebx` has to be zero.

Next is a loop where data about the memory will be collected. It starts from the call of the `0x15` BIOS interrupt, which writes one line from the address allocation table. For getting the next line we need to call this interrupt again (which we do in the loop). Before the next call `ebx` must contain the value returned previously:

```C
    intcall(0x15, &ireg, &oreg);
    ireg.ebx = oreg.ebx;
```

Ultimately, it does iterations in the loop to collect data from the address allocation table and writes this data into the `e820_entry` array:

* start of memory segment
* size  of memory segment
* type of memory segment (which can be reserved, usable and etc...).

You can see the result of this in the `dmesg` output, something like:

```
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x000000003ffdffff] usable
[    0.000000] BIOS-e820: [mem 0x000000003ffe0000-0x000000003fffffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000fffc0000-0x00000000ffffffff] reserved
```

Next we may see call of the `set_bios_mode` function after physical memory is detected. As we may see, this function is implemented only for the `x86_64` mode:

```C
static void set_bios_mode(void)
{
#ifdef CONFIG_X86_64
	struct biosregs ireg;

	initregs(&ireg);
	ireg.ax = 0xec00;
	ireg.bx = 2;
	intcall(0x15, &ireg, NULL);
#endif
}
```

The `set_bios_mode` executes `0x15` BIOS interrupt to tell the BIOS that [long mode](https://en.wikipedia.org/wiki/Long_mode) (in a case of `bx = 2`) will be used.

Keyboard initialization
--------------------------------------------------------------------------------

The next step is the initialization of the keyboard with the call of the [`keyboard_init()`](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c#L65) function. At first `keyboard_init` initializes registers using the `initregs` function and calling the [0x16](http://www.ctyme.com/intr/rb-1756.htm) interrupt for getting the keyboard status.

```c
    initregs(&ireg);
    ireg.ah = 0x02;     /* Get keyboard status */
    intcall(0x16, &ireg, &oreg);
    boot_params.kbd_status = oreg.al;
```

After this it calls [0x16](http://www.ctyme.com/intr/rb-1757.htm) again to set repeat rate and delay.

```c
    ireg.ax = 0x0305;   /* Set keyboard repeat rate */
    intcall(0x16, &ireg, NULL);
```

Querying
--------------------------------------------------------------------------------

The next couple of steps are queries for different parameters. We will not dive into details about these queries but will get back to it in later parts. Let's take a short look at these functions:

The first step is getting [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep) information by calling the `query_ist` function. First of all, it checks the CPU level and if it is correct, calls `0x15` for getting info and saves the result to `boot_params`.

The following [query_apm_bios](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/apm.c#L21) function gets [Advanced Power Management](http://en.wikipedia.org/wiki/Advanced_Power_Management) information from the BIOS. `query_apm_bios` calls the `0x15` BIOS interruption too, but with `ah` = `0x53` to check `APM` installation. After the `0x15` execution, `query_apm_bios` functions check the `PM` signature (it must be `0x504d`), carry flag (it must be 0 if `APM` supported) and value of the `cx` register (if it's 0x02, protected mode interface is supported).

Next, it calls `0x15` again, but with `ax = 0x5304` for disconnecting the `APM` interface and connecting the 32-bit protected mode interface. In the end, it fills `boot_params.apm_bios_info` with values obtained from the BIOS.

Note that `query_apm_bios` will be executed only if `CONFIG_APM` or `CONFIG_APM_MODULE` was set in the configuration file:

```C
#if defined(CONFIG_APM) || defined(CONFIG_APM_MODULE)
    query_apm_bios();
#endif
```

The last is the [`query_edd`](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/edd.c#L122) function, which queries `Enhanced Disk Drive` information from the BIOS. Let's look into the `query_edd` implementation.

First of all, it reads the [edd](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/kernel-parameters.txt#L1023) option from the kernel's command line and if it was set to `off` then `query_edd` just returns.

If EDD is enabled, `query_edd` goes over BIOS-supported hard disks and queries EDD information in the following loop:

```C
for (devno = 0x80; devno < 0x80+EDD_MBR_SIG_MAX; devno++) {
    if (!get_edd_info(devno, &ei) && boot_params.eddbuf_entries < EDDMAXNR) {
        memcpy(edp, &ei, sizeof ei);
        edp++;
        boot_params.eddbuf_entries++;
    }
    ...
    ...
    ...
    }
```

where `0x80` is the first hard drive and the value of `EDD_MBR_SIG_MAX` macro is 16. It collects data into the array of [edd_info](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/uapi/linux/edd.h#L172) structures. `get_edd_info` checks that EDD is present by invoking the `0x13` interrupt with `ah` as `0x41` and if EDD is present, `get_edd_info` again calls the `0x13` interrupt, but with `ah` as `0x48` and `si` containing the address of the buffer where EDD information will be stored.

Conclusion
--------------------------------------------------------------------------------

This is the end of the second part about Linux kernel insides. In the next part, we will see video mode setting and the rest of preparations before the transition to protected mode and directly transitioning into it.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me a PR to [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [Protected mode](http://wiki.osdev.org/Protected_Mode)
* [Long mode](http://en.wikipedia.org/wiki/Long_mode)
* [Nice explanation of CPU Modes with code](http://www.codeproject.com/Articles/45788/The-Real-Protected-Long-mode-assembly-tutorial-for)
* [How to Use Expand Down Segments on Intel 386 and Later CPUs](http://www.sudleyplace.com/dpmione/expanddown.html)
* [earlyprintk documentation](http://lxr.free-electrons.com/source/Documentation/x86/earlyprintk.txt)
* [Kernel Parameters](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/kernel-parameters.txt)
* [Serial console](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/serial-console.txt)
* [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep)
* [APM](https://en.wikipedia.org/wiki/Advanced_Power_Management)
* [EDD specification](http://www.t13.org/documents/UploadedDocuments/docs2004/d1572r3-EDD3.pdf)
* [TLDP documentation for Linux Boot Process](http://www.tldp.org/HOWTO/Linux-i386-Boot-Code-HOWTO/setup.html) (old)
* [Previous Part](linux-bootstrap-1.md)
