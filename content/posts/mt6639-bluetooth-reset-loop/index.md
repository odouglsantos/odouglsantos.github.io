---
author: Douglas Santos
title: "Bluetooth fantasma no Linux: como o btusb trava o MediaTek MT6639 tentando carregar um firmware que não existe."
date: "2026-09-09"
description: "Meu Bluetooth nunca funcionou nesta placa e eu tinha certeza de que era firmware faltando. Era — mas o que quebrava o hardware de verdade era o driver tentando resolver isso pra sempre. Aqui está a investigação inteira, do falso negativo no dmesg até uma corrida de 174 milissegundos no boot."
draft: false
slug: "mt6639-bluetooth-loop-de-reset"
tags:
  - posts
---

Comprei uma ASUS ProArt X870E-Creator WiFi e o Wi-Fi funcionou de primeira. O Bluetooth, não. Nunca apareceu adaptador nenhum, e o pior: o dispositivo simplesmente não estava no `lsusb`. Nem travado, nem com erro — ausente.

Minha primeira hipótese foi a mais óbvia, e estava certa pela metade: o firmware de Bluetooth desse chip não vem no `linux-firmware`. O que eu não imaginava é que o firmware faltando não é o que quebra o hardware. O que quebra é o que o driver faz a respeito disso.

Neste artigo vou reconstruir a investigação inteira, incluindo os dois momentos em que eu conclui a coisa errada com confiança total. Acho que essas partes são mais úteis que a solução.

| | |
|---|---|
| Placa-mãe | ASUS ProArt X870E-Creator WiFi rev 2 |
| BIOS | 2402 |
| Sistema | Bazzite (Fedora 44 Atomic) |
| Kernel | 7.2.3-ogc3.1.fc44.x86_64 |
| Bluetooth | MediaTek MT6639, USB `0489:e13a` |
| Wi-Fi | MediaTek MT7927, PCIe `14c3:7927` |

## O sintoma: um adaptador que não existe

Nenhum adaptador. O `bluetoothctl show` trava sem imprimir nada, porque o `bluetoothd` fica esperando um controlador que nunca vai aparecer. E o `dmesg` não tinha absolutamente nenhuma mensagem do subsistema Bluetooth — nem sucesso, nem falha de firmware, nem erro de USB. Silêncio total.

Silêncio total é um resultado estranho. Se o firmware faltasse, eu deveria ver o driver reclamando. Se o dispositivo estivesse com defeito, eu deveria ver o USB reclamando. Não ver nada significa que o kernel nem tentou.

## O falso negativo que me custou uma rodada inteira

Aqui está o primeiro erro, e ele é constrangedor de tão simples.

```bash
dmesg | grep -iE 'btusb|bluetooth: hci'
```

Isso voltava vazio. Eu li "vazio" como "não há mensagens de Bluetooth". Errado. O que estava acontecendo é que o `kernel.dmesg_restrict=1` faz o `dmesg` sem root retornar **zero linhas de qualquer tipo**. Não havia mensagem nenhuma pra filtrar — de Bluetooth, de USB, de nada.

```bash
$ dmesg | wc -l
0
$ sysctl -n kernel.dmesg_restrict
1
```

Um `grep` vazio em cima de uma entrada vazia parece exatamente igual a um `grep` vazio em cima de mil linhas. A lição prática: use `journalctl -k`, que funciona sem sudo e te dá o log de verdade.

```bash
$ journalctl -k -b 0 --no-pager | wc -l
1893
```

Mil oitocentas e noventa e três linhas que eu vinha ignorando. E dentro delas estava tudo.

## O dispositivo não sumiu — ele travou

Com o log de verdade em mãos, o quadro mudou completamente. O dispositivo **está** lá, na porta `1-6`, vizinha do controlador de LED da placa. Ele é detectado eletricamente. Ele simplesmente não responde a nada.

```
usb 1-6: new high-speed USB device number 4 using xhci_hcd
usb 1-6: device descriptor read/64, error -110
usb 1-6: device descriptor read/64, error -110
usb 1-6: new high-speed USB device number 5 using xhci_hcd
usb 1-6: device descriptor read/64, error -110
usb 1-6: device descriptor read/64, error -110
usb usb1-port6: attempt power cycle
usb 1-6: new high-speed USB device number 6 using xhci_hcd
usb 1-6: Device not responding to setup address.
usb 1-6: device not accepting address 6, error -71
usb 1-6: new high-speed USB device number 7 using xhci_hcd
usb 1-6: Device not responding to setup address.
usb 1-6: device not accepting address 7, error -71
usb usb1-port6: unable to enumerate USB device
```

Traduzindo: `-110` é timeout, `-71` é erro de protocolo. O hub vê que tem alguma coisa plugada, tenta ler o descritor do dispositivo, não recebe resposta, corta e religa a energia da porta, tenta de novo, desiste. Quatro tentativas, sessenta e três segundos, nada.

Isso também explica por que o `btusb` não estava carregado, e por que isso **não** era o problema. O udev carrega o módulo quando aparece um modalias que casa com ele. Como nada enumerou, não existe modalias, então o módulo não carrega. Carregar na mão funciona sem erro nenhum e não cria dispositivo HCI algum:

```bash
$ sudo modprobe btusb && ls /sys/class/bluetooth/
# (vazio)
```

A pilha de software está impecável. O hardware é que não aparece.

## A causa: o btusb tenta pra sempre

O journal do Bazzite guarda os boots anteriores, e é aí que a coisa fica interessante. Varri todos os boots registrados procurando pelo dispositivo, e em dois deles ele **funcionou** — enumerou normalmente. Fui olhar o que tinha acontecido:

```
[    3.068951] usb 1-6: New USB device found, idVendor=0489, idProduct=e13a
[    3.069092] usb 1-6: Product: Wireless_Device
[    8.221018] usbcore: registered new interface driver btusb
[    8.233139] Bluetooth: hci0: Failed to load firmware file (-2)
[    8.233145] Bluetooth: hci0: Failed to set up firmware (-2)
[    8.564037] usb 1-6: reset high-speed USB device number 4 using xhci_hcd
[    8.817354] Bluetooth: hci0: Failed to load firmware file (-2)
[    9.147127] usb 1-6: reset high-speed USB device number 4 using xhci_hcd
[    9.402354] Bluetooth: hci0: Failed to load firmware file (-2)
[    9.727020] usb 1-6: reset high-speed USB device number 4 using xhci_hcd
```

Achou o padrão? O firmware falha com `-2` (ENOENT, arquivo não encontrado), e o `btusb` **reseta o dispositivo por USB e tenta de novo**. Aí falha de novo, reseta de novo, tenta de novo. A cada 0,58 segundos. Sem backoff, sem limite de tentativas, sem desistir nunca.

Naquele boot isso rodou por 13 minutos até eu desligar a máquina. Foram 1335 resets.

E é isso que trava o chip. A cadeia inteira é assim:

1. Partida a frio limpa: o controlador enumera normalmente, uns 3 segundos depois do boot.
2. O `btusb` faz bind aos ~8 segundos, cria o `hci0`, pede o firmware.
3. O arquivo não existe. Retorna `-2`.
4. **O `btusb` reseta o dispositivo e tenta de novo. E de novo. Indefinidamente.**
5. Depois de algumas centenas de resets, o firmware do controlador trava.
6. A partir daí, a porta detecta o dispositivo mas ele não responde mais nada.
7. Isso **sobrevive a reboots**, porque o trilho de energia de standby mantém o chip alimentado.

O ponto 7 é o que torna isso cruel. Você reinicia, não resolve. Reinstala o sistema, não resolve. O chip continua travado porque nunca foi desenergizado de verdade.

## As contas batem

O que me convenceu de que essa era mesmo a causa, e não uma coincidência, foi contar. Se o loop de reset é consequência da falha de firmware, os dois números têm que ser idênticos:

| Boot | Resets da porta 1-6 | Falhas de firmware |
|---|---:|---:|
| -8 | 398 | 398 |
| -1 | 1335 | 1335 |

Batem exatamente. Um reset para cada falha, nos dois boots.

E o padrão de reincidência entre boots conta o resto da história:

| Boot | Porta 1-6 | Leitura |
|---|---|---|
| -8 | enumerou | partida limpa, 398 resets, trava |
| -7 a -3 | sem eventos | **cinco boots mortos**, consequência do -8 |
| -2 | falha na enumeração | travado |
| -1 | enumerou | destravado por corte de energia, 1335 resets, trava de novo |
| 0 | falha na enumeração | travado |

Dois ciclos completos. Toda vez que eu destravava o chip cortando energia, ele voltava, entrava no loop, e travava de novo. Eu estava reproduzindo o problema sem saber.

Para destravar, o caminho é cortar a energia de verdade: habilitar **ErP em S4+S5** na BIOS e usar `poweroff` (não `reboot`, que nunca corta o standby), ou tirar da tomada por uns 10 segundos.

## Ah, e o boot lento era o mesmo bug

Eu tinha uma segunda queixa que achava não ter relação: o boot estava demorando quase dois minutos. Tinha.

O `systemd-udev-settle.service` fica na **raiz da cadeia crítica** do boot — tudo espera por ele. E as retentativas de enumeração na porta 1-6 mantêm o udev ocupado:

| Estado da porta 1-6 | Boots | udev-settle |
|---|---|---:|
| dispositivo ausente | -7 a -3 | 8,6 – 9,7 s |
| dispositivo travado | -2, 0 | **67,8 s** |

Uns 59 segundos de penalidade, batendo com a janela de 63 segundos das retentativas. Depois que o controlador subiu direito, o boot foi de **1min52s para 41s**. Dois sintomas, um bug só.

## Onde colocar firmware quando /usr é read-only

O Bazzite é um sistema imutável: `/usr` é somente-leitura, então não dá pra simplesmente jogar o arquivo em `/usr/lib/firmware`. A rota padrão é usar `/var/lib/firmware` e apontar o kernel pra lá com um parâmetro de boot:

```bash
sudo install -Dm644 BT_RAM_CODE_MT6639_2_1_hdr.bin \
  /var/lib/firmware/mediatek/mt7927/BT_RAM_CODE_MT6639_2_1_hdr.bin

sudo rpm-ostree kargs --append=firmware_class.path=/var/lib/firmware
```

Fiz isso, reiniciei, e o Bluetooth funcionou. Fim do artigo, certo?

Não. Fui conferir o log e o firmware **tinha falhado com `-2` de novo** — e mesmo assim o dispositivo subiu. Isso não fazia sentido nenhum, e a explicação é a parte mais interessante de tudo.

## A corrida que eu ganhei por 174 milissegundos

Em sistemas ostree, o `/var` é um subvolume montado por uma unit do systemd **depois** do switch-root. E o `btusb` faz probe exatamente dentro dessa janela.

| Tempo | Evento |
|---:|---|
| 7,294 s | switch-root |
| 8,677 s | `btusb` registra o interface driver |
| **8,688 s** | **primeira tentativa de firmware falha, `-2`** |
| 9,021 s | `btusb` reseta o dispositivo e reagenda |
| **9,235 s** | **`var.mount` conclui** |
| 9,409 s | o retry **encontra** o firmware |
| 28,758 s | `Device setup in 19036182 usecs` |
| 28,930 s | `AOSP extensions version v1.00` |

Ou seja: funcionou pelo motivo errado. A primeira iteração do loop de reset — o mesmo loop que trava o chip — foi justamente o que salvou, porque o `/var` montou no meio dela. A margem foi de 174 milissegundos.

Isso é sorte reproduzível, não garantia. Se o retry caísse 200 ms mais cedo, o loop começaria e o chip travaria. Ainda por cima, tem um detalhe cruel: existe um `/var` stub não-vazio embaixo do ponto de montagem no deployment, então a busca falha **em silêncio** em vez de dar erro de diretório ausente.

## A correção de verdade

A solução é colocar o firmware num lugar que já esteja legível no switch-root. O `/etc` serve: ele é um bind mount do próprio subvolume do deployment, montado pelo `ostree-prepare-root` ainda dentro do initrd. Está disponível aos 7,294 s, mais de um segundo antes do `btusb` fazer probe.

```bash
sudo install -Dm644 BT_RAM_CODE_MT6639_2_1_hdr.bin \
  /etc/firmware/mediatek/mt7927/BT_RAM_CODE_MT6639_2_1_hdr.bin

sudo rpm-ostree kargs \
  --replace=firmware_class.path=/var/lib/firmware=/etc/firmware
```

Detalhe simpático: o SELinux rotula o arquivo como `cpucontrol_conf_t`, porque `/etc/firmware` já é um caminho que a policy conhece (usado para microcode de CPU). E não bloqueia nada — a policy carregada nem define a permissão `firmware_load`.

Para validar depois do reboot, os dois números têm que dar zero:

```bash
journalctl -k -b 0 | grep -c 'Failed to load firmware file'
journalctl -k -b 0 | grep -c 'reset high-speed USB device'
```

## A armadilha do /etc staged — quase desisti aqui

Esse foi o segundo momento em que eu conclui a coisa errada com confiança total, e valeu um susto.

Depois do `rpm-ostree kargs`, fui conferir o deployment novo antes de reiniciar. O `/etc/firmware` **não estava lá**. E o deployment é read-only, então nem dava pra colocar na mão.

Isso parecia fatal. Com o parâmetro de boot apontando só pro `/etc/firmware`, um arquivo ausente significa que a primeira tentativa **e o retry** falham, já que o `/var` saiu do caminho de busca. Ou seja: pior do que não ter mexido em nada.

Antes de reverter, resolvi comparar as duas árvores de `/etc`. A do deployment novo estava sem 23 itens que a atual tinha:

```
bazzite    cardwire   cni        crypttab   firmware
fstab      group-     gshadow-   hostname   iwd
locale.conf localtime passwd-    sddm.conf.d shadow-
subgid-    subuid-    vconsole.conf
```

Olha o `fstab` ali no meio.

É isso que resolve a questão. Se a finalização não fizesse o merge do `/etc`, **nenhum sistema ostree conseguiria bootar depois de um upgrade**, porque ficaria sem `fstab`. Logo, o `/etc` de um deployment staged é o *pristine* do commit novo, e o merge de 3 vias é diferido para o `ostree admin finalize-staged`, que roda no shutdown.

Ou seja: inspecionar um `/etc` staged **sempre** vai parecer que suas modificações locais sumiram. É o esperado, não um defeito. Meu arquivo ia ser carregado junto com o `fstab` e o `hostname`, e foi.

O que me tirou do buraco não foi conhecimento prévio de ostree, foi procurar uma evidência que decidisse a questão em vez de apostar num palpite.

## De onde vem esse firmware, afinal

O blob de Bluetooth não está no `linux-firmware`. O MR !946 foi fechado porque o projeto só aceita blobs submetidos por quem detém os direitos — precisa vir da própria MediaTek. O de Wi-Fi entrou pelo MR !1055, e é exatamente por isso que a metade Wi-Fi do chip funciona de fábrica e a metade Bluetooth não.

Dá pra extrair dos pacotes de driver Windows da ASUS com o [extract_firmware.py](https://github.com/jetm/mediatek-mt7927-dkms) do projeto `jetm/mediatek-mt7927-dkms`. Extraí de dois pacotes diferentes, de forma independente, e os arquivos saem **byte a byte idênticos**:

| Pacote | Container | Tamanho |
|---|---|---:|
| Bluetooth V1.1147.0.610 | `mtkbt_v2.dat` | 571349 B |
| Wi-Fi V5706054 | `mtkwlan.dat` | 571349 B |

```
sha256  2135f2c4220cfa6e8eb9fdf430517098c13b862a95844bbff0153240a768efa8
caminho mediatek/mt7927/BT_RAM_CODE_MT6639_2_1_hdr.bin
build   20260611041233
```

Duas observações que economizam tempo. O caminho é `mediatek/mt7927/`, e não `mediatek/mt6639/` — confirme direto nas strings do módulo com `modinfo -F firmware btmtk` em vez de deduzir da mensagem de erro, porque essa convenção já mudou entre versões do kernel. E circula por aí um hash `669c5c99...` com uns 688 KB que não reproduziu aqui em nenhum dos dois pacotes; duas extrações independentes concordando pesam mais que um número solto de comunidade.

## O que não fazer — leia antes de replicar

- **Não rode `modprobe -r btusb`.** O firmware do MT6639 trava durante reload de módulo e o dispositivo some do `lsusb` de forma persistente. Foi assim que essa história começou. Prefira reboot.
- **Não instale arquivos `WIFI_*.bin` no caminho de firmware.** O `linux-firmware` já traz os corretos, comprimidos, em `/usr/lib/firmware/mediatek/mt7927/`. Uma cópia solta sombra silenciosamente o blob mais novo e quebra o Wi-Fi que hoje funciona. Só o arquivo de Bluetooth deve ser instalado.
- **Não espere que reboot destrave o controlador.** O trilho de standby mantém ele energizado. Só corte real de energia resolve.
- **Não deixe o loop rodar depois de ver a falha de firmware.** Cada ciclo de reset arrisca travar o chip de novo e custa mais um corte de energia. Desligue assim que aparecer `Failed to load firmware file (-2)`.

## O que deveria mudar no kernel

Um arquivo de firmware que não existe vai continuar não existindo na próxima tentativa. Retentar o mesmo pedido milhares de vezes não tem como dar certo — e aqui faz mal ativamente, porque empurra o controlador para um estado que sobrevive a reboots e exige intervenção física.

Um limite de tentativas, ou um backoff, ou simplesmente não retentar em `-2` quando a tentativa anterior falhou pelo mesmo motivo, transformaria isso de um dispositivo travado numa linha de log dizendo que o firmware está faltando. Vou levar isso pra `linux-bluetooth`.

## Conclusão

O que eu tiro dessa investigação não é o comando final, que cabe em duas linhas. São duas outras coisas.

A primeira é que ausência de evidência não é evidência de ausência, e ferramenta silenciosa mente. Um `grep` vazio me convenceu por um bom tempo de que não havia log, quando na verdade não havia permissão. Vale sempre confirmar que a ferramenta está te devolvendo dados antes de interpretar o silêncio dela.

A segunda é que o segundo susto — o `/etc` staged aparentemente vazio — se resolveu porque eu procurei uma evidência que decidisse a questão em vez de confiar no que eu achava que o ostree fazia. O `fstab` naquela lista valia mais que qualquer certeza minha sobre o funcionamento do sistema.

E, no fim, o firmware faltando era mesmo o problema. Só que ele não quebrava nada sozinho: quem quebrava era o driver, tentando resolver com afinco infinito uma coisa que não tinha como ser resolvida daquele jeito.

Se você tem esse chip e caiu aqui procurando por que seu Bluetooth não funciona, espero que este artigo economize os dias que ele me custou. Qualquer dúvida ou correção, é só me chamar.

Até a próxima!
