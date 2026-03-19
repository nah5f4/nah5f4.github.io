---

title: "Tutorial 1 - Configurando o Ambiente de Desenvolvimento"
date: 2026-03-18 10:00:00 -0300
categories: [Linux, Kernel, Virtualização]
tags: [qemu, libvirt, kernel, vm, linux]
---

# Tutorial 1 - Configurando o Ambiente de Desenvolvimento

Neste primeiro tutorial da disciplina, a proposta foi montar todo o ambiente de desenvolvimento para trabalhar com o kernel Linux utilizando QEMU e libvirt. A ideia é ter uma máquina virtual (VM) isolada para testar modificações no kernel sem comprometer o sistema principal.

O tutorial está bem estruturado, mas tive apenas uma dificuldade no processo: o link para o download da imagem do kernel estava quebrado, e contei com o auxílio do monitor para conseguir um link funcional.

---

## 1) Preparando o Diretório e o Script de Ativação

A primeira tarefa foi preparar o diretório, ajustando as permissões para que o usuário e o libvirt pudessem trabalhar juntos. Utilizei os seguintes comandos:

```bash
sudo mkdir /home/lk_dev
sudo chown -R libvirt-qemu:libvirt-qemu /home/lk_dev
sudo chmod -R 2770 /home/lk_dev
sudo usermod -aG libvirt-qemu "$USER"
```

Tive que fazer logout e login novamente para as mudanças entrarem em vigor. Logo em seguida, criei um script para automatizar as tarefas repetitivas. O uso do script activate.sh foi excelente: ele automatiza as variáveis de ambiente e muda o preâmbulo do terminal para (LK-DEV), o que ajuda muito a não rodar comandos no lugar errado. Com o activate.sh rodando, segui para o próximo passo.

---

## 2) Configuração da VM e Manipulação de Imagem

Criei um diretório para os artefatos da VM no interior de lk_dev e adicionei o caminho à variável VM_DIR no activate.sh. A única parte onde tive um problema foi no download da Imagem Debian, onde o link estava quebrado. Após o auxílio do monitor, baixei e renomeei o arquivo usando os seguintes comandos: 

```bash
wget --directory-prefix="${VM_DIR}" http://cdimage.debian.org/cdimage/cloud/bookworm/daily/20260225-2399/debian-12-nocloud-arm64-daily-20260225-2399.qcow2
mv "${VM_DIR}/debian-12-nocloud-arm64-daily-20260225-2399.qcow2" "${VM_DIR}/base_arm64_img.qcow2" 
```

Como a imagem original era muito pequena para suportar compilações do kernel, foi necessário redimensioná-la:

```bash
qemu-img create -f qcow2 -o preallocation=metadata "${VM_DIR}/arm64_img.qcow2" 5G
```

```bash
virt-resize --expand /dev/sda1 "${VM_DIR}/base_arm64_img.qcow2" "${VM_DIR}/arm64_img.qcow2"
```

---

Também extraí o kernel e o initrd da imagem para uso no boot do QEMU:

```bash
virt-copy-out --add "${VM_DIR}/arm64_img.qcow2" /boot/vmlinuz-6.1.0-43-arm64 "$BOOT_DIR"
```

```bash
virt-copy-out --add "${VM_DIR}/arm64_img.qcow2" /boot/initrd.img-6.1.0-43-arm64 "$BOOT_DIR"
```

Essa etapa foi feita após criar um diretório para os artefatos de inicialização e definir a variável `BOOT_DIR` no `activate.sh`.

O uso de ferramentas como virt-resize e virt-copy-out facilita muito a vida, pois permite manipular o conteúdo do disco virtual sem precisar ligar a VM. Com tudo pronto, adicionei a função launch_vm_qemu ao meu activate.sh. Essa função é um comando do QEMU que define a arquitetura (ARM64), a memória, o processador e os caminhos para o kernel e a imagem de disco.

Para testar:

```bash
launch_vm_qemu
```

---

Para um melhor controle, o tutorial introduz o libvirt. Ele permite tratar a VM como um serviço do sistema. Primeiro, precisei garantir que o daemon estivesse rodando:

```bash
sudo systemctl start libvirtd
sudo virsh net-start default
```

Depois, configurei a função create_vm_virsh no script de ativação. A vantagem aqui é que o libvirt cria uma rede virtual que facilita muito a comunicação entre o meu notebook e a VM.

---

## 3) Configuração de Acesso SSH

Por fim, configurei o acesso SSH para facilitar a interação com a VM.

Dentro da VM, editei o arquivo:

```
/etc/ssh/sshd_config
```

Permitindo:

* login como root
* uso de senhas vazias (apenas para ambiente de testes)

Após reiniciar o serviço SSH, utilizei o seguinte comando no host para descobrir o IP da VM:

```bash
sudo virsh net-dhcp-leases default
```

E então conectei via SSH:

```bash
ssh root@192.168.122.38
```

---

## Conclusão

No geral, a experiência foi muito enriquecedora para compreender melhor o processo de virtualização e manipulação de ambientes isolados para desenvolvimento de kernel.

