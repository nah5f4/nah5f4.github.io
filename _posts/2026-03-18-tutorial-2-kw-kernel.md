---

title: "Tutorial 2 - kw e a Compilação do Kernel"
date: 2026-03-18
categories: [Linux, Kernel, Virtualização]
tags: [kernel, kw, kworkflow, linux, build]
---

# Tutorial 2 - kw e a Compilação do Kernel

Dando continuidade ao ambiente preparado no primeiro tutorial, o objetivo aqui foi configurar, compilar e rodar uma versão customizada dentro da VM ARM64. Para facilitar o processo, o tutorial introduz o **kworkflow (kw)**, uma ferramenta que automatiza as tarefas repetitivas de build e deploy.

---

## 1) Instalação do kw

A primeira tarefa foi instalar o kw. Adicionei a variável `KW_DIR` no meu `activate.sh`, clonei o repositório e rodei o script de instalação completa:

```bash
./setup.sh --full-installation
```

A instalação foi tranquila e pude validar rodando:

```bash
kw --version
```

---

## 2) Clonando a árvore do Kernel Linux

Em seguida, precisei baixar o código do kernel. Como o kernel é gigantesco, fizemos um "shallow clone" (apenas os últimos 10 commits) da branch de testes do subsistema IIO. Adicionei a variável `IIO_TREE` ao `activate.sh` e usei o comando:

```bash
git clone git://git.kernel.org/pub/scm/linux/kernel/git/jic23/iio.git "${IIO_TREE}" --branch testing --single-branch --depth 10
```

---

## 3) Configuração do kw no contexto local para o desenvolvimento do IIO

Com a árvore em mãos, inicializei o kw dentro do diretório com:

```bash
kw init
```

E configurei o acesso remoto para a nossa VM:

```bash
kw remote --add arm64 root@192.168.122.38 --set-default
```

Confirmei se tudo foi adicionado corretamente com o comando:

```bash
kw ssh
```

que iniciou a conexão com a máquina virtual.

---

## 4) Configurando a compilação do kernel linux

Um ponto importante dessa etapa foi a criação do arquivo `.config`, que define exatamente o que será incluído ou não na compilação do kernel. Em vez de criar tudo do zero, utilizei o defconfig base para ARM64 fornecido pelo kernel:

```bash
make ARCH=arm64 defconfig
make ARCH=arm64 olddefconfig
```

Para deixar essa configuração mais otimizada, usei a lista de módulos que a própria VM estava utilizando. A ideia aqui é bem interessante: em vez de compilar tudo, o sistema gera uma configuração baseada apenas nos módulos realmente necessários naquele ambiente.

```bash
kw ssh --get '~/vm_mod_list'
make ARCH=arm64 LSMOD=vm_mod_list localmodconfig
```

Durante essa etapa, encontrei um erro ao tentar copiar o arquivo da VM:

```text
line 1: rsync: command not found
rsync: connection unexpectedly closed
```

O problema foi direto: o comando `kw ssh --get` utiliza o **rsync** internamente para transferir arquivos, e ele não estava instalado na VM. Depois de identificar isso, bastou instalar o rsync para o comando funcionar corretamente.

Para facilitar o gerenciamento das configurações, usei o `kw kernel-config-manager`, que permite salvar e reutilizar diferentes `.config` sem precisar renomear arquivos manualmente. Em vez de editar direto o `.config`, o ideal é usar as interfaces do próprio kernel (como o `menuconfig`).

```bash
kw kernel-config-manager --save arm64-optimized
kw kernel-config-manager --list
kw kernel-config-manager --get arm64-optimized
```

---

## 5) Construindo o Kernel (Build)

Na etapa de compilação, entra uma coisa importante: como meu host é x86_64 e a VM é ARM64, precisei usar cross-compilation. Para isso, instalei o compilador apropriado:

```bash
sudo apt update && sudo apt install gcc-aarch64-linux-gnu
```

Depois, configurei o kw para usar a arquitetura e o compilador corretos:

```bash
kw config build.arch 'arm64'
kw config build.cross_compile 'aarch64-linux-gnu-'
kw config build.kernel_img_name 'Image.gz'
kw config --show build
```

Com tudo configurado, rodei:

```bash
kw build
```

A compilação demorou alguns minutos, mas no final consegui ver os arquivos gerados em `arch/arm64/boot`, o que indicava que o build tinha funcionado corretamente.

---

## 6) Instalando os modulos

Por fim, usei o kw para enviar os módulos compilados para dentro da VM:

```bash
kw deploy --modules
```

Fiz os ajustes necessários no `activate.sh`, desliguei a VM e a recriei no libvirt para garantir que ela usasse o novo kernel:

```bash
sudo virsh shutdown arm64
sudo virsh undefine arm64
create_vm_virsh
```

Para confirmar que estava tudo certo, rodei:

```bash
uname --kernel-release
```

dentro da VM e obtive a saída esperada com a versão que acabamos de construir.

## Conclusão

Este tutorial aprofundou o processo de desenvolvimento no kernel ao mostrar, na prática, como configurar, compilar e implantar uma versão customizada utilizando o kworkflow. Além de entender melhor o fluxo de build com cross-compilation, também ficou claro como ferramentas como o `kw` simplificam tarefas complexas e recorrentes.

Outro ponto importante foi a capacidade de adaptar a configuração do kernel ao ambiente real da VM, tornando o processo mais eficiente e direcionado. No geral, essa etapa consolidou o ambiente preparado anteriormente e estabeleceu uma base sólida para modificações e experimentos futuros no kernel Linux.

