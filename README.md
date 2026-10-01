# Scripts de Boot do Amlogic para Armbian

**Language / Idioma:** [🟢 Português](README.md) | [English](README.en.md)

## Índice
- [Visão Geral](#visão-geral)
- [Configuração](#configuração)
- [Personalização do `aml_autoscript`](#personalização-do-aml_autoscript)
- [Dispositivos Suportados](#dispositivos-suportados)
- [Solução de Problemas — Executando o `aml_autoscript` Manualmente](#solução-de-problemas--executando-o-aml_autoscript-manualmente)
- [Como Funciona Internamente](#como-funciona-internamente)

---

## Visão Geral

As imagens do Armbian para TV Boxes Amlogic normalmente dependem de blobs secundários de u-boot para inicializar o kernel mainline. Na prática, eles não são necessários: o bootloader u-boot que veio de fábrica com a sua box já é capaz de fazer isso. Tudo que é preciso são algumas modificações nos scripts de boot do Armbian.

> **Pré-requisito:** o u-boot do fabricante deve estar rodando na eMMC. Se sua box foi reflashada com outro bootloader, restaure a imagem Android original com a ferramenta [Amlogic USB Burning Tool](https://androidmtk.com/download-amlogic-usb-burning-tool) antes de continuar.

> **Foco em mainline:** os scripts deste projeto removem as variáveis do Android do U-Boot. Se você quer continuar usando o Android, use os autoscripts originais do devmfc, de quem este projeto é um fork. Detalhes em [Personalização do `aml_autoscript`](#personalização-do-aml_autoscript).

---

## Configuração

### Passo 1 — Baixe a imagem do Armbian

Baixe a versão mais recente do Armbian para s9xxx-box. Recomendamos a [bookworm minimal](https://dl.armbian.com/aml-s9xx-box/Bookworm_current_minimal).

---

### Passo 2 — Prepare a mídia de instalação

Grave a imagem no pendrive usando o **[balenaEtcher](https://etcher.balena.io/)** — a opção mais simples — ou via linha de comando:

```bash
sudo dd if=Armbian_*.img of=/dev/sdX bs=4M status=progress conv=fsync
```

> ⚠️ Substitua `/dev/sdX` pelo seu pendrive. Use `lsblk` ou `fdisk -l` para confirmar o dispositivo correto. Com `dd`, gravar no dispositivo errado apaga os dados sem confirmação.

Monte a partição FAT do pendrive e substitua os scripts de boot pelos modificados, sobrescrevendo os existentes:

- **[aml_autoscript](https://github.com/projetotvbox/amlogic-bootscripts-Armbian/blob/main/aml_autoscript)**
- **[s905_autoscript](https://github.com/projetotvbox/amlogic-bootscripts-Armbian/blob/main/s905_autoscript)**
- **[emmc_autoscript](https://github.com/projetotvbox/amlogic-bootscripts-Armbian/blob/main/emmc_autoscript)**
- **[gxl-fixup.scr](https://github.com/projetotvbox/amlogic-bootscripts-Armbian/blob/main/gxl-fixup.scr)** *(apenas se o seu SoC for GXBB/S905 ou GXL/S905X/W/L)*

> **Compilando imagens próprias ou preparando múltiplos pendrives?** É possível modificar a imagem `.img` diretamente antes de gravar, evitando ter que editar cada pendrive individualmente. Para isso, monte a imagem com `losetup`:
>
> ```bash
> sudo losetup -fP Armbian_*.img
> lsblk | grep loop          # identifique o dispositivo e a partição FAT (geralmente loopXp1)
> sudo mount /dev/loop0p1 /mnt/armbian_boot
> ```
>
> Copie os scripts normalmente para `/mnt/armbian_boot/` e **prossiga pelos próximos passos normalmente**, editando o `armbianEnv.txt` e os demais parâmetros com a imagem ainda montada. Só ao final, após concluir todas as configurações, desmonte e grave:
>
> ```bash
> sudo umount /mnt/armbian_boot
> sudo losetup -d /dev/loop0
> sudo dd if=Armbian_*.img of=/dev/sdX bs=4M status=progress conv=fsync
> ```

---

### Passo 3 — Configure o `armbianEnv.txt`

O `armbianEnv.txt` controla parâmetros essenciais do boot. Antes de editar, faça um backup do arquivo original:

```bash
sudo cp /mnt/seu_pendrive/armbianEnv.txt /mnt/seu_pendrive/armbianEnv.txt.bak
```

Edite com nano ou o editor de sua preferência:

```bash
sudo nano /mnt/seu_pendrive/armbianEnv.txt
```

**Conteúdo de referência:**

```bash
extraargs=earlycon=meson,0xfe07a000 console=ttyS0,921600n8 rootflags=data=writeback rw no_console_suspend consoleblank=0 fsck.fix=yes fsck.repair=yes net.ifnames=0 watchdog.stop_on_reboot=0 pd_ignore_unused clk_ignore_unused rootdelay=5
bootlogo=false
verbosity=7
usbstoragequirks=0x2537:0x1066:u,0x2537:0x1068:u
console=both

# Arquivo DTB para este TV Box
# fdtfile=amlogic/meson-gxl-s905x-nexbox-a95x.dtb
fdtfile=amlogic/meson-sm1-x96-air-gbit.dtb

# defina isto para o UUID da partição raiz (o valor pode ser encontrado com blkid ou no fstab)
#rootdev=UUID=92139c84-3871-41d7-a3f2-e8a943cbfa87
# ou use o label padrão da partição:
#rootdev=LABEL=ROOTFS

# Ativar APENAS para gxbb (S905) / gxl (S905X/L/W) para criar cabeçalho u-boot falso
#soc_fixup=gxl-
```

> ⚠️ **Este arquivo é um ponto de partida, não uma configuração universal.**
>
> O conteúdo acima funciona para muitos dispositivos, mas pode não funcionar para o seu. Diferentes boxes, SoCs e versões do Armbian podem exigir parâmetros distintos — especialmente a linha `extraargs`.
>
> **Antes de substituir**, compare com o `armbianEnv.txt` original da imagem e mescle com cuidado. Parâmetros presentes no original e ausentes aqui podem ser necessários para o seu hardware. Em caso de dúvida, parta do original e aplique apenas as alterações que você compreende. Se o sistema não inicializar, restaurar o backup (`armbianEnv.txt.bak`) é o primeiro passo para diagnosticar.

---

### Passo 4 — Ajuste o `fdtfile`

Altere a linha `fdtfile` para o DTB correspondente à sua box. Os arquivos disponíveis estão em `/boot/dtb/amlogic/` dentro da imagem do Armbian.

---

### Passo 5 — Ajuste o `rootdev` *(opcional desde a versão 3)*

Por padrão, `rootdev` está comentado e o sistema usa o label `ROOTFS` automaticamente. Se precisar especificar manualmente:

| Mídia | Valor |
|-------|-------|
| Pendrive USB | `/dev/sda2` |
| Cartão SD | `/dev/mmcblk0p2` |
| Por UUID *(recomendado)* | `UUID=<seu-uuid>` |
| Por label | `LABEL=ROOTFS` |

**Como obter o UUID da partição raiz:**

Com o sistema rodando a partir do pendrive ou SD, execute:

```bash
blkid
```

Saída esperada:

```
/dev/sda2: UUID="92139c84-3871-41d7-a3f2-e8a943cbfa87" TYPE="ext4" PARTUUID="..."
```

Copie o valor `UUID=` da partição raiz (geralmente `sda2` ou `mmcblk0p2`) e cole no `armbianEnv.txt`. O UUID também pode ser encontrado em `/etc/fstab` ou no `armbianEnv.txt` original da imagem, se já estiver preenchido.

---

### Passo 6 — Ative o SoC fixup *(apenas GXBB/GXL)*

Se sua box usar um SoC GXBB (S905) ou GXL (S905X/W/L), descomente a linha:

```
soc_fixup=gxl-
```

---

### Passo 7 — Inicialize pelo pendrive

1. Desligue a box.
2. Insira o pendrive USB.
3. Pressione e **mantenha pressionado** o botão reset.
4. Ligue a box e continue segurando por aproximadamente **7 segundos**.
5. Se tudo estiver correto, o Armbian iniciará com kernel mainline — sem nenhum blob u-boot secundário.

> Na primeira vez, segurar o reset é o que faz o `aml_autoscript` rodar e gravar as novas variáveis. Depois disso, a box passa a procurar SD → USB → eMMC sozinha. Se o reset não surtir efeito, veja [Solução de Problemas](#solução-de-problemas--executando-o-aml_autoscript-manualmente).

---

## Personalização do `aml_autoscript`

O `aml_autoscript` deste projeto é um **fork do código original do devmfc, focado em Linux mainline**. Além de definir as variáveis do U-Boot para iniciar o Armbian, ele remove dezenas de variáveis que só existem para o Android (recovery, burning, Dolby Vision, A/B slots etc.). Isso deixa o ambiente do U-Boot limpo, facilitando o debug e a compreensão do código.

> ⚠️ **O Android deixa de iniciar pela eMMC.** O `bootcmd` passa a ser apenas `run start_autoscript` e a variável `storeboot` é removida. Se quiser continuar usando o Android, use os autoscripts originais do **devmfc**. Para voltar ao Android depois de rodar este script, restaure a imagem original com o [Amlogic USB Burning Tool](https://androidmtk.com/download-amlogic-usb-burning-tool).

O código-fonte editável é o `aml_autoscript.command`. O arquivo `aml_autoscript` (sem extensão) é a versão compilada, a que vai para a partição de boot.

> **Qualquer alteração só tem efeito depois de recompilar o script e executá-lo novamente na box** (segurando o reset, ou manualmente via console serial — veja a seção de solução de problemas). As variáveis só são gravadas pelo `saveenv` no final da execução.

### Recompilando o script

```bash
sudo apt install u-boot-tools   # fornece o mkimage (Debian/Ubuntu)
mkimage -C none -A arm -T script -d aml_autoscript.command aml_autoscript
```

Copie o `aml_autoscript` gerado para a partição FAT de boot, sobrescrevendo o existente.

---

### Bootlogo do U-Boot no Linux mainline

O `aml_autoscript` injeta uma função de bootlogo no U-Boot, algo que normalmente só o Android oferece. O logo aparece assim que a box liga, antes do kernel carregar.

Tudo que você precisa fazer é colocar um arquivo chamado **`bootlogo.bmp`** na **partição 1 (a de boot, FAT)** da mídia. O U-Boot procura o arquivo nesta ordem e usa o primeiro que encontrar:

1. Pendrive USB (portas 0 a 3)
2. Cartão SD
3. eMMC

Se nenhum arquivo for encontrado, a box simplesmente inicia sem logo. Para usar outro nome, altere a variável `bootlogo_filename` no `aml_autoscript.command` (sem a extensão `.bmp`) e recompile.

**Formato aceito**

O U-Boot da box só exibe BMPs em um formato específico. Na maioria dos casos é este (RGB565, 16 bits):

```bash
file bootlogo.bmp
# bootlogo.bmp: PC bitmap, Windows 3.x format, 320 x 388 x 16, 3 compression, image size 248320, cbSize 248386, bits offset 66
```

**Convertendo um PNG ou JPEG para o formato correto**

```bash
ffmpeg -i bootlogo.png -pix_fmt rgb565 -compression_level 0 bootlogo.bmp
```

> ⚠️ **Este formato é o mais comum, não um padrão garantido.** Cada U-Boot pode ter suas particularidades (resolução, profundidade de cor etc.). Se o logo não aparecer ou aparecer distorcido, você precisará adaptar a conversão ao seu caso — não há uma solução única que cubra todas as boxes.

---

### Exibindo o bootlogo na saída CVBS

Por padrão, o bootlogo é exibido apenas via HDMI. Se a sua box tiver saída CVBS (vídeo composto), é possível habilitá-la:

1. No `aml_autoscript.command`, altere:
   ```bash
   setenv cvbs_boot 0
   ```
   para:
   ```bash
   setenv cvbs_boot 1
   ```
2. [Recompile o script](#recompilando-o-script).
3. Copie o novo `aml_autoscript` para a partição de boot e execute-o novamente na box.

> ⚠️ **Padrão de vídeo do CVBS.** O `aml_autoscript` vem com o modo definido como `480cvbs` (525 linhas / 60 Hz), o formato aceito no Brasil e nos Estados Unidos:
>
> ```bash
> setenv cvbsmode 480cvbs
> ```
>
> Se a sua TV usar um padrão de 625 linhas / 50 Hz (comum na Europa, por exemplo), altere para `576cvbs`, recompile e execute o script novamente:
>
> ```bash
> setenv cvbsmode 576cvbs
> ```

---

### Reutilizando outro `aml_autoscript`

Por padrão, essa funcionalidade vem **desabilitada**: o U-Boot não procura mais um novo `aml_autoscript` ao ligar, o que mantém o boot mais simples e previsível.

Para reativá-la, edite o `aml_autoscript.command`: **descomente** as quatro linhas do bloco indicado e **comente** a linha `setenv update` que fica logo abaixo dele.

```bash
# Descomente estas linhas:
setenv check_update_button ${upgrade_key}
setenv update 'run load_aml_autoscript'
setenv load_aml_autoscript 'if mmcinfo; then if fatload mmc 0 1020000 aml_autoscript; then autoscr 1020000; fi; fi; if usb start; then for usbdev in 0 1 2 3; do if fatload usb ${usbdev} 1020000 aml_autoscript; then autoscr 1020000; fi; done; fi'
setenv bootcmd 'run check_update_button; run start_autoscript'

# E comente esta (mais abaixo no arquivo):
#setenv update
```

Com isso, o `bootcmd` volta a verificar o botão de reset e o `load_aml_autoscript` procura um `aml_autoscript` no SD e no USB. Recompile e execute o script para aplicar.

---

## Dispositivos Suportados

**✅ Totalmente Testado e Funcionando:**
- S905X, S905W, S912, S905X2, S922X, S905X3, S905X4 (HTV H8)

**⚠️ Suporte Parcial:**
- S905: Inicia apenas na primeira tentativa (limitação conhecida)

**❓ Não Testado:**
- S905W2: Provavelmente compatível mas não testado (não suportado atualmente pelo kernel do Armbian)

Todos os arquivos e fontes estão disponíveis no [Github](https://github.com/projetotvbox/amlogic-bootscripts-Armbian).

---

## Solução de Problemas — Executando o `aml_autoscript` Manualmente

> ⚠️ **Esta seção é para o caso em que segurar o botão reset não executa o `aml_autoscript`.** Se o método principal funcionou, você não precisa disso.
>
> Não é mais necessário digitar variáveis do U-Boot à mão: o `aml_autoscript` já faz toda a configuração. A única coisa que resta é executá-lo manualmente, interrompendo o U-Boot pelo console serial.

### Pré-requisitos

- **Adaptador Serial TTL (3.3V UART):** ⚠️ **Use apenas 3.3V. 5V danificará o dispositivo.** Requer solda nos pads TX/RX/GND da placa.
- **Software de terminal serial:** PuTTY, Minicom ou picocom.
- **Um pendrive formatado em FAT32** com o arquivo `aml_autoscript` na raiz.

### 🔒 Faça Backup da eMMC Antes de Qualquer Coisa

O `aml_autoscript` apaga o ambiente de fábrica do U-Boot (`defenv`) e remove as variáveis do Android. Se houver qualquer chance de você querer voltar atrás, faça o backup antes (com um sistema ARM Linux rodando pelo pendrive):

```bash
# Backup com compressão (um backup de 16GB vira 2-4GB)
sudo dd if=/dev/mmcblkX bs=1M status=progress | gzip -c > backup_emmc_full.img.gz

# Para restaurar:
# gunzip -c backup_emmc_full.img.gz | sudo dd of=/dev/mmcblkX bs=1M status=progress
```

### Passo 1 — Conectar o cabo serial

Solde TX, RX e GND nos pads UART do dispositivo e conecte ao PC.

### Passo 2 — Abrir o console serial

```bash
ls -la /dev/ttyUSB*

picocom -b 115200 /dev/ttyUSB0
# ou:
minicom -D /dev/ttyUSB0 -b 115200
```

### Passo 3 — Interromper o U-Boot

Com o pendrive conectado, ligue o dispositivo e pressione rapidamente `Ctrl+C` ou `Enter` para interromper o U-Boot antes de ele inicializar.

### Passo 4 — Executar o `aml_autoscript`

No console do U-Boot, execute:

```bash
usb start
fatload usb 0 $loadaddr aml_autoscript
autoscr $loadaddr
```

> ⚠️ **Mantenha apenas 1 pendrive conectado** durante este procedimento. O comando acima lê o primeiro dispositivo USB (`0`).

O script reescreve as variáveis, executa o `saveenv` e termina com `run start_autoscript`, iniciando o Armbian na sequência. Nas próximas vezes, não será mais necessário usar o console serial nem o botão reset.

---

## Como Funciona Internamente

Para quem quer entender o que acontece por baixo dos panos — o papel de cada arquivo na cadeia de boot.

### `aml_autoscript` — O Injetor de Nova Rota

Executado **uma única vez** por instalação, no momento em que você força o modo de recuperação (segurando o botão reset ao ligar) ou o roda manualmente pelo console serial. Como as variáveis são gravadas com `saveenv`, o resultado persiste nos boots seguintes. Em ordem, ele:

1. **Restaura o ambiente de fábrica** (`defenv`, `env default -a` e `saveenv`), partindo de uma base limpa.
2. **Define a nova rota de boot** (`start_autoscript`): SD card → USB → eMMC, redirecionando o fluxo para os scripts abaixo. O `bootcmd` passa a ser somente `run start_autoscript`, sem o `storeboot` do Android.
3. **Configura o bootlogo e a saída de vídeo** (`init_display`, executado no `preboot`): escolhe o modo de saída (HDMI ou CVBS), procura o `bootlogo.bmp` no USB, SD e eMMC, e o exibe.
4. **Remove as variáveis do Android** (recovery, burning, Dolby Vision, rede, A/B slots etc.), mantendo o ambiente enxuto.
5. **Grava tudo** com `saveenv` e chama `run start_autoscript` para iniciar o boot imediatamente.

Opcionalmente, é possível manter a capacidade de carregar outro `aml_autoscript` em boots futuros, como explicado em [Reutilizando outro `aml_autoscript`](#reutilizando-outro-aml_autoscript).

### `s905_autoscript` — O Carregador de Mídia Externa

Executado toda vez que a placa liga com pendrive ou SD card conectado. Lê o `armbianEnv.txt`, carrega o Kernel, DTB e Initrd para a RAM, prepara os `bootargs` e passa o controle para o Kernel inicializar o sistema.

### `emmc_autoscript` — O Carregador da Memória Interna

Idêntico ao anterior em função, mas acionado quando não há pendrive ou SD inicializável conectado. Aponta para o endereço físico da eMMC (`devnum 1`) e monta a raiz do sistema a partir da partição interna.

### `gxl-fixup.scr` — O Hack do Cabeçalho Falso *(apenas GXBB/GXL)*

Os bootloaders de fábrica das famílias GXBB (S905) e GXL (S905X/W/L) só aceitam kernels no formato legado `uImage` (comando `bootm`). O Armbian moderno usa o formato `Image` (comando `booti`), que esses bootloaders simplesmente recusam.

Em vez de compilar kernels legados, o script resolve isso em tempo de execução:

1. Substitui a rotina de boot padrão (`cmd_do_boot`).
2. Usa `mw.l` para escrever diretamente na memória um cabeçalho u-boot legado **falso** no endereço `0x1ffffc0`, logo antes do Kernel.
3. Injeta um CRC válido no cabeçalho (`cmd_hdr_crc`).
4. Dispara `bootm` — o bootloader vê o cabeçalho falso, acredita estar lidando com um `uImage` legítimo e inicializa o kernel moderno normalmente.

> **Restrição:** o arquivo do Kernel não pode ultrapassar **32MB**.

É por isso que o Passo 6 instrui a descomentar `soc_fixup=gxl-` nesses SoCs: sem esse hack, o bootloader travaria na inicialização.

---

Feito com 🐧 no IFSP Salto · Tecnologia a serviço da educação pública
