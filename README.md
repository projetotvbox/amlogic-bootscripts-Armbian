# Scripts de Boot do Amlogic para Armbian

**Language / Idioma:** [🟢 Português](README.md) | [English](README.en.md)

## Índice
- [Visão Geral](#visão-geral)
- [Configuração](#configuração)
- [Dispositivos Suportados](#dispositivos-suportados)
- [Solução de Problemas Avançada — Modificação do Bootloader](#solução-de-problemas-avançada--modificação-do-bootloader)
- [Como Funciona Internamente](#como-funciona-internamente)

---

## Visão Geral

As imagens do Armbian para TV Boxes Amlogic normalmente dependem de blobs secundários de u-boot para inicializar o kernel mainline. Na prática, eles não são necessários: o bootloader u-boot que veio de fábrica com a sua box já é capaz de fazer isso. Tudo que é preciso são algumas modificações nos scripts de boot do Armbian.

> **Pré-requisito:** o u-boot do fabricante deve estar rodando na eMMC. Se sua box foi reflashada com outro bootloader, restaure a imagem Android original com a ferramenta [Amlogic USB Burning Tool](https://androidmtk.com/download-amlogic-usb-burning-tool) antes de continuar.

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

## Solução de Problemas Avançada — Modificação do Bootloader

> ⚠️ **Esta seção é para casos onde os scripts simplesmente não funcionam.** Se o método principal funcionou, você não precisa disso.
>
> Alguns dispositivos possuem bootloaders de fábrica que não suportam a execução de scripts externos por padrão. Nesses casos, é possível modificar as variáveis do bootloader diretamente via console serial para forçar esse suporte. Trata-se de um procedimento de baixo nível, com risco real de brick. **Prossiga apenas se souber o que está fazendo.**

### Pré-requisitos

- **Sistema ARM Linux funcional:** Armbian, Debian ou Ubuntu ARM rodando a partir de USB/SD no dispositivo — necessário para acessar a eMMC e o shell.
- **Adaptador Serial TTL (3.3V UART):** ⚠️ **Use apenas 3.3V. 5V danificará o dispositivo.** Requer solda nos pads TX/RX/GND da placa.
- **Software de terminal serial:** PuTTY, Minicom ou picocom.

### 🔒 Faça Backup da eMMC Antes de Qualquer Coisa

```bash
# Backup com compressão (um backup de 16GB vira 2-4GB)
sudo dd if=/dev/mmcblkX bs=1M status=progress | gzip -c > backup_emmc_full.img.gz

# Para restaurar:
# gunzip -c backup_emmc_full.img.gz | sudo dd of=/dev/mmcblkX bs=1M status=progress
```

### Verificando o Suporte do Bootloader

#### Passo 1: Conectar o cabo serial
Solde TX, RX e GND nos pads UART do dispositivo e conecte ao PC.

#### Passo 2: Abrir o console serial

```bash
ls -la /dev/ttyUSB*

picocom -b 115200 /dev/ttyUSB0
# ou:
minicom -D /dev/ttyUSB0 -b 115200
```

#### Passo 3: Interromper o U-Boot
Ligue o dispositivo e pressione rapidamente `Ctrl+C` ou `Enter` para interromper o U-Boot antes de ele inicializar.

#### Passo 4: Verificar as variáveis do bootloader

```bash
printenv bootcmd
```

Saída esperada:
```
bootcmd=run start_autoscript; run storeboot
```

Verifique também:
```bash
printenv start_usb_autoscript
printenv start_mmc_autoscript
printenv start_emmc_autoscript
```

> Os nomes das variáveis podem variar. Procure por padrões como `start_*_autoscript`.

### Modificando as Variáveis

Se o bootloader for gravável, execute no console do U-Boot:

```bash
setenv start_autoscript 'if mmcinfo; then run start_mmc_autoscript; fi; if usb start; then run start_usb_autoscript; fi; run start_emmc_autoscript'
setenv start_emmc_autoscript 'if fatload mmc 1 1020000 emmc_autoscript; then setenv devtype "mmc"; setenv devnum 1; autoscr 1020000; fi;'
setenv start_mmc_autoscript 'if fatload mmc 0 1020000 s905_autoscript; then setenv devtype "mmc"; setenv devnum 0; autoscr 1020000; fi;'
setenv start_usb_autoscript 'for usbdev in 0 1 2 3; do if fatload usb ${usbdev} 1020000 s905_autoscript; then setenv devtype "usb"; setenv devnum 0; autoscr 1020000; fi; done'
setenv upgrade_step 2
setenv bootdelay 1
```

#### ⚠️ Configurando o `bootcmd` — Preserve o Comando Original

**Não use simplesmente `run start_autoscript; run storeboot`** sem antes verificar o seu `bootcmd` original. Um comando genérico pode danificar seu dispositivo se o original for diferente.

1. **Anote o `bootcmd` original:**
   ```bash
   printenv bootcmd
   ```

2. **Configure preservando o original:**
   ```bash
   setenv bootcmd 'run start_autoscript; [SEU BOOTCMD ORIGINAL AQUI]'
   ```

**Exemplos reais:**

```bash
# Exemplo 1 — Box Amlogic genérica (original: run storeboot)
setenv bootcmd 'run start_autoscript; run storeboot'

# Exemplo 2 — HTV H8 (original: run start_emmc_autoscript; run storeboot)
setenv bootcmd 'run start_autoscript; run start_emmc_autoscript; run storeboot'

# Exemplo 3 — bootcmd complexo
# Original: if test -n ${upgrade_step}; then echo BOOT_STEP equals $upgrade_step; setenv upgrade_step; fi; run storeboot
setenv bootcmd 'run start_autoscript; if test -n ${upgrade_step}; then echo BOOT_STEP equals $upgrade_step; setenv upgrade_step; fi; run storeboot'
```

#### Salve e verifique

```bash
saveenv
reset
```

Interrompa o U-Boot novamente e confirme:

```bash
printenv bootcmd
```

- **Variáveis salvas** → bootloader gravável, modificações aplicadas com sucesso.
- **Variáveis não salvas** → bootloader somente leitura; este método não é aplicável.

---

## Como Funciona Internamente

Para quem quer entender o que acontece por baixo dos panos — o papel de cada arquivo na cadeia de boot.

### `aml_autoscript` — O Injetor de Nova Rota

Executado **apenas uma vez**, no momento em que você força o modo de recuperação (segurando o botão reset ao ligar). Ele reescreve as variáveis de ambiente do U-Boot de fábrica via `saveenv`, estabelecendo uma nova ordem de boot: SD card → USB → eMMC, redirecionando o fluxo para os scripts abaixo.

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
