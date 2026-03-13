# Scripts de Boot do Amlogic para Armbian

**Language / Idioma:** [🟢 Português](README.md) | [English](README.en.md)

## Índice
- [Visão Geral](#visão-geral)
- [Configuração](#configuração)
- [Dispositivos Suportados](#dispositivos-suportados)
- [Avançado - Personalização do Bootloader](#avançado---personalização-do-bootloader)

## Visão Geral

As imagens do Armbian para TV Boxes Amlogic usam blobs secundários de u-boot carregados em cadeia para inicializar imagens do kernel mainline.
Os bootloaders u-boot do fabricante, no entanto, podem inicializar o Linux mainline perfeitamente sem eles. Portanto, eles não são necessários.

Tudo o que é necessário são algumas modificações simples em alguns dos scripts u-boot do Armbian.

## Configuração
Pressuposição: você tem o u-boot do fabricante (o que veio com a box) rodando na eMMC. Se não, você pode restaurar a imagem Android original com a ferramenta Amlogic USB Burning.

+ **Passo 1:** Baixe a versão mais recente do Armbian para s9xxx-box, vamos usar [bookworm minimal](https://dl.armbian.com/aml-s9xx-box/Bookworm_current_minimal)  
+ **Passo 2:** Grave a imagem em um pendrive USB  
+ **Passo 3:** Copie os scripts de boot modificados (**[aml_autoscript](https://github.com/projetotvbox/amlogic-bootscripts-Armbian/blob/main/aml_autoscript)**, **[s905_autoscript](https://github.com/projetotvbox/amlogic-bootscripts-Armbian/blob/main/s905_autoscript)**, **[emmc_autoscript](https://github.com/projetotvbox/amlogic-bootscripts-Armbian/blob/main/emmc_autoscript)** ) para a partição FAT no pendrive. Sobrescreva os arquivos existentes.  
+ **Passo 4:** Se você tem um SoC GXBB (S905) ou GXL (S905X/W/L), você também precisa de **[gxl-fixup.scr](https://github.com/projetotvbox/amlogic-bootscripts-Armbian/blob/main/gxl-fixup.scr)**  
+ **Passo 5:** Adicione um arquivo armbianEnv.txt com o seguinte conteúdo (o arquivo também está no github):  
```bash
extraargs=earlycon=meson,0xfe07a000 console=ttyS0,921600n8 rootflags=data=writeback rw no_console_suspend consoleblank=0 fsck.fix=yes fsck.repair=yes net.ifnames=0 watchdog.stop_on_reboot=0 pd_ignore_unused clk_ignore_unused rootdelay=5
bootlogo=false
verbosity=7
usbstoragequirks=0x2537:0x1066:u,0x2537:0x1068:u
console=both

# Arquivo DTB para este TV Box
# fdtfile=amlogic/meson-gxl-s905x-nexbox-a95x.dtb
fdtfile=amlogic/meson-sm1-x96-air-gbit.dtb

# defina isto para o UUID da partição raiz (o valor pode ser encontrado 
# em /extlinux/extlinux.conf depois de APPEND root= ou com blkid)
rootdev=UUID=92139c84-3871-41d7-a3f2-e8a943cbfa87

# Ativar APENAS para gxbb (S905) / gxl (S905X/L/W) para criar cabeçalho u-boot falso
#soc_fixup=gxl-
```
+ **Passo 6:** Altere *fdtfile* para o DTB da sua box.  
+ **Passo 7:** (opcional desde a versão 3:) Altere *rootdev* para o UUID correto do rootfs para sua imagem ou mude para /dev/sda2 quando inicializar do USB ou /dev/mmcblk0p2 quando inicializar do SDCARD  
+ **Passo 8:** Apenas se sua box tiver um SoC GXBB (S905) ou GXL (S905X/W/L), descomente a linha *soc_fixup=gxl-*  
+ **Passo 9:** Desligue a box.  
+ **Passo 10:** Coloque o disco USB na sua box.  
+ **Passo 11:** Pressione o botão reset e mantenha pressionado  
+ **Passo 12:** Ligue a box enquanto mantém o botão reset pressionado por aproximadamente 7 segundos.  
+ **Passo 13:** Se você tiver sorte, agora ele inicializará o Armbian com um kernel mainline. Sem nenhum blob u-boot secundário.  

## Dispositivos Suportados

**✅ Totalmente Testado e Funcionando:**
- S905X, S905W, S912, S905X2, S922X, S905X3, S905X4 (HTV H8)

**⚠️ Suporte Parcial:**
- S905: Inicia apenas na primeira tentativa (limitação conhecida)

**❓ Não Testado:**
- S905W2: Provavelmente compatível mas não testado (não suportado atualmente pelo kernel do Armbian)

Todos os arquivos usados e arquivos de origem podem ser encontrados no [Github](https://github.com/projetotvbox/amlogic-bootscripts-Armbian).

---

## Avançado - Personalização do Bootloader

### ⚠️ Aviso Legal & Pré-requisitos

**AVISO:** Modificar o bootloader do seu dispositivo pode resultar em um dispositivo travado (brick). Qualquer dano ou perda de dados é de sua responsabilidade exclusiva. Proceda apenas se entender os riscos.

**Pré-requisitos Obrigatórios:**

- **Sistema ARM Linux Funcional:** Armbian, Debian ou Ubuntu ARM rodando a partir de USB/SD no seu dispositivo Amlogic
  - Necessário para acessar a eMMC interna e executar comandos de análise/extração
  - O sistema deve inicializar corretamente para fornecer acesso shell
  
- **Adaptador Serial TTL (3.3V UART):** Adaptador série USB de alta qualidade
  - ⚠️ **CRÍTICO:** Use apenas 3.3V. 5V danificará o dispositivo!
  - Requer habilidades de soldagem para conectar TX/RX/GND na placa
  
- **Software de Terminal Serial:** PuTTY, Minicom ou picocom
  
- **Paciência e Metodologia:** Siga cada passo cuidadosamente

### 🔒 Obrigatório: Faça Backup da sua eMMC

Antes de QUALQUER experimento, crie um backup completo:

```bash
# Backup bit-a-bit com compressão (economiza espaço)
sudo dd if=/dev/mmcblkX bs=1M status=progress | gzip -c > backup_emmc_full.img.gz

# Para restaurar em caso de desastre:
# gunzip -c backup_emmc_full.img.gz | sudo dd of=/dev/mmcblkX bs=1M status=progress
```

Por que gzip? Um backup de 16GB se torna 2-4GB, economizando espaço significativo.

### Verificando o Suporte do Bootloader do Fabricante

#### Passo 1: Conectar Cabo Serial
Solde TX, RX, GND nos pads UART do seu dispositivo e conecte ao seu PC.

#### Passo 2: Abrir Console Serial
Usando picocom como exemplo:

```bash
# Encontre seu dispositivo serial
ls -la /dev/ttyUSB*

# Conecte a 115200 baud (ajuste se diferente)
picocom -b 115200 /dev/ttyUSB0

# Ou com minicom:
minicom -D /dev/ttyUSB0 -b 115200
```

#### Passo 3: Interromper U-Boot
Ligue o dispositivo e pressione rapidamente `Ctrl+C` ou `Enter` para interromper o U-Boot antes de inicializar.

#### Passo 4: Verificar Variáveis do Bootloader
Uma vez no console do U-Boot, digite:

```bash
printenv bootcmd
```

A saída esperada deve ser semelhante a:
```
bootcmd=run start_autoscript; run storeboot
```

Verifique as variáveis relacionadas:
```bash
printenv start_usb_autoscript
printenv start_mmc_autoscript
printenv start_emmc_autoscript
```

**Nota:** Os nomes das variáveis podem ser ligeiramente diferentes. Procure por padrões como `start_*_autoscript`.

### Modificando Bootloader do Fabricante (Apenas Usuários Avançados)

Se seu bootloader é gravável e você quer forçar o suporte a scripts, execute esses comandos no console do U-Boot:

```bash
setenv start_autoscript 'if mmcinfo; then run start_mmc_autoscript; fi; if usb start; then run start_usb_autoscript; fi; run start_emmc_autoscript'
setenv start_emmc_autoscript 'if fatload mmc 1 1020000 emmc_autoscript; then setenv devtype "mmc"; setenv devnum 1; autoscr 1020000; fi;'
setenv start_mmc_autoscript 'if fatload mmc 0 1020000 s905_autoscript; then setenv devtype "mmc"; setenv devnum 0; autoscr 1020000; fi;'
setenv start_usb_autoscript 'for usbdev in 0 1 2 3; do if fatload usb ${usbdev} 1020000 s905_autoscript; then setenv devtype "usb"; setenv devnum 0; autoscr 1020000; fi; done'
setenv bootdelay 1
```

#### ⚠️ CRÍTICO: Configurando bootcmd - Preserve Seu Comando Original

**NÃO use simplesmente `run start_autoscript; run storeboot`** - Isto é genérico e pode danificar seu dispositivo se seu bootcmd original foi diferente!

**Abordagem passo-a-passo:**

1. **Primeiro, ANOTE seu bootcmd original:**
   ```bash
   printenv bootcmd
   # Anote a saída EXATA aqui:
   # _________________________________
   ```

2. **Então configure bootcmd para preservá-lo:**
   ```bash
   setenv bootcmd 'run start_autoscript; [COLE SEU BOOTCMD ORIGINAL AQUI]'
   ```

**Exemplos de dispositivos reais:**

**Exemplo 1 - Box Amlogic Genérica:**
```bash
# Original era:
# bootcmd=run storeboot

# Então você faz:
setenv bootcmd 'run start_autoscript; run storeboot'
```

**Exemplo 2 - Fabricante Diferente (HTV H8):**
```bash
# Original era:
# bootcmd=run start_emmc_autoscript; run storeboot

# Então você faz:
setenv bootcmd 'run start_autoscript; run start_emmc_autoscript; run storeboot'
```

**Exemplo 3 - Bootcmd Complexo:**
```bash
# Original era:
# bootcmd=if test -n ${upgrade_step}; then echo BOOT_STEP equals $upgrade_step; setenv upgrade_step; fi; run storeboot

# Então você faz:
setenv bootcmd 'run start_autoscript; if test -n ${upgrade_step}; then echo BOOT_STEP equals $upgrade_step; setenv upgrade_step; fi; run storeboot'
```

#### Passo Final - Salve e Verifique as Alterações

```bash
saveenv
reset
```

Interrompa o U-Boot novamente e verifique se as variáveis foram salvas:

```bash
printenv bootcmd
```

**Indicadores de Sucesso:**
- Variáveis foram salvas → Bootloader é gravável e as modificações devem funcionar
- Variáveis não foram salvas → Bootloader somente leitura; não é possível aplicar este método

---
