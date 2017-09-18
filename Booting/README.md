# Kernel Boot Process

Este capítulo descreve o processo de boot do kernel Linux. Aqui você verá alguns posts que descrevem o ciclo completo do processo de boot do kernel::

* [Do bootloader ao kernel](linux-bootstrap-1.md) - describes all stages from turning on the computer to running the first instruction of the kernel.
* [Primeiros passos no código de configuração do kernel](linux-bootstrap-2.md) - describes first steps in the kernel setup code. You will see heap initialization, query of different parameters like EDD, IST and etc...
* [Inicialização do modo de vídeo e transição para o modo protegido](linux-bootstrap-3.md) - describes video mode initialization in the kernel setup code and transition to protected mode.
* [Transição para o modo de 64-bit](linux-bootstrap-4.md) - describes preparation for transition into 64-bit mode and details of transition.
* [Kernel Decompression](linux-bootstrap-5.md) - describes preparation before kernel decompression and details of direct decompression.
