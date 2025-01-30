# arch-linux_tutorial

**Olá! Meu nome é Matheus Lima. Desenvolvi este manual de instalação para auxiliar os desenvolvedores a realizar a instalação do sistema Arch Linux de forma manual.**

*Este tutorial não tem intenção alguma de substituir a documentação original do Arch que se encontra em [ArchWiki](https://wiki.archlinux.org/title/Installation_guide), que por sinal é extremamente completa e intuitiva. A finalidade deste documento tem como objetivo auxiliar desenvolvedores que não foram capazes, ou econtraram alguma dificuldade de seguir a documentação original.*
#### Observação: Toda e qualquer ajuda é bem vinda! Caso encontre algum erro, carência de informação ou comandos desnecessários. Fique à vontade para corrigir os erros e entrar em contato para compartilhar o conhecimento com este humilde programador.

# Estrutura do sistema

#### **Informações de Hardware da máquina utilizada:**
- Processador: Intel Core i7 (13ª geração)
- Arquitetura: 64-bits
- Memória RAM: 16GB
- Armazenamento: SSD NVMe de 500GB
- Placa de Vídeo: Intel Iris Xe Graphics

#### **Estrutura:**
```
**nvme0n1**
├─ nvme0n1p1 → EFI (550 MiB)
├─ nvme0n1p2 → /boot (2 GiB)
└─ nvme0n1p3 → LUKS (restante)
    └─ cryptlvm → LVM
        ├─ swap → 16 GiB
        ├─ root → 50 GiB
        ├─ home → 200 GiB
        └─ lab → Restante
```
#### Observação: Irei utilizar uma criptografia de disco neste tutorial. Caso não deseje criptografar seu HD ou SSD, deverá pular algumas partes deste tutorial.

#### **Criptografia utilizada:**
- LUKS

# Conexão a rede (Wi-fi)

Você tem duas opções aqui; plugar o cabo de rede e ser feliz, ou desafiar o senso comum e conectar-se manualmente como irei ensinar aqui. *Eu prefiro viver o desafio :)*

```
	Comando 'help' para listagem de outros comandos.

iwctl | # Esse comando basicamente é uma ferramenta de linha de comando que iremos utilizar para gerenciar nossas conexões sem fio (Wi-Fi).

iwctl device list | # Caso você não saiba o nome da sua rede doméstica ou rede que usará para a instalação, você pode utilizar esse comando para consultar as redes disponíveis a você no momento.

station wlan0 connect "nome-da-rede" | # Comando para fazer conexão a rede domestica. Após esse comando, será solicitado a passfrase da rede, só digitar a senha e conectar.

Ctrl + C | # Para sair.

dhcpcd | # Obtém IP via DHCP.

ping archlinux.org | # Verifique a conexão. Ctrl + C para parar o loop.
```

# Atualização do relógio

```
timedatectl set-ntp true | # Ativa a sincronização automática do horário do sistema via NTP (Network Time Protocol).

timedatectl status | # Confirma a configuração.
```

# Definição do layout e fonte do teclado do console

Por padrão o mapa de teclas do console é US, então se não quiser utilizar um teclado com configurações americana, você deve seguir esta parte do tutorial.

```
localectl list-keymaps | # Listagem de layouts disponíveis. Só procurar o layout de sua escolha, anotar o nome e substituir a seguir.

loadkeys "Nome do layout" | # Comando utilizado para definir o layout do teclado.
```

# Verificando o modo de inicialização

Essa parte não é necessariamente essencial, mas sim para se ter uma noção da arquitetura do seu sistema.

```
cat /sys/firmware/efi/fw_platform_size | # Comando utilizado para verificar o modo de inicialização. Se retornar "64", então o sistema é inicializado no modo UEFI de 64 bits. Se retornar 32, então o UEFI é de 32 bits.
```

# Criação das partições

Agora vamos começar de verdade! Esta parte é importante você saber de uma coisa. Minha máquina possui um SSD nvme, então caso você possua um HD por exemplo, você deverá alterar partes deste código. A arquitetura deverá ser manupulada à sua vontade também.

```
fdisk -l | # Comando para listar os dispositivos.

gdisk /dev/nvme0n1 | # Comando utilizado para gerenciar as partições do disco.

o          # Criar nova tabela GPT
n          # Nova partição (EFI)
1          # Número
Enter      # Primeiro setor (padrão)
+550M      # Tamanho
ef00       # Tipo EFI

n          # Nova partição (/boot)
2          # Número
Enter      # Primeiro setor (padrão)
+2G        # Tamanho
8300       # Tipo Linux

n          # Nova partição (LUKS)
3          # Número
Enter      # Primeiro setor (padrão)
Enter      # Usar espaço restante
8309       # Tipo Linux LUKS

w          # Salvar e sair
```

Configuração de criptografia LUKS

```
cryptsetup luksFormat --type luks2 --hash sha512 --iter-time 5000 /dev/nvme0n1p3 | # Comando para criptografar partição (--iter-time aumenta a segurança contra ataques de força bruta).

cryptsetup open /dev/nvme0n1p3 cryptlvm | # Abrir container LUKS "cryptlvm" é o nome do mapeamento.
```

Configurando o LVM

```
pvcreate /dev/mapper/cryptlvm | # Cria volume físico LVM.
vgcreate vg0 /dev/mapper/cryptlvm | # Cria grupo de volumes "vg0".

---- Cria volumes lógicos ----
lvcreate -L 16G -n swap vg0  |  # Swap (tamanho igual à RAM).
lvcreate -L 50G -n root vg0  |  # Partição raiz.
lvcreate -L 200G -n home vg0 |  # Home do usuário.
lvcreate -l 100%FREE -n lab vg0 | # Partição para laboratório.
```

# Formatação e montagem

#### **Formatação:**

```
mkfs.fat -F32 /dev/nvme0n1p1                # /efi
mkfs.ext4 /dev/nvme0n1p2                    # /boot
mkfs.ext4 /dev/mapper/vg0-root              # /
mkfs.ext4 /dev/mapper/vg0-home              # /home
mkfs.ext4 /dev/mapper/vg0-lab               # /lab
mkswap /dev/mapper/vg0-swap                 # Swap
```

#### **Montagem:**

```
mount /dev/mapper/vg0-root /mnt           # Monta raiz
mkdir -p /mnt/boot/efi                    # Cria diretório /efi
mount /dev/nvme0n1p2 /mnt/boot            # Monta /boot
mount /dev/nvme0n1p1 /mnt/boot/efi        # Monta /efi
mkdir /mnt/{home,lab}                     # Cria diretórios
mount /dev/mapper/vg0-home /mnt/home      # Monta /home
mount /dev/mapper/vg0-lab /mnt/lab        # Monta /lab
swapon /dev/mapper/vg0-swap               # Ativa swap
```

# Instalação do sistema base

Como meu hardware é intel irei instalar o software para atender ao hardware da intel. Então se esse não for o seu caso, você deverá consultar o guia oficial em [pacotes essenciais](https://wiki.archlinux.org/title/Installation_guide#Install_essential_packages) para mais dúvidas.

```
pacstrap /mnt base base-devel linux linux-firmware lvm2 vim nano intel-ucode

# base-devel: ferramentas de compilação
# intel-ucode: microcódigo para CPUs Intel
```

# Configurações básicas

```
genfstab -U /mnt >> /mnt/etc/fstab | # Geração do fstab.

arch-chroot /mnt | # Acessa o sistema instalado.

echo "arch-lab" > /etc/hostname | # Configuração do hostname.

ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime | # Difinindo o time zone.

hwclock --systohc | # Sincroniza hardware clock.

sed -i 's/#pt_BR.UTF-8 UTF-8/pt_BR.UTF-8 UTF-8/' /etc/locale.gen | # Habilita locale.

locale-gen | # Gera locales.

echo "LANG=pt_BR.UTF-8" > /etc/locale.conf | # Define a linguagem.
echo "KEYMAP=br-abnt2" > /etc/vconsole.conf | # Teclado ABNT2.
```

# Configurações do GRUB

```
nano /etc/mkinitcpio.conf | # Entrar no modo de edição de arquivo para editar o mkinitcpio.conf


HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block encrypt lvm2 filesystems fsck)

# Modificar a linha HOOKS para configura-la para rodar nossa criptografia. Só alterar e deixar como está acima, mas cuidado! Existem mais linhas HOOKS, modifiquem a correta!


mkinitcpio -P | # Regenerar initramfs


pacman -S grub efibootmgr | # Instalação do GRUB


grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB


---- Configuração do GRUB para LUKS ----

blkid -s UUID -o value /dev/nvme0n1p3 | # Comando para obter UUID da partição LUKS.


nano /etc/default/grub | # Comando para editar o arquivo.


GRUB_CMDLINE_LINUX="cryptdevice=UUID=<UUID_AQUI>:cryptlvm root=/dev/mapper/vg0-root" | # Pega o value UUID gerado e altere neste código, substituindo o "UUID_AQUI" pelo valor.


grub-mkconfig -o /boot/grub/grub.cfg | # Comando para gerar a configuração do GRUB.
```

# Configurações finais

```
useradd -m -G wheel,users,video,audio,storage,network -s /bin/bash seuusuario | # Comando utilizado para realizar a criação de usuário. Altere "seuusuario" pelo nome desejado.

passwd seuusuario | # Comando utilizado para definir a senha do usuário. Após esse comando você deverá digitar uma senha de sua escolha e digitar novamente para confirma-la.

EDITOR=nano visudo | # Configurar sudo.
	%wheel ALL=(ALL) ALL | # Descomente a linha.
```

Fica a seu critério as escolhas a seguir, pois a escolha de ambiente gráfico e os programas a serem instalados é de escolha pessoal do usuário.

```
---- Instalação de ambiente gráfico e programas ----

# Ambiente Gráfico e Drivers.
pacman -S gnome gnome-extra gdm xorg-server xf86-video-intel

# Ferramentas de Desenvolvimento.
pacman -S git python nodejs npm go rust jdk-openjdk ruby dotnet-sdk

# Virtualização e Containers.
pacman -S docker podman qemu virt-manager vagrant

# Ferramentas de Hacking/Segurança.
pacman -S nmap wireshark-qt metasploit john hashcat netcat

# Otimizações.
pacman -S zram-generator tlp powertop
systemctl enable tlp

# Utilitários.
pacman -S firefox neofetch htop
```
```
---- Habilitar serviços ----

systemctl enable gdm | # Interface gráfica.
systemctl enable NetworkManager | # Gerenciamento de rede.
systemctl enable docker | # Docker.
```

# Finalização

Se você chegou até aqui, parabéns! Agora é orar e pedir para dar certo. *Brincadeira :) ou não...*

```
exit | # Comando para sair do chroot.

umount -R /mnt | # Desmontar tudo.
swapoff -a

reboot | # Comando para reiniciar o sistema.
```

Após isso seu sistema estará instalado e pronto para uso!

# Contato

*Se tiver desejar passar algum insights, ou alguma recomendação fique à vontade para entrar em contato comigo.*
[contato@matheuslima.com](matheuslalvs@gmail.com)
