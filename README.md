# Guia: Instalação de Armbian em TV Box RK3228/RK3229

Procedimento completo para identificar o modelo da placa, fazer backup da firmware original e instalar o Armbian (kernel 4.4) em TV Boxes com processador RK3228a, RK3228b ou RK3229.

> Este guia é baseado no tutorial em vídeo: [https://www.youtube.com/watch?v=OMuoUVIoBBo](https://www.youtube.com/watch?v=OMuoUVIoBBo)

> ⚠️ **Atenção:** esse processo substitui o Android pelo Linux (Armbian) e envolve risco de dar errado na hora de gravar na eMMC. Faça o backup da firmware original (item 6) antes de prosseguir e siga a ordem dos passos com cuidado.
> 
> **A boa notícia:** tecnicamente, dispositivos Rockchip **não "brickam" de verdade**. Se a memória interna (flash) não tiver um sistema inicializável, o aparelho sempre tentará dar boot pelo cartão SD. Caso o sistema interno fique corrompido e nem o SD funcione, ainda é possível forçar o **modo maskrom** curto-circuitando o pino de clock da eMMC na placa — veja a seção [Recuperação em caso de falha (Unbrick)](#recupera%C3%A7%C3%A3o-em-caso-de-falha-unbrick) no final deste guia.

---

## 1. Verificação do modelo e da placa

- Confirme que a arquitetura da TV Box é **armhf**.
- Identifique se o processador é **RK3228a**, **RK3228b** ou **RK3229**.
- Verifique se a memória RAM é **DDR2** ou **DDR3** (isso influencia qual imagem baixar).
- **Importante:** essas placas **não dão boot por pendrive** — é obrigatório usar **cartão de memória (SD)**.
- Use um cartão SD com **no mínimo 8 GB** de capacidade.
- Identifique também o **tipo de memória flash interna** da placa: **NAND**, **eMMC** ou **eMCP**. Isso não depende do modelo comercial da TV Box — fabricantes usam o que for mais barato no momento da fabricação, mesmo em placas iguais.
  - **eMMC**: memória não-volátil + controlador dedicado. Mais fácil de suportar.
  - **eMCP**: igual à eMMC, mas com um banco de memória RAM (DDR) no mesmo chip.
  - **NAND**: apenas a memória, sem controlador dedicado — mais difícil de suportar e **o kernel mainline atualmente não consegue acessá-la**. Exige cuidados especiais.
  - O próprio **Multitool** detecta e avisa na tela inicial se a placa tiver um chip NAND.

---

## 2. Download e links úteis para instalação

| Arquivo | Descrição | Link |
| --- | --- | --- |
| Armbian (versões) | Imagem do sistema a ser instalado: `Armbian_21.05.1_Rk322x-box_buster_legacy_4.4.194.img.xz` (Debian Buster, kernel legacy 4.4.194). É a imagem usada neste projeto e a única em que o Wi-Fi funcionou (veja o [guia de Wi-Fi](wi-fi_settings/WI-FI_RK322x.md)). | [Download direto](https://armbian.tnahosting.net/archive/rk322x-box/archive/Armbian_21.05.1_Rk322x-box_buster_legacy_4.4.194.img.xz) · [Todas as versões](https://armbian.tnahosting.net/archive/rk322x-box/archive/) |
| Multitool (.img) | Ferramenta de boot/backup/flash para RK322x | [https://www.mediafire.com/file/2wzb3y4er4zdmld/multitool.img/file](https://www.mediafire.com/file/2wzb3y4er4zdmld/multitool.img/file) |
| balenaEtcher | Programa para gravar imagens em SD/USB | [https://etcher.balena.io/#download-etcher](https://etcher.balena.io/#download-etcher) |
| Documentação oficial Armbian | Referência geral de instalação e escolha de imagem | [https://docs.armbian.com/getting-started/](https://docs.armbian.com/getting-started/) |
| Fórum Armbian (RK322x) | Tópico da comunidade com placas e boards específicas | [https://forum.armbian.com/topic/34923-csc-armbian-for-rk322x-tv-box-boards/](https://forum.armbian.com/topic/34923-csc-armbian-for-rk322x-tv-box-boards/) |

> **Escolhendo o tipo de imagem:** o Armbian geralmente oferece três variantes por placa — **Mínima/CLI** (apenas pacotes essenciais, sem interface gráfica), **Servidor** (utilitários de servidor pré-instalados, sem interface gráfica) e **Desktop** (ambiente gráfico completo). Para algumas placas com hardware mais limitado (caso de várias TV Boxes RK322x), só a imagem mínima está disponível. Para um cluster/servidor, a variante **Mínima** ou **Servidor** costuma ser a mais indicada.
>
> **Current x Legacy:** a imagem **current** usa um kernel mais novo para aquela geração do Armbian, enquanto a **legacy** usa o kernel antigo específico daquela plataforma. Neste guia usamos a **legacy (kernel 4.4)**.

---

## 3. Gravar o Multitool no cartão SD

1. Abra o **balenaEtcher** (ou use um comando de terminal equivalente, ex: `dd`).
2. Selecione o arquivo **multitool.img**.
3. Selecione o cartão SD como destino.
4. Clique em **Flash** e aguarde a gravação terminar.

---

## 4. Verificar compatibilidade da TV Box

1. Ligue a TV Box normalmente (sem o cartão SD).
2. Conecte-a ao Wi-Fi.
3. Baixe e instale o aplicativo **AIDA64** diretamente na TV Box (Android) — não precisa ser baixado antes, só nesta etapa.
4. Verifique as especificações da máquina e confirme os critérios do **item 1**: arquitetura **armhf**, processador **RK3228a, RK3228b ou RK3229** e RAM **DDR2 ou DDR3**.

> Só avance para a próxima etapa se o processador for compatível.

---

## 5. Fazer boot pelo Multitool

1. Desligue a TV Box.
2. Insira o cartão SD com o Multitool.
3. Ligue a TV Box novamente — ela deverá dar boot pela ISO do cartão SD.
4. Um menu será exibido.

---

## 6. Backup da firmware original

1. No menu do Multitool, selecione a primeira opção: **Backup Flash**.
2. Aguarde o processo terminar.
3. Após finalizado, o backup da firmware original da TV Box estará salvo na pasta **backup** do cartão SD.

---

## 7. Preparar a nova imagem (Armbian)

1. Baixe a imagem **Armbian legacy 4.4.194** (`Armbian_21.05.1_Rk322x-box_buster_legacy_4.4.194.img.xz`, link na tabela acima).
2. No cartão SD, coloque o arquivo da imagem dentro da pasta **imagens**.

> **Dica:** sempre **ejete o cartão SD corretamente** (em vez de puxá-lo direto) para evitar corrupção dos arquivos.

---

## 8. Gravar a nova imagem na TV Box (Burn Image to Flash)

1. Insira novamente o cartão SD na TV Box e ligue-a.
2. No menu do Multitool, selecione a opção **5 — Burn Image to Flash**.
3. Selecione o **dispositivo de destino** (geralmente `mmcblk2`, que é a memória interna/eMMC da TV Box).
4. Selecione a ISO do Armbian Legacy que você colocou na pasta **imagens**.
5. Aguarde o processo de gravação terminar completamente — **não interrompa** nem desligue no meio do processo.
6. Ao finalizar, volte ao menu principal e escolha a opção **Shutdown**.
7. Só então **retire o cartão SD** e **desconecte a TV Box da fonte de energia**.

---

## 9. Primeiro boot com o novo sistema

1. Reconecte a fonte de energia — a TV Box fará boot diretamente com a nova imagem instalada (sem o cartão SD). O LED costuma piscar/mudar de cor e, após alguns segundos, o HDMI liga com as mensagens de inicialização.
2. Recomenda-se usar **cabo de rede (Ethernet)** nessa primeira inicialização, em vez de Wi-Fi. O Wi-Fi depende do chip de cada placa e só foi testado em RK3228A com chip SSV: veja o [guia de Wi-Fi](wi-fi_settings/WI-FI_RK322x.md).
3. No primeiro boot, o Armbian pedirá uma configuração inicial via terminal:
   - Defina a **senha do usuário root**.
   - Escolha o **shell padrão** (recomendado: `bash`, geralmente opção `1`).
   - Crie um **usuário comum** (nome e senha).
   - Confirme o **idioma/localização** detectado automaticamente (`y` para confirmar).
4. Após o login, use o comando `armbian-config` para ajustar fuso horário, locales e outras configurações do sistema.
5. Atualize o sistema. Esta imagem é baseada no Debian Buster, que já foi arquivado, então o `apt update` com os repositórios padrão pode falhar. Siga o guia [Configuração do APT — Armbian RK322x](apt_install/APT-ARMBIAN-RK322X.md) antes de rodar `apt upgrade`.

---

## Recuperação em caso de falha (Unbrick)

Se, por algum motivo, o sistema gravado na memória interna ficar corrompido ou não der boot corretamente, ainda é possível recuperar o aparelho na maioria dos casos:

- **Boards com eMMC ou eMCP com pinos expostos:** é possível forçar o **modo maskrom** curto-circuitando o pino de clock da eMMC na placa, usando apenas uma chave de fenda. O aparelho entra em modo de recuperação e permite regravar o bootloader/sistema.
  - [Procedimento genérico](https://forum.armbian.com/topic/12656-wip-armbian-for-rk322x-devices/?do=findComment&comment=99167) para placas com eMMC não-BGA.
- **Boards mais novas com chip eMMC/eMCP em encapsulamento BGA** (sem pinos expostos) — o curto-circuito pode ser difícil ou impossível. Nesses casos, **redobre a atenção** antes de gravar na eMMC/eMCP: sempre teste a imagem primeiro rodando pelo cartão SD, para confirmar que funciona, antes de gravar na memória interna.

**Links com pinos/procedimentos específicos por placa (exemplos da comunidade):**

- [MXQPRO_V71 (eMCP)](https://forum.armbian.com/topic/12656-csc-armbian-for-rk322x-tv-boxes/?do=findComment&comment=157061)
- [H20 (eMCP)](https://forum.armbian.com/topic/12656-csc-armbian-for-rk322x-tv-boxes/?do=findComment&comment=170092)
- [ZQ01 (eMCP)](https://forum.armbian.com/topic/34923-csc-armbian-for-rk322x-tv-box-boards/?do=findComment&comment=183472)

> ⚠️ **Caso especial — chip NAND:** se a placa usa NAND em vez de eMMC/eMCP, **evite atualizar o bootloader** a menos que você saiba exatamente o que está fazendo — o processo apaga o conteúdo existente e, se der errado, pode exigir repetir todo o procedimento de unbrick acima ou restaurar um bootloader mais antigo.

---

## Links úteis

- Tutorial base (vídeo): [https://www.youtube.com/watch?v=OMuoUVIoBBo](https://www.youtube.com/watch?v=OMuoUVIoBBo)
- Armbian Legacy (versões): [https://armbian.tnahosting.net/archive/rk322x-box/archive/](https://armbian.tnahosting.net/archive/rk322x-box/archive/)
- Multitool: [https://www.mediafire.com/file/2wzb3y4er4zdmld/multitool.img/file](https://www.mediafire.com/file/2wzb3y4er4zdmld/multitool.img/file)
- balenaEtcher: [https://etcher.balena.io/#download-etcher](https://etcher.balena.io/#download-etcher)
- Documentação oficial Armbian: [https://docs.armbian.com/getting-started/](https://docs.armbian.com/getting-started/)
- Fórum Armbian (RK322x): [https://forum.armbian.com/topic/34923-csc-armbian-for-rk322x-tv-box-boards/](https://forum.armbian.com/topic/34923-csc-armbian-for-rk322x-tv-box-boards/)
- Procedimento genérico de unbrick: [https://forum.armbian.com/topic/12656-wip-armbian-for-rk322x-devices/?do=findComment&comment=99167](https://forum.armbian.com/topic/12656-wip-armbian-for-rk322x-devices/?do=findComment&comment=99167)
- Configuração do APT (Debian Buster): [apt_install/APT-ARMBIAN-RK322X.md](apt_install/APT-ARMBIAN-RK322X.md)
- Conectar no Wi-Fi (RK3228A com chip SSV): [wi-fi_settings/WI-FI_RK322x.md](wi-fi_settings/WI-FI_RK322x.md)