# Criar uma imagem Armbian personalizada (.img.xz) para RK322x

Complemento do [guia de instalação](../README.md). Este procedimento gera uma imagem `.img.xz` leve (próxima do tamanho da imagem oficial), já com **repositórios APT corrigidos**, **driver Wi-Fi ativado** e **perfil de Wi-Fi pré-configurado**, pronta para gravar pelo Multitool (opção 5, *Burn Image to Flash*).

Vale para TV Box **RK3228A** com chip Wi-Fi **SSV6051P/6256P**, usando a imagem **legacy 4.4.194**. Outras placas podem precisar de arquivos diferentes.

## Por que não usar o backup do Multitool

O backup do Multitool (*Backup Flash*) copia a eMMC inteira, e a imagem fica grande (cerca de 1 GB mesmo comprimida), porque leva junto:

- os pacotes atualizados com `apt upgrade`;
- usuário, senhas e chaves SSH da instalação;
- logs, cache e espaço livre.

Além disso, a imagem de backup não mostra mais o assistente de primeiro boot, porque o usuário já existe.

A estratégia usada aqui é outra: partir da **imagem oficial limpa** e copiar para dentro dela **somente as configurações** (arquivos de texto de poucos KB). O resultado mantém o tamanho da imagem oficial e não leva nenhum dado pessoal.

## O que vai para dentro da imagem

| Arquivo | Para quê |
|---|---|
| `/etc/apt/sources.list` | Repositórios apontando para `archive.debian.org` (o Debian Buster foi arquivado) |
| `/etc/apt/apt.conf.d/99archive-debian` | Ignora a validade temporal dos metadados do repositório arquivado |
| `/etc/modprobe.d/blacklist-rk322x-box.conf` | Libera o driver `ssv6x5x` (ele vem bloqueado na imagem limpa) |
| `/boot/armbianEnv.txt` | Duas linhas: `overlays=emmc led-conf1` e `usbstoragequirks=...` |
| `/etc/NetworkManager/system-connections/*.nmconnection` | Perfil do Wi-Fi |

Fica **de fora**: pacotes atualizados, `passwd`, `shadow`, chaves SSH, `machine-id`, `initrd`/`uInitrd` regenerados, `oh-my-zsh` e qualquer arquivo ligado ao usuário. Os serviços `armbian-firstrun` e `armbian-resize-filesystem` continuam na imagem, então o assistente de primeiro boot e a expansão da partição seguem funcionando.

## Requisitos

- Um PC com Linux (ou WSL) com `xz`, `losetup`, `mount`, `diff`, `ssh` e `scp`.
- A TV Box já configurada (APT e Wi-Fi funcionando) e acessível por SSH.
- Cerca de 4 GB livres no PC.

## 1. Baixar a imagem base

Use a imagem **legacy 4.4.194**. A imagem com kernel 5.10.12 (mainline) não tem driver para o chip Wi-Fi SSV.

```bash
cd ~
wget https://armbian.tnahosting.net/archive/rk322x-box/archive/Armbian_21.05.1_Rk322x-box_buster_legacy_4.4.194.img.xz
```

## 2. Copiar `/etc` e `/boot` da TV Box para o PC

Descubra o IP da TV Box (no terminal dela):

```bash
ip a show wlan0     # ou eth0, se estiver no cabo
```

Conecte pelo PC e gere o pacote:

```bash
ssh SEU_USUARIO@IP_DA_TV_BOX
sudo tar czf /tmp/etc-boot.tgz /etc /boot
sudo chown $USER:$USER /tmp/etc-boot.tgz
exit
```

Traga o arquivo para o PC e apague-o da TV Box:

```bash
scp SEU_USUARIO@IP_DA_TV_BOX:/tmp/etc-boot.tgz ~/
ssh SEU_USUARIO@IP_DA_TV_BOX "rm /tmp/etc-boot.tgz"
```

> **Cuidado:** esse arquivo contém hashes de senha, chaves SSH e a senha do Wi-Fi. Guarde só no seu PC, não envie ao GitHub e apague quando terminar.

Extraia em uma pasta:

```bash
mkdir ~/tvbox
tar xzf ~/etc-boot.tgz -C ~/tvbox
```

## 3. Montar a imagem oficial

```bash
xz -dk Armbian_21.05.1_Rk322x-box_buster_legacy_4.4.194.img.xz
LOOP=$(sudo losetup -fP --show Armbian_21.05.1_Rk322x-box_buster_legacy_4.4.194.img)
echo $LOOP
lsblk $LOOP
sudo mount ${LOOP}p1 /mnt
ls /mnt
```

Use sempre o dispositivo que o `losetup` devolver (por exemplo `/dev/loop55`). O `/dev/loop0` costuma estar ocupado por pacotes snap. A imagem tem uma partição só (`p1`), e o `/boot` fica dentro dela.

## 4. Descobrir o que mudou

Compare a imagem limpa com o backup da TV Box:

```bash
diff -rq /mnt/etc ~/tvbox/etc > ~/diff-etc.txt 2>/dev/null
diff -rq /mnt/boot ~/tvbox/boot > ~/diff-boot.txt 2>/dev/null
grep -iE 'apt|NetworkManager|system-connections|modprobe|modules|ssv|rk322x|wpa|netplan|network|armbian|hostname' ~/diff-etc.txt
cat ~/diff-boot.txt
```

A lista é grande e cheia de ruído do `apt upgrade` (`oh-my-zsh`, certificados, links do `systemd`). Os arquivos que importam são os da tabela acima. Veja o conteúdo de cada um antes de copiar:

```bash
diff /mnt/etc/apt/sources.list ~/tvbox/etc/apt/sources.list
diff /mnt/boot/armbianEnv.txt ~/tvbox/boot/armbianEnv.txt
diff /mnt/etc/modprobe.d/blacklist-rk322x-box.conf ~/tvbox/etc/modprobe.d/blacklist-rk322x-box.conf
```

O resultado esperado:

- **`sources.list`:** a imagem limpa usa `deb.debian.org`, que não serve mais o Buster. O seu usa `archive.debian.org`, sem `buster-backports` (que dá erro de chave `NO_PUBKEY`).
- **`blacklist-rk322x-box.conf`:** na imagem limpa existe `blacklist ssv6x5x`. No da TV Box ela está comentada (`#blacklist ssv6x5x`), o que libera o driver e faz o `wlan0` aparecer. Aparecem também `blacklist r8188eu` e `blacklist 8189es`, adicionados pelo `rk322x-config`.
- **`armbianEnv.txt`:** só duas linhas a mais (`overlays=...` e `usbstoragequirks=...`). Não sobrescreva o arquivo inteiro: o `rootdev` e as demais linhas devem continuar os da imagem.

## 5. Copiar as configurações para a imagem

```bash
# repositórios
sudo cp ~/tvbox/etc/apt/sources.list /mnt/etc/apt/sources.list
echo 'Acquire::Check-Valid-Until "false";' | sudo tee /mnt/etc/apt/apt.conf.d/99archive-debian

# driver Wi-Fi
sudo cp ~/tvbox/etc/modprobe.d/blacklist-rk322x-box.conf /mnt/etc/modprobe.d/blacklist-rk322x-box.conf

# armbianEnv.txt: acrescenta as duas linhas sem sobrescrever o arquivo
sudo sed -i -e '$a\' /mnt/boot/armbianEnv.txt
grep -q '^overlays=' /mnt/boot/armbianEnv.txt || echo 'overlays=emmc led-conf1' | sudo tee -a /mnt/boot/armbianEnv.txt
grep -q '^usbstoragequirks=' /mnt/boot/armbianEnv.txt || echo 'usbstoragequirks=0x2537:0x1066:u,0x2537:0x1068:u' | sudo tee -a /mnt/boot/armbianEnv.txt
```

> `overlays=emmc led-conf1` é específico de placa com flash **eMMC** e LED **led-conf1**. Em outra placa, ajuste com `sudo rk322x-config` depois do boot.

### Perfil do Wi-Fi

O chip SSV só funciona em **2,4 GHz**. Leve apenas o perfil da rede 2,4 GHz.

```bash
ls ~/tvbox/etc/NetworkManager/system-connections/
```

Copie o arquivo (troque `NOME_DO_PERFIL`) e remova a linha de MAC, se houver, para o perfil não ficar preso à placa de origem:

```bash
sudo mkdir -p /mnt/etc/NetworkManager/system-connections
sudo cp ~/tvbox/etc/NetworkManager/system-connections/NOME_DO_PERFIL.nmconnection /mnt/etc/NetworkManager/system-connections/
sudo sed -i '/^mac-address=/d' /mnt/etc/NetworkManager/system-connections/NOME_DO_PERFIL.nmconnection
sudo chown root:root /mnt/etc/NetworkManager/system-connections/NOME_DO_PERFIL.nmconnection
sudo chmod 600 /mnt/etc/NetworkManager/system-connections/NOME_DO_PERFIL.nmconnection
```

As permissões `600` são obrigatórias: sem elas o NetworkManager ignora o perfil.

## 6. Conferir, desmontar e comprimir

```bash
cat /mnt/boot/armbianEnv.txt
cat /mnt/etc/modprobe.d/blacklist-rk322x-box.conf
cat /mnt/etc/apt/sources.list
sudo ls -l /mnt/etc/NetworkManager/system-connections/

sudo umount /mnt
sudo losetup -d $LOOP

xz -9e -T0 -c Armbian_21.05.1_Rk322x-box_buster_legacy_4.4.194.img > Armbian_RK3228A_legacy_custom.img.xz
ls -lh Armbian_RK3228A_legacy_custom.img.xz
```

O `-c` com redirecionamento evita que o `xz` tente sobrescrever o arquivo original. O tamanho final fica próximo ao da imagem oficial (na faixa de 200 a 300 MB).

## 7. Gravar e testar

1. Copie o `.img.xz` para a pasta `imagens` do cartão SD com o Multitool.
2. **Teste primeiro dando boot pelo cartão SD**, antes de gravar na eMMC.
3. Grave pela opção **5 (Burn Image to Flash)** do Multitool.
4. No primeiro boot, o assistente pede senha de root, shell e usuário comum.
5. Verifique:

```bash
ip link                  # deve existir wlan0
nmcli device status      # wlan0 conectado
sudo apt update          # sem erro de repositório
```

Se a data estiver errada, o HTTPS falha. Sincronize pela rede:

```bash
sudo timedatectl set-ntp true
```

## Cuidados

- **Senha do Wi-Fi:** o perfil guarda a senha em texto dentro da imagem. Não publique a imagem em lugar público. Para distribuir, gere a imagem **sem** o perfil e deixe cada pessoa conectar com o `nmtui` (veja [wi-fi_settings](../wi-fi_settings/)).
- **Kernel:** o Wi-Fi depende do kernel 4.4.194. Depois do primeiro `apt upgrade`, confira a lista antes de aceitar e bloqueie pacotes de kernel com `sudo apt-mark hold NOME_DO_PACOTE` (os nomes costumam começar com `linux-image`, `linux-dtb` ou `armbian-firmware`).
- **Atenção ao `.gitignore`:** não suba para o repositório os arquivos `*.img`, `*.img.xz`, `*.tgz` nem as pastas de diff e backup. Sugestão:

```gitignore
*.img
*.img.xz
*.img.gz
*.tgz
etc-boot*
tvbox/
diff-*.txt
```

## Resumo do processo

```text
Imagem oficial legacy 4.4.194 (limpa)
        ↓
Montar com losetup
        ↓
Comparar com /etc e /boot da TV Box (diff)
        ↓
Copiar só: sources.list, apt.conf.d, blacklist, armbianEnv.txt, perfil Wi-Fi
        ↓
Desmontar e comprimir com xz -9e
        ↓
.img.xz → pasta imagens do SD → Multitool (opção 5)
```