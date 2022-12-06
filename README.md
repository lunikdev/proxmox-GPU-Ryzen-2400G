# Proxmox-GPU-Ryzen-2400G
Como fazer passthrough de GPU em Ryzen 2400G

Esse tutorial foi o mais simplificado possivel para auxiliar você usuario mediano ou até mesmo avançado que não está conseguindo fazer a passagem da GPU no proxmox com o processador Ryzen 5 2400G ou similar sejá por incompatibilidade com o processador(travamentos e outros bugs) ou até mesmo pela placa mãe ser um modelo mais simples (A320) e não auxiliar na passagem gpu e acabar causando diversos outros problema não permitindo que a passagem seja feita.

Pre-configuração (requisistos)

Esse tutorial apenas server como base para você realizar uma passagem de GPU bem sucedida, mais abaixo vou esta listando o meu hardware e configuração para servir como referencia, se caso não funcionar você podera realizar alterações para se adequar e seu setup.

Para começarmos o seu hardware deve ter suporte a: VT-d, interrupt mapping, and UEFI BIOS.

Hardware:

Placa-mãe: Asrock A320M-HD
Cpu: Ryzen 2400G, AM4
Memoria: 16 DDR4 gb
GPU: RX 5500XT 8gb (51RISC)


Software:

Proxmox VE 7.3
Windows 10 21H2 ISO

Configuração bios:

As configurações apresentadas a seguir obtevem exito atraves de tentativa erro, infelizmente configurações diferente causavam bugs diferente e as vezes a maquina nem chega a iniciar, travava o proxmox ou nem chegava a dar video.


****

Configuração Promox

Focaremos o tutorial na configuração do passthrough a parte basica da instalação do proxmox você pode consultar no site oficial do proxmox https://www.proxmox.com/en/proxmox-ve/get-started ou no propio youtube.

Passo 1: Configurando Grub

Apos logar dentro da interface do proxmox pelo navegador clique em seu Node e abra o Shell.
Com o terminal aberto realizaremos a editação do grub com o editor Nano digite o comando abaixo:

nano /etc/default/grub

Encontre a linha:

GRUB_CMDLINE_LINUX_DEFAULT="quiet"

Vamos modificar ela para:

RUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt pcie_acs_override=downstream,multifunction nofb nomodeset consoleblank=1 initcall_blacklist=sysfb_init"

Esse foi o comando que funcionou para mim, tentei diversas variações que encontrei em foruns e pela internet porém o Proxmox acabava tendo acesso a placa e não permitindo a passagem da GPU para a VM, os comandos chave nessa linha que mais me ajudaram a corrigir o meu problema foram "initcall_blacklist=sysfb_init" (ele ajudou a bloquear o acesso do proxmox a placa e conseguir realizar o boot da maquina com o HDMI conectado)  e "consoleblank=1" (retiar as letras do consola na tela do HDMI que ficava piscando), antes desses comandos eu tinha que iniciar a VM sem nenhuma nenhum monitor conectado ao HDMI da GPU e depois da maquina iniciada eu poderia conectar o monitor.

Caso não tenha exito no final do tutorial eu deixou outras três variações do comando abaixo.

GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt pcie_acs_override=downstream,multifunction nofb nomodeset consoleblank=1"

GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt pcie_acs_override=downstream,multifunction nofb nomodeset

GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt pcie_acs_override=downstream,multifunction nofb nomodeset video=vesafb:off,efifb:off"


Após realizar a alteração salve o arquivo com o nano retorne ao terminal e digite:

update-grub

Após isso no terminal:

reboot

Realizar 

Passo 2: VFIO Modules

Com a maquina reiniciada retorne ao Shell e digite:

nano /etc/modules

Adicione dentro:

vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd

Salve e saia

Passo 3:IOMMU interromper o remapeamento

Digite esses comandos no shell e de enter:

echo "options vfio_iommu_type1 allow_unsafe_interrupts=1" > /etc/modprobe.d/iommu_unsafe_interrupts.conf
echo "options kvm ignore_msrs=1" > /etc/modprobe.d/kvm.conf

Apos realize o reboot da maquina.


Passo 4: Blacklist drives ???

Geralmente iriamos realizar o blacklist do drive para evitar que o proxmox acessase a GPU e causase conflito com a VM porém fazer isso por algum motivo causou mais conflito com a VM do que ajudou então eu não realizei, deixei esse passo aqui apenas para você saber o motivo de eu não realizar o Blacklist do drive de video.

Passo 5: Adicionando GPU ao VFIO

Use esse comando para puxar todos os ID relacionado a amd/ati

lspci -nn | grep -i amd/ati

Resultado:

01:00.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 10 XL Upstream Port of PCI Express Switch [1002:1478] (rev c5)
02:00.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 10 XL Downstream Port of PCI Express Switch [1002:1479]
03:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 14 [Radeon RX 5500/5500M / Pro 5500M] [1002:7340] (rev c5)
03:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 10 HDMI Audio [1002:ab38]

Use esse comando para puxar todos os ID relaciona a USB caso queira fazer a passagem da USB para a VM.

lspci -nn | grep -i usb

Resultado:

04:00.0 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Device [1022:43bc] (rev 02)
09:00.3 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Raven USB 3.1 [1022:15e0]
09:00.4 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Raven USB 3.1 [1022:15e1]

Peguei todos os os ids e coloquem eles separados por virgula.

1002:1478,1002:1479,1002:7340,1002:ab38,1022:43bc,1022:15e0,1022:15e1

Agora una ele a esse comando echo "options vfio-pci ids=(CHANGE) disable_vga=1"> /etc/modprobe.d/vfio.conf

Ira ficar assim:

echo "options vfio-pci ids=1002:1478,1002:1479,1002:7340,1002:ab38,1022:43bc,1022:15e0,1022:15e1 disable_vga=1"> /etc/modprobe.d/vfio.conf

Cole e execute o comando no terminal Shell.

Cole e execute:

update-initramfs -u

Cole e execute:

reset

E por fim reinicie a maquina:

reboot
