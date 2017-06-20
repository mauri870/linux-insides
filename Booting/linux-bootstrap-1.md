Processo de Booting do Kernel. Parte 1.
================================================================================

Do bootloader ao kernel
--------------------------------------------------------------------------------

Se você chegou a ler meus [blog posts](https://0xax.github.io/categories/assembler/) anteriores você pode ver que a algum tempo eu comecei a me envolver com programação de baixo nível. Eu escrevi alguns posts sobre programação assembly x86_64 para Linux e, por algum tempo, também me aventurei no código fonte do Linux. Eu tenho muito interesse em compreender como as coisas funcionam em baixo nível, como programas rodam no meu computador, como são organizados em memória, como o kernel manipula processos e a memória, como blocos são transferidos pela rede em baixo nível além de muitas outras coisas. Então eu decidi escrever mais uma série de outros posts sobre o kernel Linux para **x86_64**.

Note que eu não sou um especialista em kernel e de fato não escrevo código para ele como trabalho. É somente um passatempo. Eu simplesmente gosto de como as coisas funcionam a baixo nível e isso é interessante pra mim. Se você encontrar qualquer parte confusa ou tiver qualquer questão ou sugestão me contate pelo twitter [0xAX](https://twitter.com/0xAX), [email](anotherworldofworld@gmail.com) ou crie uma [issue](https://github.com/0xAX/linux-insides/issues/new). Todos os posts serão acessíveis também através do repositório [linux-insides](https://github.com/0xAX/linux-insides) e se caso encontrar algo errado com meu inglês(ou português, no caso do tradutor) fique a vontade para abrir uma pull request no respectivo repositório.


*Note que esta não é uma documentação oficial, estou somente distribuindo conhecimento.*

**Conhecimento necessário**

* Noções de código C
* Noções de código assembly (sintaxe AT&T)

De qualquer forma, se você for um iniciante eu tentarei explicar algumas partes durante esse e os próximos posts. Certo, esse é o fim da nossa curta introdução, vamos agora começar a nos aprofundar no kernel e nas operações de baixo nível.

Todo o código apresentado foi baseado no kernel 3.18. Se houverem quaisquer mudanças eu atualizarei os posts com as correções.

O botão liga/desliga mágico, mas o que acontece depois?
--------------------------------------------------------------------------------

Apesar dessa série de posts ser sobre o kernel Linux, nós não iniciaremos pelo código do kernel em si, pelo menos não nesse parágrafo. Assim que você preciona o botão mágico no seu laptop ou computador, ele liga e inicia normalmente. A placa-mãe envia um sinal para a [fonte de alimentação](https://en.wikipedia.org/wiki/Power_supply) que após receber esse sinal, provê a quantia necessária de energia para o computador. Assim que a placa-mãe recebe de volta o [sinal power good](https://en.wikipedia.org/wiki/Power_good_signal) ela tenta iniciar o CPU. O CPU então limpa todo e qualquer resquício de dados nos seus registradores e preenche cada um deles com valores predefinidos.


[80386](https://en.wikipedia.org/wiki/Intel_80386) e processadores seguintes utilizam os seguintes dados predefinidos nos registradores depois que o computador é iniciado:

```
IP          0xfff0
CS selector 0xf000
CS base     0xffff0000
```

O processador inicia trabalhando em [modo real] https://en.wikipedia.org/wiki/Real_mode). Vamos voltar um pouco e tentar entender a [segmentação de memória](https://en.wikipedia.org/wiki/Memory_segmentation) nesse modo. O modo real/real mode/RMode é suportado em todos os processadores compatíveis com a arquitetura x86, desde o [8086](https://en.wikipedia.org/wiki/Intel_8086) até os CPUs modernos Intel de 64 bits. O processador 8086 tem um barramento de endereços de 20 bits, o que quer dizer que ele pode trabalhar com espaços de memória 0-0xFFFFF (1 megabyte). Porém ele tem somente registradores de 16 bits, proporcionando um endereço máximo de `2^16 - 1` ou `0xffff` (64 kilobytes). 

[Segmentação de memória](http://en.wikipedia.org/wiki/Memory_segmentation) é utilizada para aproveitar todo o espaço de endereços disponível. A memória é dividida em partes pequenas e de um tamanho fixo de 65536 bytes (64 KB). Como nós não podemos ter endereços de memória menores que 64KB com registradores de 16 bits, um método alternativo foi criado. 

Um endereço consiste em duas partes: um seletor de segmento, que possui um endereço base, e um deslocamento ou offset sobre esse endereço base. No modo real/RMode o endereço de memória base associado a um seletor de segmento é `Seletor de Segmento * 16`. Entretanto, para conseguirmos o endereço físico em memória precisamos multiplicar o seletor de segmento por `16` e adicionar o offset:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

Por exemplo, se o `CS:IP` é `0x2000:0x0010` então o valor correspondente ao endereço físico é:

```python
>>> hex((0x2000 << 4) + 0x0010)
'0x20010'
```

Porém, se voce pegar o maior seletor de segmento e offset, `0xffff:0xffff`, o resultado será:

```python
>>> hex((0xffff << 4) + 0xffff)
'0x10ffef'
```

que é `65520` bytes a mais que o primeiro megabyte. Como somente um megabyte é acessível em modo real, `0x10ffef` se torna `0x00ffef` com a [linha A20](https://en.wikipedia.org/wiki/A20_line) desabilitada.

Certo, agora nós conhecemos um pouco mais sobre o modo real e sobre endereçamento de memória. Vamos voltar e discutir sobre os valores dos registradores depois do reset:

O registrador `CS` consiste de duas partes: o seletor de segmento visível e um endereço base desconhecido. Normalmente o endereço base é formado multiplicando o valor do seletor de segmento por 16, porém durante o reset do hardware o segmento seletor no registrador CS é carregado com `0xf000` e o endereço base com `0xffff0000`. O processador utiliza esse endereço base especial até que o `CS` seja modificado.

O endereço inicial é formado adicionando o endereço base ao valor do registrador EIP:

```python
>>> 0xffff0000 + 0xfff0
'0xfffffff0'
```

O resultado é `0xfffffff0`, que é 16 bytes abaixo dos 4GB. Esse ponto é chamado de [Reset vector](http://en.wikipedia.org/wiki/Reset_vector). Esse é o endereço de memória onde o CPU espera encontrar a primeira instrução de execução depois do reset. Ele contém uma instrução [jump](http://en.wikipedia.org/wiki/JMP_%28x86_instruction%29) (`jmp`) que normalmente aponta para o ponto de inicialização da BIOS. Por exemplo, se observarmos o código fonte  (`src/cpu/x86/16bit/reset16.inc`) do [coreboot](http://www.coreboot.org/), podemos ver o seguinte:

```assembly
    .section ".reset"
    .code16
.globl  reset_vector
reset_vector:
    .byte  0xe9
    .int   _start - ( . + 2 )
    ...
```

Podemos observar a instrução `jmp` [opcode](http://ref.x86asm.net/coder32.html#xE9), que é `0xe9`e seu endereço de destino em `_start - ( . + 2)`. 

Podemos ainda ver que a seção `reset` é composta por `16` bytes e inicia no endereço `0xfffffff0`:

```
SECTIONS {
    _ROMTOP = 0xfffffff0;
    . = _ROMTOP;
    .reset . : {
        *(.reset)
        . = 15 ;
        BYTE(0x00);
    }
}
```

Agora a BIOS inicializa. Depois de iniciar e verificar o hardware, a BIOS precisa encontrar um dispositivo bootável. A ordem de boot é armazenada na configuração da BIOS e controla quais dispositivos a BIOS deve tentar o processo de boot. Ao dar prosseguimento ao processo de boot de um disco rígido, a BIOS tenta encontrar um setor de boot. Em discos particionados com o [layout MBR](https://pt.wikipedia.org/wiki/Master_Boot_Record) o setor de boot fica localizado nos primeiros `446` bytes do primeiro setor, sendo que cada setor possui `512` bytes. Os dois bytes finais do primeiro setor são `0x55` e `0xaa`, esses dois bytes que dizem a BIOS que este dispositivo é bootável. Por exemplo:

```assembly
;
; Nota: esse exemplo é escrito na sintaxe Assembly Intel
;
[BITS 16]

boot:
    mov al, '!'
    mov ah, 0x0e
    mov bh, 0x00
    mov bl, 0x07

    int 0x10
    jmp $

times 510-($-$$) db 0

db 0x55
db 0xaa
```

Compile e execute o código acima com:

```
nasm -f bin boot.nasm && qemu-system-x86_64 boot
```

Isso irá instruir o [QEMU](http://qemu.org) a usar o binário `boot` que compilamos acima como uma imagem de disco. Como o binário gerado pelo código assembly satisfaz os requerimentos para um setor de boot (o início é setado para `0x7c00` e terminado com a sequência mágica) o QEMU tratará o executável como um master boot record (MBR) de uma imagem de disco.

Você verá o seguinte:

![Um bootloader simples que imprime `!`](http://oi60.tinypic.com/2qbwup0.jpg)

Nesse exemplo podemos ver que o código será executado em modo real de `16 bits` iniciando no endereço em memória`0x7c00`. Depois de inicializar, ele chama o interruptor [0x10](http://www.ctyme.com/intr/rb-0106.htm) que somente imprime o símbolo `!`. Ele também preenche os 510 bytes restantes com zeros e finaliza com os dois bytes mágicos `0xaa` e `0x55`.

Você pode ver um dump do binário utilizando a ferramenta `objdump`:

```
nasm -f bin boot.nasm
objdump -D -b binary -mi386 -Maddr16,data16,intel boot
```

Um setor de boot real teria mais código para seguir com o processo de boot e uma tabela de partição ao invés de vários zeros e um ponto de exclamação :) Daqui por diante a responsabilidade da BIOS termina e o controle é transferido para o bootloader.

**NOTA**: Como explicado acima, o CPU está em modo real. Nesse modo calcular o endereço físico em memória é feito da seguinte forma:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

como foi explicado anteriormente. Nós temos somente registradores de 16 bits, o valor máximo de um registrador de 16 bits é `0xffff`, logo se adotarmos os maiores valores teremos:

```python
>>> hex((0xffff * 16) + 0xffff)
'0x10ffef'
```

onde `0x10ffef` é igual a `1MB + 64KB - 16b`. Um processador [8086](https://en.wikipedia.org/wiki/Intel_8086) (que foi o primeiro cpu com modo real), em contraste, tem um barramento de endereços de 20 bits. Como `2^20 = 1048576` é 1MB temos um total de 1MB de memória disponível.

Segue um mapeamento geral de memória no modo real:

```
0x00000000 - 0x000003FF - Tabela Interrupt Vector do modo real
0x00000400 - 0x000004FF - Área de dados da BIOS
0x00000500 - 0x00007BFF - Não utilizado
0x00007C00 - 0x00007DFF - Nosso Bootloader
0x00007E00 - 0x0009FFFF - Não utilizado
0x000A0000 - 0x000BFFFF - Memória RAM de vídeo (VRAM)
0x000B0000 - 0x000B7777 - Memória de vídeo monocromática
0x000B8000 - 0x000BFFFF - Memória de vídeo em cores
0x000C0000 - 0x000C7FFF - Memória ROM de vídeo da BIOS
0x000C8000 - 0x000EFFFF - BIOS Shadow Area
0x000F0000 - 0x000FFFFF - BIOS do sistema
```

No início deste post eu escrevi que a primeira instrução executada pelo CPU é localizada no endereço `0xFFFFFFF0`, que é muito maior que `0xFFFFF` (1MB). Como pode o CPU acessar esse endereço em modo real? Podemos encontrar a resposta na documentação do [coreboot](http://www.coreboot.org/Developer_Manual/Memory_map):

```
0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapeada em espaço de endereçamento
```

No início da execução a BIOS não está na RAM, mas sim na ROM.

Bootloader
--------------------------------------------------------------------------------

Existem vários bootloaders que conseguem bootar o Linux, como o [GRUB 2](https://www.gnu.org/software/grub/) e o [syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project). O Kernel Linux tem um [protocolo de boot]((https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/boot.txt) que especifica os requerimentos para um bootloader implementar suporte ao Linux. Neste exemplo utilizaremos o GRUB2.

Continuando de onde paramos, agora que a `BIOS` encontrou um dispositivo de boot válido e transferiu o controle para o código no setor de boot a execução inicia apartir do [boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD). Esse código é bem simples devido a grande quandidade de espaço disponível e contém um ponteiro que é utilizado para pular para a localização da imagem principal do GRUB 2. A imagem principal começa com o [diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD), que normalmente é armazenado imediatamente após o primeiro setor no espaço disponível antes da primeira partição. O código acima carrega em memória o restante da imagem principal, que contém o kernel do GRUB2 e drivers para o sistema de arquivos.  Depois de carregar o restante da imagem principal, ele executa o [grub_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c).

O `grub_main` inicializa o console, obtém o endereço base para os módulos, seleciona o dispositivo root, carrega/analisa o arquivo de configuração do grub, carrega módulos, etc. No fim da execução, o `grub_main` coloca o grub em modo normal. `grub_normal_execute` (localizado em `grub-core/normal/main.c`) completa as preparações finais e mostra o menu para a escolha do sistema operacional. Quando selecionamos uma das entradas no menu do grub, `grub_menu_execute_entry` é chamada, executando o comando `boot` do grub e bootando o sistema operacional selecionado.

Como podemos ler no protocolo de boot do Kernel Linux, o bootloader precisa ler e preencher alguns campos do cabeçalho de inicialização do kernel, que inicia no deslocamento `0x01f1` a partir do código de inicialização do kernel. Você pode dar uma olhada no [script do linker]((https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L16) do boot para confirmar esse deslocamento. O cabeçalho do kernel [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S) inicia com:

```assembly
    .globl hdr
hdr:
    setup_sects: .byte 0
    root_flags:  .word ROOT_RDONLY
    syssize:     .long 0
    ram_size:    .word 0
    vid_mode:    .word SVGA_MODE
    root_dev:    .word 0
    boot_flag:   .word 0xAA55
```

O bootloader precisa preencher esse e o restante dos headers (que são marcados como tipo `write` no protocolo de boot Linux, como [neste exemplo](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/boot.txt#L354)) com valores que foram calculados ou recebidos da linha de comando. Nós não abordaremos cada detalhe dos headers de inicialização do kernel por agora mas sim como o kernel os utiliza. Você pode encontrar um resumo sobre os campos dos headers no [protocolo de boot](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/boot.txt#L156).

Como podemos ver no protocolo de boot do kernel, o mapeamento de memória será o seguinte depois de carregar o kernel:

```shell
         | Modo protegido do kernel  |
100000   +---------------------------+
         | Reservado para I/O        |
0A0000   +---------------------------+
         | Reservado para a BIOS     | Deixe o máximo possivel livre
         ~                           ~
         | Command line              | (Pode estar também abaixo d X+10000)
X+10000  +---------------------------+
         | Stack/heap                | Para uso do kernel em modo real
X+08000  +---------------------------+
         | Inicialização do kernel   | O código do modo real do kernel
         | Setor de boot do kernel   | O setor de boot legacy do kernel
       X +---------------------------+
         | Boot loader               | <- Entrada do setor de boot 0x7C00
001000   +---------------------------+
         | Reservado para MBR/BIOS   |
000800   +---------------------------+
         | Tipicamente MBR           |
000600   +---------------------------+
         | Uso exclusivo da BIOS     |
000000   +---------------------------+

```

Quando o bootloader transfere o controle para o kernel ele inicia em:

```
X + sizeof(KernelBootSector) + 1
```

onde `X` é o endereço do setor de boot do kernel que está sendo carregado. No nosso caso, `X` é `0x10000`, como podemos ver em um dump de memória:

![primeiro endereço do kernel](http://oi57.tinypic.com/16bkco2.jpg)

O bootloader agora tem o Linux Kernel carregado em memória e os campos do header preenchidos, então ele pula para o endereço de memória correspondente. Podemos então entrar no código de inicialização do kernel propriamente dito.

A inicialização do Kernel
--------------------------------------------------------------------------------

Finalmente estamos no kernel! Bem, quase lá... Tecnicamente o kernel não está rodando ainda. Primeiramente precisamos inicializar o kernel, gerenciador de memória, gerenciador de processos, etc... O código de inicialização do Kernel começa no arquivo [arch/x86/boot/header.S]((https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S) na seção [_start](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L292). Pode ser um pouco estranho a primeira vista já que existem várias instruções antes dela.

A muito tempo atrás, o Kernel Linux tinha o seu próprio bootloader. Agora, entretanto, se você tentar executar:

```
qemu-system-x86_64 vmlinuz-3.18-generic
```

você verá o seguinte:

![Teste o vmlinuz no qemu](http://oi60.tinypic.com/r02xkz.jpg)

De fato, o `header.S` inicia a partir do [MZ](https://en.wikipedia.org/wiki/DOS_MZ_executable) (veja a image acima), a mensagem de erro impressa segue o padrão de header [PE](https://pt.wikipedia.org/wiki/Portable_Executable):

```assembly
#ifdef CONFIG_EFI_STUB
# "MZ", MS-DOS header
.byte 0x4d
.byte 0x5a
#endif
...
...
...
pe_header:
    .ascii "PE"
    .word 0
```

Ele precisa disso para carregar um sistema operacional com suporte a [UEFI](https://pt.wikipedia.org/wiki/EFI). Não entraremos em mais detalhes sobre o seu funcionamento agora porém eles serão abordados em capítulos futuros.

O código de entrada do processo de inicialização do kernel é o seguinte:

```assembly
// header.S linha 292
.globl _start
_start:
```

O bootloader(GRUB2 e outros) tem conhecimento desse endereço (offset `0x200` a partir do `MZ`) e pulam diretamente para ele, apesar do `header.S` iniciar a partir da seção `.bstext`, que imprime uma mensagem de erro:

```
//
// arch/x86/boot/setup.ld
//
. = 0;                    // posição atual
.bstext : { *(.bstext) }  // coloca a seção .bstext na posição 0
.bsdata : { *(.bsdata) }
```

O código de inicialização é o seguinte:

```assembly
    .globl _start
_start:
    .byte  0xeb
    .byte  start_of_setup-1f
1:
    //
    // restante do header
    //
```

Aqui podemos ver uma instrução opcode `jmp` (`0xeb`) que pula para o ponto `start_of_setup-1f`. Na notação `Nf`, `2f` se refere ao rótulo `2:`. No nosso caso o rótulo `1` está presente logo após o jump. Ele contém o restante da inicialização do [header]((https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/boot.txt#L156). Depois da inicialização do header, podemos ver a seção `.entrytext`, que inicia no rótulo `start_of_setup`.

Esse é o primeiro código que de fato é executado (com exceçao das instruções jump anteriores, claro). Depois da configuração inicial do kernel receber o controle do bootloader, a primeira instrução `jmp` é localizada no offset `0x200` a partir do início do modo real do kernel, ou seja, depois dos primeiros 512 bytes. Nós podemos ver isso tanto lendo o protocolo de boot do Kernel Linux quanto o código fonte do GRUB2.

```C
segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;
```

Isso significa que os registradores de segmento conterão os seguindes valores depois da inicialização do kernel:

```
gs = fs = es = ds = ss = 0x10000
cs = 0x10200
```

No meu caso o kernel foi carregado no endereço `0x10000`.

Depois do jump para `start_of_setup`, o kernel precisa se encarregar do seguinte:

* Garantir que todos os valores nos registradores de segmento são iguais
* Inicializar a stack corretamente, se necessário
* Inicializar o [bss](https://en.wikipedia.org/wiki/.bss)
* Pular para o código C em [main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c)

Vamos conferir a implementação.

Alinhamento dos Registradores de Segmento
--------------------------------------------------------------------------------

Num primeiro momento o kernel se certifica que os registradores de segmento `ds` e `es` apontam para o mesmo endereço. Em seguida a flag de direção é limpa utilizando a instrução `cld`:

```assembly
    movw    %ds, %ax
    movw    %ax, %es
    cld
```

Como foi descrito anteriormente, o grub2 carrega o código de inicialização do kernel no endereço `0x10000` e `cs` em `0x1020` pelo fato da execução não começar do início do arquivo, mas sim do

```assembly
_start:
    .byte 0xeb
    .byte start_of_setup-1f
```

`jump`, a um offset de 512 bytes a partir de [4d 5a](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L46). Ele ainda precisa alinhar a `cs` de `0x10200` para `0x10000`, assim como todos os demais registradores de segmento. Depois disso, nós inicializamos a stack:

```assembly
    pushw   %ds
    pushw   $6f
    lretw
```

que envia o valor de `ds` para a stack com o endereço do label [6](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L494) e executa a instrução `lretw`. Quando a instrução `lretw` é chamada, ela carrega o endereço do label `6` no registrador [pc](https://en.wikipedia.org/wiki/Program_counter) e carrega  o `cs` com o valor de `ds`. Assim, `ds` e `cs` conterão os mesmos valores.

Inicialização da Stack
--------------------------------------------------------------------------------

Quase todo o código de inicialização ocorre em preparação para o ambiente de linguagem C no modo real. O próximo [passo](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L569) é checar o valor do registrador `ss` e criar a stack corretamente caso o `ss` esteja incorreto:

```assembly
    movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f
```

Isso pode resultar em 3 cenários distintos:

* `ss` tem o valor válido `0x1000` (assim como todos os outros registradores de segmento com exceção de `cs`)
* `ss` é inválido e a flag `CAN_USE_HEAP` está setada (veja abaixo)
* `ss` é inválido e a flag `CAN_USE_HEAP` não está setada (veja abaixo)

Vamos ver cada um desses cenários a seguir:

* `ss` tem o valor válido (`0x1000`). Nesse caso, nós pulamos para a label [2](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L584):

```assembly
2:  andw    $~3, %dx
    jnz     3f
    movw    $0xfffc, %dx
3:  movw    %ax, %ss
    movzwl  %dx, %esp
    sti
```

Aqui nós podemos ver o alinhamento de `dx` (contém o `sp` fornecido pelo bootloader) para 4 bytes e uma verificação se o resultado é zero ou não. Se for zero, nós armazenamos `0xfffc` (um endereço com alinhamento de 4 bytes antes do tamanho máximo de segmento que é 64 KB) em `dx`. Se não for zero, nós continuamos a utilizar o `sp`, fornecido pelo bootloader (0xf7f4 no meu caso). Depois disso, nós armazenamos o valor de `ax` em `ss`, que agora contém o endereço de segmento correto `0x1000` e inicializa o `sp` corretamente. Agora nós temos uma stack correta:

![stack](http://oi58.tinypic.com/16iwcis.jpg)

* No segundo cenário, (`ss` != `ds`). Primeiramente, nós armazenamos o valor de [_end](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L52) (o endereço do fim do código de inicialização) em `dx` e checamos o campo do header `loadflags` utilizando a instrução `testb` para verificar se podemos utilizar a heap. [loadflags](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L321) é um header bitmask definido como:

```C
#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)
```

e, como podemos ler no protocolo de boot:

```
Field name: loadflags

  This field is a bitmask.

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.
```

Se o bit `CAN_USE_HEAP` estiver setado, nós armazenaremos `heap_end_ptr` em `dx` (que aponta para `_end`) e adicionaremos `STACK_SIZE` (tamanho mínimo da stack, 1024 bytes) a ele. Depois disso, se `dx` não for carregado (ele não será carregado, dx = _end + 1024), pulamos para o label `2` (assim como descrito no cenário anterior) e criamos a stack correta.

![stack](http://oi62.tinypic.com/dr7b5w.jpg)

* Quando `CAN_USE_HEAP` não for setada, nós simplesmente utilizamos uma stack mínima de `_end` a `_end + STACK_SIZE`:

![minimal stack](http://oi60.tinypic.com/28w051y.jpg)

Inicialização BSS
--------------------------------------------------------------------------------

Os últimos dois passos necessários antes de chegarmos ao código C principal é inicializar a área [BSS](https://en.wikipedia.org/wiki/.bss) e checar a assinatura "mágica". Primeiramente, a checagem da assinatura:

```assembly
    cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad
```

O código acima simplesmente compara o [setup_sig](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L39) com o número mágico `0x5a5aaa55`. Se eles não forem iguais, um erro fatal é reportado.

Se ambos forem iguais, sabendo que temos os registradores de segmento setados corretamente e uma stack, nós somente precisamos setar a seção BSS antes de pular para o código C.

A seção BSS é utilizada para armazenar dados não inicializados, alocados estaticamente. O Linux se certifica cuidadosamente para que essa área seja limpa (preenchida com zeros) utilizando o código a seguir:

```assembly
    movw    $__bss_start, %di
    movw    $_end+3, %cx
    xorl    %eax, %eax
    subw    %di, %cx
    shrw    $2, %cx
    rep; stosl
```

Primeiro, o endereço [__bss_start]((https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L47) é movido para `di`. Em seguida, o endereço `_end + 3` (+3 - alinha para 4 bytes) é movido para `cx`. O registrador `eax` é limpo (utilizando uma instrução `xor`), e o tamanho da seção bss (`cx`-`di`) é calculado e colocado em `cx`. Então, `cx` é dividido por quatro (o tamanho de uma 'word' ou 'palavra'), e a instrução `stosl` é usada repetidamente, armazenando o valor de `eax` (zero) no endereço apontado por `di`, automaticamente incrementando `di` com 4, repetindo até que `cx` chegue a zero). Basicamente este código escreve zeros em todas as words em memória de `__bss_start` a `_end`:

![bss](http://oi59.tinypic.com/29m2eyr.jpg)

O salto para main
--------------------------------------------------------------------------------

Isso é tudo, nós temos a stack e a BSS, então podemos pular para a função C `main()`:

```assembly
    calll main
```

A função `main()` é implementada em [arch/x86/boot/main.c]((https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c). Você pode ler mais sobre o que essa função faz na próxima parte.

Conclusão
--------------------------------------------------------------------------------

Este é o fim da primeira parte sobre o interior do kernel Linux. Se você tiver alguma dúvida ou sugestão, me chame no twitter [0xAX](https://twitter.com/0xAX), me envie um [email](anotherworldofworld@gmail.com), ou simplesmente crie uma [issue](https://github.com/0xAX/linux-internals/issues/new). Na próxima parte, nós veremos o primeiro código C a ser executado na inicialização do kernel Linux, a implementação das rotinas de memória como `memset`, `memcpy`, `earlyprintk`, implementação e inicialização do console inicial e muito mais.

Links
--------------------------------------------------------------------------------

  * [Manual de referência do programador Intel 80386 - 1986](http://css.csail.mit.edu/6.858/2014/readings/i386.pdf)
  * [Boot Loader mínimo para arquitetura Intel®](https://www.cs.cmu.edu/~410/doc/minimal_boot.pdf)
  * [8086](http://pt.wikipedia.org/wiki/Intel_8086)
  * [80386](http://pt.wikipedia.org/wiki/Intel_80386)
  * [Reset vector](http://en.wikipedia.org/wiki/Reset_vector)
  * [Modo Real](http://pt.wikipedia.org/wiki/Modo_real)
  * [Protocolo de boot do kernel Linux](https://www.kernel.org/doc/Documentation/x86/boot.txt)
  * [Manual do desenvolvedor CoreBoot](http://www.coreboot.org/Developer_Manual)
  * [Lista de interruptores por Ralf Brown](http://www.ctyme.com/intr/int.htm)
  * [Fonte de alimentação](http://pt.wikipedia.org/wiki/Fonte_de_alimentação)
  * [Sinal Power good](http://en.wikipedia.org/wiki/Power_good_signal)
