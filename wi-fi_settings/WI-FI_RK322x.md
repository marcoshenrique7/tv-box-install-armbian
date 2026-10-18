# Conectar no Wi-Fi (RK322x com chip SSV, Armbian legacy 4.4.194)

Complemento do [guia de instalação](../README.md). Vale para TV Box **RK3228A** com chip Wi-Fi **South Silicon Valley 6051p/6256p** (SDIO ID `3030:3030`). Outras placas podem ter outro chip Wi-Fi, e este guia não se aplica a elas.

## Requisito importante

O Wi-Fi só funcionou na imagem com **kernel legacy 4.4.194**. Na imagem com kernel 5.10.12 (mainline) o kernel não tem driver para esse chip:

- `ip link` não mostra `wlan0`
- `sudo modprobe ssv6051` retorna `Module ssv6051 not found`
- `grep -i ssv /boot/config-5.10.12-rk322x` não retorna nada

Imagem usada:

```
https://armbian.tnahosting.net/archive/rk322x-box/archive/Armbian_21.05.1_Rk322x-box_buster_legacy_4.4.194.img.xz
```

Confira o kernel antes de começar:

```bash
uname -r
```

Deve mostrar `4.4.194-rk322x`.

## 1. Escolher o driver no rk322x-config

```bash
sudo rk322x-config
```

Selecione SoC `RK3228A`, flash `emmc`, LED `led-conf1` e, na tela do driver do Wi-Fi, o driver da família **ssv6x5x**. Depois:

```bash
sudo reboot
```

## 2. Verificar a interface

```bash
ip link
```

Deve existir a interface `wlan0`.

## 3. Conectar na rede

**Pelo terminal:**

```bash
sudo nmcli radio wifi on
sudo nmcli device wifi rescan
nmcli device wifi list
sudo nmcli device wifi connect "NOME_DA_REDE" password "SENHA"
```

**Ou pela interface em texto:**

```bash
sudo nmtui
```

Como navegar no `nmtui`: use as **setas** para mover, **Enter** para selecionar, **Tab** para alternar entre campos e botões, **Espaço** para marcar/desmarcar uma caixa e **Esc** para voltar.

**Caminho A: a rede aparece na lista (o mais simples)**

1. No menu inicial, selecione **Ativar uma conexão** (*Activate a connection*) e aperte Enter.
2. Aparece uma lista com as redes Wi-Fi encontradas. Selecione a sua.
3. Vá até o botão **Ativar** (*Activate*) e aperte Enter.
4. Digite a senha da rede quando pedir e confirme com **OK**.
5. Aguarde alguns segundos. Ao lado do nome da rede aparece um `*`, indicando que está conectado.

Se só aparecer "Wired connection 1" e nenhuma rede Wi-Fi, não há interface Wi-Fi funcionando: volte ao passo 1.

**Caminho B: criar o perfil manualmente (se a rede não aparece na lista, ou está oculta)**

1. No menu inicial, selecione **Editar uma conexão** (*Edit a connection*) e aperte Enter.
2. Selecione **`<Adicionar>`** (*Add*) e escolha o tipo **Wi-Fi**.
3. Preencha o formulário:

| Campo | O que colocar |
|---|---|
| Profile name | Qualquer nome, por exemplo `Wi-Fi connection 1` |
| Device | `wlan0` |
| SSID | O nome exato da rede (maiúsculas e minúsculas contam) |
| Mode | `Client` (já vem assim) |
| Security | Aperte Enter no campo, escolha **WPA & WPA2 Personal** e confirme. O padrão é `None`, que não funciona em rede com senha |
| Password | Aparece depois de escolher a segurança: digite a senha da rede |
| Automatically connect | Deixe marcado `[X]` para reconectar sozinho a cada boot |

4. Desça até o final do formulário, selecione **`<OK>`** e aperte Enter.
5. Volte ao menu inicial com **Esc** (ou `<Voltar>`), abra **Ativar uma conexão**, selecione o perfil criado e ative.

Se a rede for antiga e não conectar com WPA & WPA2 Personal, tente **WPA2 Personal** ou confira a senha.

## 4. Confirmar

```bash
nmcli device status
ip a show wlan0
ping -c 4 8.8.8.8
```

O `wlan0` deve aparecer como `connected`. Teste também sem o cabo de rede.

## Problemas comuns

**`wlan0` existe, mas fica DOWN / `unavailable` e o `dmesg` mostra `Initialize ssv6200 mac fail`:**
o driver carregado é o do ssv6051, que não consegue iniciar esse chip (ele se identifica como `SSV6006C`). Volte ao passo 1, escolha o driver `ssv6x5x` e reinicie.

**Nenhuma rede aparece:**
o chip só suporta 2,4 GHz. Use a rede de 2,4 GHz, que não pode estar oculta. Rode `sudo nmcli device wifi rescan`, espere uns 10 segundos e liste de novo.

**Conecta, mas `ping google.com` falha e `ping 8.8.8.8` funciona (DNS):**

```bash
sudo nmcli connection modify "Wi-Fi connection 1" ipv4.dns "8.8.8.8 1.1.1.1"
sudo nmcli connection modify "Wi-Fi connection 1" ipv4.ignore-auto-dns yes
sudo nmcli connection down "Wi-Fi connection 1"
sudo nmcli connection up "Wi-Fi connection 1"
```

## Alternativa

Um adaptador USB Wi-Fi compatível com Linux (chipset Mediatek MT7601U ou Realtek RTL8188) costuma funcionar sem driver extra.