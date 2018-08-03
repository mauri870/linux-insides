Desenvolvimento do Kernel linux
================================================================================

Introdução
--------------------------------------------------------------------------------
Como vocês já sabem, eu iniciei uma série de [postagens em meu bog](http://0xax.github.io/categories/assembly/) no último ano sobre programação em assembly para arquiteturas `x86_64`. Eu nunca havia escrito uma linha de código de baixo nível antes disso, exceto por alguns exemplos de `Hello World` na universidade. Isso foi a bastante tempo atrás e, como eu já havia dito, eu nunca havia escrito uma linha de código de baixo nível até então. Algum tempo atrás eu comecei a me interessar em tais coisas. Eu entendi que eu posso escrever programas, mas eu na verdade não entendi como meu programa é organizado.

Após escrever algum código em assember eu comecei a entender como meu programa se assemelha após compilado, **aproximadamente**. Mas de qualquer forma, eu não entendi muitas outras coisas. Por exemplo: o que acontece quando a instrução `syscall` é executada em meu código assembler, o que acontece quando a função `printf` começa a funcionar, ou como pode meu programa conversar com outros computadores via rede.


A linguagem de programação [Assembler](https://en.wikipedia.org/wiki/Assembly_language#Assembler) não proveu respostas para as minhas questões e eu decidi me aprofundar em minha pesquisa. Eu comecei a aprender através do código fonte do kernel Linux e tentar entender as coisas que eu estava interessado. O código fonte do kernel Linux não me deu as respostas para **todas** as minhas questões, mas agora meu conhecimento sobre o kernel Linux e dos processos em sua volta é muito melhor.

Eu estou escrevendo esta parte há nove meses e meio após eu começar a aprender através do código fonte do kernel Linux e então publicar a primeira [parte](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-1.html) deste livro. Agora ele contém quarenta partes e isso não é o fim. Eu decidi escrever esta série sobre o kernel Linux eu mesmo. Como você sabe, o kernel Linux é um pedaço gigante de código e é fácil de esquecer o que essa ou aquela outra parte do kernel Linux significa e como isso implementa alguma coisa. Mas logo que o repositório [linux-insides](https://github.com/0xAX/linux-insides) tornou-se popular, e após nove meses, ele possui `9096` estrelas:

![github](http://s2.postimg.org/jjb3s4frt/stars.png)

Parece que as pessoas estão interessadas no kernel Linux. Além disso, durante todo o tempo em que eu estive escrevendo `linux-insides`, eu tenho recebido muitas perguntas de diferentes pessoas sobre como começar a contribuir para o kernel Linux. Geralmente pessoas estão interessadas em contribuir para projetos open-source e o kernel Linux não é uma exceção:

![google-linux](http://s4.postimg.org/yg9z5zx0d/google_linux.png)

Então, parece que as pessoas estão interessadas no processo de desenvolvimento do kernel Linux. Eu pensava que seria estranho se um livro sobre o kernel Linux não contesse uma parte descrevendo como fazer parte do desenvolvimento do kernel Linux e isso é a razão pela qual eu decidi escrever este livro. Você não irá encontrar informações sobre por que você deveria estar interessado em contribuir para o kernel Linux nesta parte. Mas se você está interessado em como iniciar com o desenvolvimento do kernel Linux, esta parte é para você.

Vamos começar.

Como iniciar com o kernel Linux
---------------------------------------------------------------------------------

Primeiro de tudo, vamos começar em como obter, compilar, e executar o kernel Linux. Você pode executar seu compilado customizado do kernel Linux de duas formas:
* Rodando o kernel Linux em uma máquina virtual;
* Rodando o kernel Linux em uma máquina real.

Eu irei guiá-lo em ambos os métodos. Antes de nós começarmos a fazer qualquer coisa com o kernel Linux, nós precisamos obtê-lo. Existem algumas maneiras de se fazer isso dependendo de sua proposta. Se você apenas quer atualizar a versão atual do kernel Linux no seu computador, você pode utilizar as instruções específicas para a sua [distribuição](https://en.wikipedia.org/wiki/Linux_distribution) Linux.

No primeiro caso você apenas precisa baixar uma nova versão do kernel Linux utilizando o [package manager](https://en.wikipedia.org/wiki/Package_manager). Por exemplo, para atualizar a versão do kernel Linux para a versão `4.1` para o [Ubuntu (Vivid Vervet)](http://releases.ubuntu.com/15.04/), você irá apenas precisar executar os seguintes comandos:

```
$ sudo add-apt-repository ppa:kernel-ppa/ppa
$ sudo apt-get update
```

Após isso, execute estes comandos:

```
$ apt-cache showpkg linux-headers
```

e escolha a versão do kernel Linux no qual você está interessado. No final, execute o próximo comando e substitua `${version}` com a versão que você escolheu na saída do comando anterior:

```
$ sudo apt-get install linux-headers-${version} linux-headers-${version}-generic linux-image-${version}-generic --fix-missing
```

e então reinicie seu sistema. Após reiniciá-lo, você verá o novo kernel no menu do [grub](https://en.wikipedia.org/wiki/GNU_GRUB).

De outra forma, se você está interessado no desenvolvimento do kernel Linux, você irá precisar obter o código fonte do kernel Linux. Você pode encontrá-lo no website [kernel.org](https://kernel.org/) e baixar um arquivo com o código fonte do kernel Linux. Na realidade, o processo de desenvolvimento do kernel Linux é completamente realizado em torno do `git` [version control system](https://en.wikipedia.org/wiki/Version_control). Então, você pode obtê-lo utilizando o `git` através do `kernel.org`:

```
$ git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

Eu não sei quanto a você, mas eu prefiro o `github`. Existe um [mirror](https://github.com/torvalds/linux) do repositório principal do kernel Linux, desta forma você pode clona-lo com:

```
$ git clone git@github.com:torvalds/linux.git
```

Eu utilizo meu próprio [fork](https://github.com/0xAX/linux) para o desenvolvimento e quando eu quero obter atualizações do repositório principal, eu apenas executo o seguinte comando:

```
$ git checkout master
$ git pull upstream master
```

Note que o nome remoto do repositório principal é `upstream`. Para adicionar um novo remoto com o repositório principal do Linux, você pode executar:

```
git remote add upstream git@github.com:torvalds/linux.git
```

Após isso, você terá dois remotos:

```
~/dev/linux (master) $ git remote -v
origin	git@github.com:0xAX/linux.git (fetch)
origin	git@github.com:0xAX/linux.git (push)
upstream	https://github.com/torvalds/linux.git (fetch)
upstream	https://github.com/torvalds/linux.git (push)
```

Um é o seu fork (`origin`) e o segundo é para o repositório principal (`upstream`).

Agora que nós temos uma cópia local do código fonte do kernel Linux, nós precisamos configurá-lo e compilá-lo. O kernel Linux pode ser configurado de diferentes formas. A forma mais simples é apenas copiar o arquivo de configuração do kernel já instalado em sua máquina que está localizado no diretório `/boot`:

```
$ sudo cp /boot/config-$(uname -r) ~/dev/linux/.config
```

Se o seu kernel Linux atual foi compilado com o suporte para o acesso ao arquivo `/proc/config.gz`, você pode copiar o arquivo de configuração do seu kernel atual com o seguinte comando:

```
$ cat /proc/config.gz | gunzip > ~/dev/linux/.config
```

Se você não está satisfeito com a configuração padrão do kernel que é providenciada pelos mantenedores de sua distribuição, você pode configurar o kernel Linux manualmente. Existem algumas formas de se fazer isso. O [Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Makefile) raíz do kernel Linux fornece um conjunto de alvos que possibilita que você os configure. Por exemplo, `menuconfig` fornece uma interface baseada em menus para a configuração do kernel:

![menuconfig](http://s21.postimg.org/zcz48p7yf/menucnonfig.png)

O argumento `defconfig` gera o arquivo de configuração padrão para a arquitetura atual, por exemplo [x86_64 defconfig](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/configs/x86_64_defconfig). Você pode passar o argumento via linha de commando `ARCH` para o `make` para compilar o `defconfig` para a dada arquitetura:

```
$ make ARCH=arm64 defconfig
```

Os argumentos `allnoconfig`, `allyesconfig` e `allmodconfig` possibilitam que você gere um novo arquivo de configuração onde todas as opções serão desabilitadas, habilitadas, e habilitadas como módulos, respectivamente. O argumento de linha de comando `nconfig` que provê  o programa `ncurses` com o menu para configurar o kernel Linux:

![nconfig](http://s29.postimg.org/hpghikp4n/nconfig.png)

E até mesmo o `randconfig` para gerar um arquivo de configuração do kernel Linux aleatório.  Eu não irei escrever sobre como configurar o kernel Linux ou quais opções devem ser habilitadas pois isso não faz sentido de ser feito por dois motivos: Primeiro de tudo eu não sei o seu hardware, e segundo, se você conhece o seu hardware, a única tarefa restante é encontrar como utilizar os programas para a configuração do kernel, e todos eles são bastante simples de utilizar.

OK, agora nós temos o código fonte do kernel Linux configurado. O próximo passo é a compilação do kernel Linux. A forma mais simples de compilar o kernel Linux é apenas executando:

```
$ make
scripts/kconfig/conf  --silentoldconfig Kconfig
#
# configuration written to .config
#
  CHK     include/config/kernel.release
  UPD     include/config/kernel.release
  CHK     include/generated/uapi/linux/version.h
  CHK     include/generated/utsrelease.h
  ...
  ...
  ...
  OBJCOPY arch/x86/boot/vmlinux.bin
  AS      arch/x86/boot/header.o
  LD      arch/x86/boot/setup.elf
  OBJCOPY arch/x86/boot/setup.bin
  BUILD   arch/x86/boot/bzImage
  Setup is 15740 bytes (padded to 15872 bytes).
System is 4342 kB
CRC 82703414
Kernel: arch/x86/boot/bzImage is ready  (#73)
```

Para aumentar a velocidade em que o kernel é compilado, você pode adicionar o argumento de linha de comando `-jN` ao `make`, onde `N` especifica o número de comandos para executar simultaneamente (porém isso é diretamente limitado pelo número de cores disponível em seu computador):

```
$ make -j8
```

Se você quer compilar o kernel Linux para uma arquitetura que é diferente da sua atual, a forma mais simples de fazer isso é passando dois argumentos junto da linha de comando:

* `ARCH` e o nome da arquitetura alvo;
* `CROSS_COMPILER` e o prefixo da ferramenta de croscompilação.

Por exemplo, se nós quisermos compilar o kernel Linux para a arquitetura [arm64](https://en.wikipedia.org/wiki/ARM_architecture#AArch64_features) com o arquivo de configuração do kernel padrão, nós precisamos executar os seguintes comandos:

```
$ make -j4 ARCH=arm64 CROSS_COMPILER=aarch64-linux-gnu- defconfig
$ make -j4 ARCH=arm64 CROSS_COMPILER=aarch64-linux-gnu-
```

Como resultado da compilação, nós podemos ver o a imagem comprimida do kernel - `arch/x86/boot/bzImage`. Agora que nós temos o kernel compilado, nós podemos tanto instalá-lo em nosso computador ou apenas executá-lo em um emulador.

Instalando o kernel Linux
--------------------------------------------------------------------------------

Como eu já havia escrito, nós vamos considerar duas formas de como executar um novo kernel: No primeiro caso nós vamos instalar e executar a nova versão do kernel Linux em um hardware real, e a segunda forma é executar o kernel Linux em uma máquina virtual. No parágrafo anterior nós vimos como compilar o kernel Linux a partir do código fonte e como resultado, nós obtivemos a imagem comprimida:

```
...
...
...
Kernel: arch/x86/boot/bzImage is ready  (#73)
```

Após obtermos a imagem comprimida [bzImage](https://en.wikipedia.org/wiki/Vmlinux#bzImage) nós precisamos instalar os `cabeçalhos` (`headers` do inglês), e os `módulos` (`modules` do inglês) do novo kernel Linux com os seguintes comandos:

```
$ sudo make headers_install
$ sudo make modules_install
```

e a instalação do próprio kernel:

```
$ sudo make install
```

A partir deste momento nós instalamos uma nova versão do kernel Linux e agora nós precisamos informar o `bootloader` sobre isso. É claro que nós podemos adicionar esta informação manualmente editando o arquivo de configuração `/boot/grub2/grub.cfg`, mas eu prefiro utilizar um script para este propósito. Eu estou utilizando duas distribuições Linux: Fedora e Ubuntu. Existem duas formas diferentes de atualizar o arquivo de configuração do [grub](https://en.wikipedia.org/wiki/GNU_GRUB). Eu estou utilizando o script a seguir para este propósito:

```shell
#!/bin/bash

source "term-colors"

DISTRIBUTIVE=$(cat /etc/*-release | grep NAME | head -1 | sed -n -e 's/NAME\=//p')
echo -e "Distributive: ${Green}${DISTRIBUTIVE}${Color_Off}"

if [[ "$DISTRIBUTIVE" == "Fedora" ]] ;
then
    su -c 'grub2-mkconfig -o /boot/grub2/grub.cfg'
else
    sudo update-grub
fi

echo "${Green}Done.${Color_Off}"
```

Este é o último passo para a nova instalação do kernel Linux e após isso você pode reiniciar seu computador e selecionar a nova versão do kernel durante o processo de boot.

O segundo caso é executar o kernel Linux em uma máquina virtual. Eu prefiro utilizar o [qemu](https://en.wikipedia.org/wiki/QEMU). Primeiramente, nós precisamos compilar o `initial ramdisk` - [initrd](https://en.wikipedia.org/wiki/Initrd) para isso. O `initrd` é um sistema de arquivos raíz temporário que é utilizado pelo kernel Linux durante o processo de initialização enquanto outros sistemas de arquivos não são montados. Nós podemos compilar o `initrd` com os seguintes comandos:

Antes de tudo, nós precisamos baixar o [busybox](https://en.wikipedia.org/wiki/BusyBox) e executar o `menuconfig` para configurá-lo:

```shell
$ mkdir initrd
$ cd initrd
$ curl http://busybox.net/downloads/busybox-1.23.2.tar.bz2 | tar xjf -
$ cd busybox-1.23.2/
$ make menuconfig
$ make -j4
```

O `busybox` é um arquivo executável - `/bin/busybox` que contém um conjunto de ferramentas padrões como o [coreutils](https://en.wikipedia.org/wiki/GNU_Core_Utilities). No menu do `busysbox` nós precisamos habilitar a opção `Build BusyBox as a static binary (no shared libs)`:

![busysbox menu](http://s18.postimg.org/sj92uoweh/busybox.png)

Nós podemos encontrar este menu em:

```
Busybox Settings
--> Build Options
```

Após isso, nós podemos sair do menu de configuração do `busybox` e executar os seguintes comandos para a compilação e instalação dele:

```
$ make -j4
$ sudo make install
```

Agora que o `busybox` está instalado, nós podemos começar a compilar o nosso `initrd`. Para fazer isso, vamos até o diretório anterior `initrd` e:

```
$ cd ..
$ mkdir -p initramfs
$ cd initramfs
$ mkdir -pv {bin,sbin,etc,proc,sys,usr/{bin,sbin}}
$ cp -av ../busybox-1.23.2/_install/* .
```

copiamos os arquivos gerados do `busybox` para os diretórios `bin`, `sbin` e outros listados no comando. Agora nós precisamos criar o arquivo executável `init` que será executado como o primeiro processo no sistema. O meu arquivo `init` apenas monta o sistema de arquivos [procfs](https://en.wikipedia.org/wiki/Procfs) e [sysfs](https://en.wikipedia.org/wiki/Sysfs) e então executa o shell:

```shell
#!/bin/sh

mount -t proc none /proc
mount -t sysfs none /sys

exec /bin/sh
```

Agora nós podemos criar um arquivo que será o nosso `initrd`:

```
$ find . -print0 | cpio --null -ov --format=newc | gzip -9 > ~/dev/initrd_x86_64.gz
```

We can now run our kernel in the virtual machine. As I already wrote I prefer [qemu](https://en.wikipedia.org/wiki/QEMU) for this. We can run our kernel with the following command:

```
$ qemu-system-x86_64 -snapshot -m 8GB -serial stdio -kernel ~/dev/linux/arch/x86_64/boot/bzImage -initrd ~/dev/initrd_x86_64.gz -append "root=/dev/sda1 ignore_loglevel"
```

![qemu](http://s22.postimg.org/b8ttyigup/qemu.png)

From now we can run the Linux kernel in the virtual machine and this means that we can begin to change and test the kernel.

Consider using [ivandaviov/minimal](https://github.com/ivandavidov/minimal) or [Buildroot](https://buildroot.org/) to automate the process of generating initrd.

Getting started with the Linux Kernel Development
---------------------------------------------------------------------------------

The main point of this paragraph is to answer two questions: What to do and what not to do before sending your first patch to the Linux kernel. Please, do not confuse this `to do` with `todo`. I have no answer what you can fix in the Linux kernel. I just want to tell you my workflow during experimenting with the Linux kernel source code.

First of all I pull the latest updates from Linus's repo with the following commands:

```
$ git checkout master
$ git pull upstream master
```

After this my local repository with the Linux kernel source code is synced with the [mainline](https://github.com/torvalds/linux) repository. Now we can make some changes in the source code. As I already wrote, I have no advice for you where you can start and what `TODO` in the Linux kernel. But the best place for newbies is `staging` tree. In other words the set of drivers from the [drivers/staging](https://github.com/torvalds/linux/tree/master/drivers/staging). The maintainer of the `staging` tree is [Greg Kroah-Hartman](https://en.wikipedia.org/wiki/Greg_Kroah-Hartman) and the `staging` tree is that place where your trivial patch can be accepted. Let's look on a simple example that describes how to generate patch, check it and send to the [Linux kernel mail listing](https://lkml.org/).

If we look in the driver for the [Digi International EPCA PCI](https://github.com/torvalds/linux/tree/master/drivers/staging/dgap) based devices, we will see the `dgap_sindex` function on line 295:

```C
static char *dgap_sindex(char *string, char *group)
{
	char *ptr;

	if (!string || !group)
		return NULL;

	for (; *string; string++) {
		for (ptr = group; *ptr; ptr++) {
			if (*ptr == *string)
				return string;
		}
	}

	return NULL;
}
```

This function looks for a match of any character in the group and returns that position. During research of source code of the Linux kernel, I have noted that the [lib/string.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/lib/string.c#L473) source code file contains the implementation of the `strpbrk` function that does the same thing as `dgap_sinidex`. It is not a good idea to use a custom implementation of a function that already exists, so we can remove the `dgap_sindex` function from the [drivers/staging/dgap/dgap.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/staging/dgap/dgap.c) source code file and use the `strpbrk` instead.

First of all let's create new `git` branch based on the current master that synced with the Linux kernel mainline repo:

```
$ git checkout -b "dgap-remove-dgap_sindex"
```

And now we can replace the `dgap_sindex` with the `strpbrk`. After we did all changes we need to recompile the Linux kernel or just [dgap](https://github.com/torvalds/linux/tree/master/drivers/staging/dgap) directory. Do not forget to enable this driver in the kernel configuration. You can find it in the:

```
Device Drivers
--> Staging drivers
----> Digi EPCA PCI products
```

![dgap menu](http://s4.postimg.org/d3pozpge5/digi.png)

Now is time to make commit. I'm using following combination for this:

```
$ git add .
$ git commit -s -v
```

After the last command an editor will be opened that will be chosen from `$GIT_EDITOR` or `$EDITOR` environment variable. The `-s` command line argument will add `Signed-off-by` line by the committer at the end of the commit log message. You can find this line in the end of each commit message, for example - [00cc1633](https://github.com/torvalds/linux/commit/00cc1633816de8c95f337608a1ea64e228faf771). The main point of this line is the tracking of who did a change. The `-v` option show unified diff between the HEAD commit and what would be committed at the bottom of the commit message. It is not necessary, but very useful sometimes. A couple of words about commit message. Actually a commit message consists from two parts:

The first part is on the first line and contains short description of changes. It starts from the `[PATCH]` prefix followed by a subsystem, driver or architecture name and after `:` symbol short description. In our case it will be something like this:

```
[PATCH] staging/dgap: Use strpbrk() instead of dgap_sindex()
```

After short description usually we have an empty line and full description of the commit. In our case it will be:

```
The <linux/string.h> provides strpbrk() function that does the same that the
dgap_sindex(). Let's use already defined function instead of writing custom.
```

And the `Sign-off-by` line in the end of the commit message. Note that each line of a commit message must no be longer than `80` symbols and commit message must describe your changes in details. Do not just write a commit message like: `Custom function removed`, you need to describe what you did and why. The patch reviewers must know what they review. Besides this commit messages in this view are very helpful. Each time when we can't understand something, we can use [git blame](http://git-scm.com/docs/git-blame) to read description of changes.

After we have committed changes time to generate patch. We can do it with the `format-patch` command:

```
$ git format-patch master
0001-staging-dgap-Use-strpbrk-instead-of-dgap_sindex.patch
```

We've passed name of the branch (`master` in this case) to the `format-patch` command that will generate a patch with the last changes that are in the `dgap-remove-dgap_sindex` branch and not are in the `master` branch. As you can note, the `format-patch` command generates file that contains last changes and has name that is based on the commit short description. If you want to generate a patch with the custom name, you can use `--stdout` option:

```
$ git format-patch master --stdout > dgap-patch-1.patch
```

The last step after we have generated our patch is to send it to the Linux kernel mailing list. Of course, you can use any email client, `git` provides a special command for this: `git send-email`. Before you send your patch, you need to know where to send it. Yes, you can just send it to the Linux kernel mailing list address which is `linux-kernel@vger.kernel.org`, but it is very likely that the patch will be ignored, because of the large flow of messages. The better choice would be to send the patch to the maintainers of the subsystem where you have made changes. To find the names of these maintainers use the `get_maintainer.pl` script. All you need to do is pass the file or directory where you wrote code.

```
$ ./scripts/get_maintainer.pl -f drivers/staging/dgap/dgap.c
Lidza Louina <lidza.louina@gmail.com> (maintainer:DIGI EPCA PCI PRODUCTS)
Mark Hounschell <markh@compro.net> (maintainer:DIGI EPCA PCI PRODUCTS)
Daeseok Youn <daeseok.youn@gmail.com> (maintainer:DIGI EPCA PCI PRODUCTS)
Greg Kroah-Hartman <gregkh@linuxfoundation.org> (supporter:STAGING SUBSYSTEM)
driverdev-devel@linuxdriverproject.org (open list:DIGI EPCA PCI PRODUCTS)
devel@driverdev.osuosl.org (open list:STAGING SUBSYSTEM)
linux-kernel@vger.kernel.org (open list)
```

You will see the set of the names and related emails. Now we can send our patch with:

```
$ git send-email --to "Lidza Louina <lidza.louina@gmail.com>" \
  --cc "Mark Hounschell <markh@compro.net>"                   \
  --cc "Daeseok Youn <daeseok.youn@gmail.com>"                \
  --cc "Greg Kroah-Hartman <gregkh@linuxfoundation.org>"      \
  --cc "driverdev-devel@linuxdriverproject.org"               \
  --cc "devel@driverdev.osuosl.org"                           \
  --cc "linux-kernel@vger.kernel.org"
```

That's all. The patch is sent and now you only have to wait for feedback from the Linux kernel developers. After you send a patch and a maintainer accepts it, you will find it in the maintainer's repository (for example [patch](https://git.kernel.org/cgit/linux/kernel/git/gregkh/staging.git/commit/?h=staging-testing&id=b9f7f1d0846f15585b8af64435b6b706b25a5c0b) that you saw in this part) and after some time the maintainer will send a pull request to Linus and you will see your patch in the mainline repository.

That's all.

Some advice
--------------------------------------------------------------------------------

In the end of this part I want to give you some advice that will describe what to do and what not to do during development of the Linux kernel:

* Think, Think, Think. And think again before you decide to send a patch.

* Each time when you have changed something in the Linux kernel source code - compile it. After any changes. Again and again. Nobody likes changes that don't even compile.

* The Linux kernel has a coding style [guide](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/CodingStyle) and you need to comply with it. There is great script which can help to check your changes. This script is - [scripts/checkpatch.pl](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/scripts/checkpatch.pl). Just pass source code file with changes to it and you will see:

```
$ ./scripts/checkpatch.pl -f drivers/staging/dgap/dgap.c
WARNING: Block comments use * on subsequent lines
#94: FILE: drivers/staging/dgap/dgap.c:94:
+/*
+     SUPPORTED PRODUCTS

CHECK: spaces preferred around that '|' (ctx:VxV)
#143: FILE: drivers/staging/dgap/dgap.c:143:
+	{ PPCM,        PCI_DEV_XEM_NAME,     64, (T_PCXM|T_PCLITE|T_PCIBUS) },

```

Also you can see problematic places with the help of the `git diff`:

![git diff](http://oi60.tinypic.com/2u91rgn.jpg)

* [Linus doesn't accept github pull requests](https://github.com/torvalds/linux/pull/17#issuecomment-5654674)

* If your change consists from some different and unrelated changes, you need to split the changes via separate commits. The `git format-patch` command will generate patches for each commit and the subject of each patch will contain a `vN` prefix where the `N` is the number of the patch. If you are planning to send a series of patches it will be helpful to pass the `--cover-letter` option to the `git format-patch` command. This will generate an additional file that will contain the cover letter that you can use to describe what your patchset changes. It is also a good idea to use the `--in-reply-to` option in the `git send-email` command. This option allows you to send your patch series in reply to your cover message. The structure of the your patch will look like this for a maintainer:

```
|--> cover letter
  |----> patch_1
  |----> patch_2
```

You need to pass `message-id` as an argument of the `--in-reply-to` option that you can find in the output of the `git send-email`:

It's important that your email be in the [plain text](https://en.wikipedia.org/wiki/Plain_text) format. Generally, `send-email` and `format-patch` are very useful during development, so look at the documentation for the commands and you'll find some useful options such as: [git send-email](http://git-scm.com/docs/git-send-email) and [git format-patch](http://git-scm.com/docs/git-format-patch).

* Do not be surprised if you do not get an immediate answer after you send your patch. Maintainers can be very busy.

* The [scripts](https://github.com/torvalds/linux/tree/master/scripts) directory contains many different useful scripts that are related to Linux kernel development. We already saw two scripts from this directory: the `checkpatch.pl` and the `get_maintainer.pl` scripts. Outside of those scripts, you can find the [stackusage](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/scripts/stackusage) script that will print usage of the stack, [extract-vmlinux](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/scripts/extract-vmlinux) for extracting an uncompressed kernel image, and many others. Outside of the `scripts` directory you can find some very useful [scripts](https://github.com/lorenzo-stoakes/kernel-scripts) by [Lorenzo Stoakes](https://twitter.com/ljsloz) for kernel development.

* Subscribe to the Linux kernel mailing list. There are a large number of letters every day on `lkml`, but it is very useful to read them and understand things such as the current state of the Linux kernel. Other than `lkml` there are [set](http://vger.kernel.org/vger-lists.html) mailing listings which are related to the different Linux kernel subsystems.

* If your patch is not accepted the first time and you receive feedback from Linux kernel developers, make your changes and resend the patch with the `[PATCH vN]` prefix (where `N` is the number of patch version). For example:

```
[PATCH v2] staging/dgap: Use strpbrk() instead of dgap_sindex()
```

Also it must contain a changelog that describes all changes from previous patch versions. Of course, this is not an exhaustive list of requirements for Linux kernel development, but some of the most important items were addressed.

Happy Hacking!

Conclusion
--------------------------------------------------------------------------------

I hope this will help others join the Linux kernel community!
If you have any questions or suggestions, write me at [email](kuleshovmail@gmail.com) or ping [me](https://twitter.com/0xAX) on twitter.

Please note that English is not my first language, and I am really sorry for any inconvenience. If you find any mistakes please let me know via email or send a PR.

Links
--------------------------------------------------------------------------------

* [blog posts about assembly programming for x86_64](http://0xax.github.io/categories/assembly/)
* [Assembler](https://en.wikipedia.org/wiki/Assembly_language#Assembler)
* [distro](https://en.wikipedia.org/wiki/Linux_distribution)
* [package manager](https://en.wikipedia.org/wiki/Package_manager)
* [grub](https://en.wikipedia.org/wiki/GNU_GRUB)
* [kernel.org](https://kernel.org/)
* [version control system](https://en.wikipedia.org/wiki/Version_control)
* [arm64](https://en.wikipedia.org/wiki/ARM_architecture#AArch64_features)
* [bzImage](https://en.wikipedia.org/wiki/Vmlinux#bzImage)
* [qemu](https://en.wikipedia.org/wiki/QEMU)
* [initrd](https://en.wikipedia.org/wiki/Initrd)
* [busybox](https://en.wikipedia.org/wiki/BusyBox)
* [coreutils](https://en.wikipedia.org/wiki/GNU_Core_Utilities)
* [procfs](https://en.wikipedia.org/wiki/Procfs)
* [sysfs](https://en.wikipedia.org/wiki/Sysfs)
* [Linux kernel mail listing archive](https://lkml.org/)
* [Linux kernel coding style guide](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/CodingStyle)
* [How to Get Your Change Into the Linux Kernel](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/SubmittingPatches)
* [Linux Kernel Newbies](http://kernelnewbies.org/)
* [plain text](https://en.wikipedia.org/wiki/Plain_text)
