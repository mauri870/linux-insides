# Kernel e o processo de Boot

Este capítulo aborda o processo de boot do kernel linux. Nele você verá alguns posts que descrevem o ciclo completo do carregamento do kernel:

* [Do bootloader ao Kernel](linux-bootstrap-1.md) - descreve todos os estágios desde que o computador é ligado até a execução da primeira instrução do kernel.
* [Primeiros passos no código de inicialização do kernel](linux-bootstrap-2.md) - descreve as primeiras etapas do código de inicialização do kernel. Você verá a inicialização do heap, consulta a diferentes parâmetros como EDD, IST e etc...
* [Inicialização do modo de vídeo e transição para o modo protegido](linux-bootstrap-3.md) - descreve o modo de inicialização de video no código inicial do kernel e a transição para o modo protegido.
* [Transição para o modo 64 bits](linux-bootstrap-4.md) - descreve a preparação e detalhes  da transição ao modo 64 bits.
* [Descompressão do kernel](linux-bootstrap-5.md) - descreve a preparação antes da descompressão do kernel e detalhes sobre a descompressão em si.
