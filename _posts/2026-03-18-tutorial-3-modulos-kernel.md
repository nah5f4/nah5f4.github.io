---

title: "Tutorial 3 - Configurações de Build e Módulos do Kernel"
date: 2026-03-18
categories: [Linux, Kernel, Virtualização]
tags: [kernel, modules, linux, build, kworkflow]
---

# Tutorial 3 - Configurações de Build e Módulos do Kernel

No terceiro tutorial, o objetivo foi entender como criar um módulo simples, configurar o sistema de build do kernel para reconhecê-lo e lidar com as dependências entre diferentes módulos.

---

## 1) Criação de um Módulo de Exemplo Simples

A primeira tarefa foi criar o código-fonte de um módulo básico em C que apenas imprime "Hello world" ao ser carregado. Criei o arquivo `simple_mod.c` dentro de `drivers/misc/` na raiz do kernel. Para o kernel saber que meu módulo existe, precisei editar os arquivos de configuração do diretório. Adicionei o símbolo `SIMPLE_MOD` no `Kconfig` e a regra de compilação no `Makefile`.

---

## 2) Configuração e Compilação

Com os arquivos criados, usei o `menuconfig` para ativar o módulo. Um detalhe importante: selecionei a opção como `< M >` (módulo), para podermos carregar e descarregar dinamicamente.

---

## 3) Instalação dos Módulos

Essa foi a parte que mais me deu trabalho. Tentei usar o `guestmount` para instalar os módulos no rootfs da VM sem precisar ligá-la, mas enfrentei erros persistentes do `libguestfs`:

```text 
appliance closed the connection unexpectedly
```

Tive que investigar bastante, instalando pacotes extras e testando diferentes configurações de backend:

```bash 
sudo apt install libguestfs-tools qemu-utils supermin -y
export LIBGUESTFS_BACKEND=direct
# Forçando o uso de TCG caso o KVM desse problema
export LIBGUESTFS_BACKEND_SETTINGS=force_tcg
```

Mesmo assim, o `guestmount` continuava falhando em alguns momentos porque a VM ainda estava "segurando" o arquivo da imagem. Precisei garantir que a VM estivesse desligada com:

```bash 
virsh destroy arm64
```

antes de tentar montar.

No fim, a solução que funcionou para garantir a instalação foi:

```bash 
sudo -E guestmount --rw -a "${VM_DIR}/arm64_img.qcow2" -m /dev/sda2 "${VM_DIR}/arm64_rootfs"
sudo --preserve-env make -C "${IIO_TREE}" INSTALL_MOD_PATH="${VM_DIR}/arm64_rootfs" modules_install
sudo guestunmount "${VM_DIR}/arm64_rootfs"
```

---

## 4) Testando na VM

Após a dificuldade anterior, liguei a VM e testei o carregamento do módulo. Usei o `insmod` passando o caminho completo do arquivo `.ko` gerado e o `dmesg` para ver a mágica acontecendo:

```bash 
insmod /lib/modules/$(uname -r)/kernel/drivers/misc/simple_mod.ko
dmesg | tail
```

Ver o "Hello world" no log do kernel foi muito bom. Também testei o `rmmod` para remover o módulo e vi o "Goodbye world" aparecer.

---

## Conclusão

Nesta etapa, ficou claro que trabalhar com kernel é um exercício de paciência com as ferramentas. Tive problemas com o `rsync` faltando na VM (que quebrava o `kw ssh --get`), mudei o IP da VM no meio do caminho e precisei atualizar o `kw remote`.

O ambiente agora está bem configurado.

