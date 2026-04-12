---

title: "Introdução a Linux Character Device Drivers"
date: 2026-04-12 
categories: [Linux, Kernel]
tags: [linux, driver, c, flusp]
---

# Tutorial 4: Os Character Device Drivers

Neste quarto tutorial, o foco foi entender e implementar um Character Device (CD) no Linux. Como sabemos, no sistema, quase tudo é abstraído como um arquivo. O Character Device é a abstração para dispositivos que trocam dados sequencialmente, geralmente byte a byte ou em pequenos blocos. 

Eles aparecem como arquivos no diretório /dev/ e são acessados através de syscalls clássicas como open, read, write e close.

## 1. Identificação: Major e Minor Numbers

Para que o kernel saiba exatamente qual driver deve gerenciar cada arquivo de dispositivo, ele utiliza o Device ID (dev_t), um inteiro de 32 bits dividido em duas partes:

* Major Number (12 bits): Identifica o driver responsável (ex: um driver específico para gerenciar displays).
* Minor Number (20 bits): Diferencia instâncias do mesmo hardware (ex: display 1 vs display 2) ou modos de operação.

### Comandos de Inspeção e Criação

Você pode verificar os dispositivos registrados e criar os nós de comunicação com os seguintes comandos no terminal:

# Ver os drivers de character devices registrados
$ cat /proc/devices | grep simple_char

# Inspecionar o Major/Minor de um arquivo de dispositivo
$ stat /dev/ttyS0

# Criar manualmente um device node (Exemplo: Major 511, Minor 0)
$ mknod simple_char_node c 511 0

## 2. Segurança de Memória: Kernel vs. User Space

Um dos pontos mais críticos do tutorial é entender por que não podemos acessar ponteiros do user space diretamente no kernel space. Os motivos principais são:

1. Regiões Separadas: Eles residem em espaços de endereçamento distintos.
2. Volatilidade: Devido à paginação, um endereço de usuário pode estar inválido (em swap ou não alocado) no momento em que o kernel tenta acessá-lo.

Para resolver isso, usamos as funções de segurança da biblioteca <linux/uaccess.h>:

// No Driver (Kernel Space)
copy_to_user(destino_user, origem_kernel, n_bytes);   // Usado no READ (Kernel -> User)
copy_from_user(destino_kernel, origem_user, n_bytes); // Usado no WRITE (User -> Kernel)

## 3. A Estrutura cdev e o Ciclo de Vida

A struct cdev é o que une o ID do dispositivo às suas operações (file_operations). O ciclo de vida do driver segue este fluxo:

1. Alocação: cdev_alloc() prepara a estrutura na memória.
2. Configuração: cdev_init() associa a struct às funções que você implementou (open, read, write, etc).
3. Registro: cdev_add() torna o dispositivo operacional e visível para o sistema.
4. Remoção: cdev_del() libera o registro ao descarregar o módulo.

## 4. Integração com o Build System

Para que o driver seja compilado corretamente dentro da árvore do kernel, precisamos configurar o Kconfig e o Makefile:

**Kconfig:**
config SIMPLE_CHAR
       tristate "Simple character driver example"
       default m
       help
         Este símbolo ativa o driver de exemplo para operações básicas de arquivo.

**Makefile:**
obj-$(CONFIG_SIMPLE_CHAR) += simple_char.o

## 5. Testando o Driver no Ambiente

Após carregar o módulo, você pode validar o funcionamento usando comandos simples de redirecionamento ou programas de teste em C:

# Carregando o módulo no kernel
$ modprobe simple_char

# Enviando uma string para o buffer do driver
$ echo "Mensagem de teste para o kernel" > simple_char_node

# Lendo o conteúdo atual do buffer do driver
$ cat simple_char_node

---

Este tutorial foi fundamental para entender a "ponte" entre o espaço de usuário e o kernel. O domínio das syscalls e da segurança de memória é o que diferencia um desenvolvedor de drivers de um programador de aplicações comum.
