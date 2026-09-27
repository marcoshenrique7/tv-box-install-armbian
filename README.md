# Guia: Instalação de Armbian Legacy em TV Box RK3228/RK3229

Procedimento completo para identificar o modelo da placa, fazer backup da firmware original e instalar o Armbian Legacy (kernel 4.4) em TV Boxes com processador RK3228a, RK3228b ou RK3229.

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

## 2. Download dos arquivos necessários

| Arquivo | Descrição | Link |
| --- | --- | --- |
| Armbian Legacy (versões) | Imagem do sistema a ser instalado (usar 21.08.1, kernel 4.4) | [https://armbian.tnahosting.net/archive/rk322x-box/archive/](https://armbian.tnahosting.net/archive/rk322x-box/archive/) |
| Multitool (.img) | Ferramenta de boot/backup/flash para RK322x | [https://www.mediafire.com/file/2wzb3y4er4zdmld/multitool.img/file](https://www.mediafire.com/file/2wzb3y4er4zdmld/multitool.img/file) |
| balenaEtcher | Programa para gravar imagens em SD/USB | [https://etcher.balena.io/#download-etcher](https://etcher.balena.io/#download-etcher) |
| Documentação oficial Armbian | Referência geral de instalação e escolha de imagem | [https://docs.armbian.com/getting-started/](https://docs.armbian.com/getting-started/) |
| Fórum Armbian (RK322x) | Tópico da comunidade com placas e boards específicas | [https://forum.armbian.com/topic/34923-csc-armbian-for-rk322x-tv-box-boards/](https://forum.armbian.com/topic/34923-csc-armbian-for-rk322x-tv-box-boards/) |

> **Escolhendo o tipo de imagem:** o Armbian geralmente oferece três variantes por placa — **Mínima/CLI** (apenas pacotes essenciais, sem interface gráfica), **Servidor** (utilitários de servidor pré-instalados, sem interface gráfica) e **Desktop** (ambiente gráfico completo). Para algumas placas com hardware mais limitado (caso de várias TV Boxes RK322x), só a imagem mínima está disponível. Para um cluster/servidor, a variante **Mínima** ou **Servidor** costuma ser a mais indicada.

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
4. Verifique as especificações da máquina (processador, RAM, etc.) e confirme se são compatíveis com o Armbian Legacy.

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

## 7. Preparar a nova imagem (Armbian Legacy)

1. Baixe a imagem **Armbian Legacy** (link na tabela acima).
2. No cartão SD, coloque a ISO na pasta **imagens**.
3. Coloque o arquivo da ISO dentro dessa pasta.

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
2. Recomenda-se usar **cabo de rede (Ethernet)** nessa primeira inicialização, em vez de Wi-Fi.
3. No primeiro boot, o Armbian pedirá uma configuração inicial via terminal:
   - Defina a **senha do usuário root**.
   - Escolha o **shell padrão** (recomendado: `bash`, geralmente opção `1`).
   - Crie um **usuário comum** (nome e senha).
   - Confirme o **idioma/localização** detectado automaticamente (`y` para confirmar).
4. Após o login, use o comando `armbian-config` para ajustar fuso horário, locales e outras configurações do sistema.
5. Atualize o sistema:

   ```
   sudo apt update && sudo apt upgrade
   ```

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


## Dicionário de termos técnicos

| Termo | Significado |
| --- | --- |
| **Armbian** | Distribuição Linux (baseada em Debian/Ubuntu) otimizada para placas ARM, como Raspberry Pi e TV Boxes com Rockchip. |
| **Legacy** | Versão do Armbian que usa um kernel mais antigo (neste caso, 4.4), geralmente com melhor compatibilidade de hardware do que kernels mais novos ("mainline") em placas mais antigas. |
| **Kernel** | O "núcleo" do sistema operacional — o software que faz a ponte entre o hardware da placa e os programas/aplicativos. |
| **armhf** | "ARM Hard Float" — arquitetura de processador ARM de 32 bits usada nessas TV Boxes. É preciso baixar uma imagem compatível com essa arquitetura, não com ARM64 ou x86. |
| **RK3228a / RK3228b / RK3229** | Modelos específicos de processador (chipset) da fabricante Rockchip, usados nessas TV Boxes. Placas com processadores diferentes podem precisar de imagens diferentes. |
| **DDR2 / DDR3** | Tipos de memória RAM. A imagem do Armbian precisa ser compatível com o tipo de RAM da placa. |
| **Firmware** | O sistema/software original que já vem instalado de fábrica na TV Box (geralmente Android). |
| **Flash (memória)** | Memória não-volátil onde o sistema operacional fica armazenado permanentemente (equivalente a um "HD" da TV Box). |
| **Flash (gravar)** | O ato de gravar uma imagem (.img) em um cartão SD, pendrive ou memória interna — usado tanto como verbo ("dar flash") quanto no nome do programa balenaEtcher. |
| **eMMC** | Tipo de chip de memória flash interna que já vem com um controlador dedicado — mais fácil de o sistema acessar e gravar. |
| **eMCP** | Igual à eMMC, mas com um chip extra de memória RAM (DDR) embutido no mesmo componente. |
| **NAND** | Tipo de chip de memória flash mais simples, sem controlador dedicado — mais difícil de suportar por software e não acessível pelo kernel mainline do Armbian. |
| **BGA (Ball Grid Array)** | Tipo de encapsulamento de chip onde os contatos ficam embaixo do componente, soldados diretamente na placa — sem pinos expostos para acesso externo. |
| **mmcblk2** | Nome que o Linux dá ao dispositivo de armazenamento correspondente à memória interna (eMMC) da TV Box — é o "destino" selecionado ao gravar o sistema na memória interna. |
| **Multitool** | Ferramenta (imagem de boot) usada para fazer backup da firmware original, testar e gravar imagens do Armbian nessas TV Boxes Rockchip. |
| **balenaEtcher** | Programa gratuito usado para gravar arquivos de imagem (.img, .iso) em cartões SD ou pendrives. |
| **ISO / IMG** | Formatos de arquivo de imagem de disco — contêm uma cópia completa de um sistema operacional pronta para ser gravada em uma mídia (SD, pendrive, etc.). |
| **Boot** | O processo de inicialização de um dispositivo — carregar e executar o sistema operacional ao ligá-lo. |
| **Bootloader** | Pequeno programa que roda antes do sistema operacional e é responsável por carregá-lo corretamente na inicialização. |
| **Maskrom** | Modo especial de recuperação dos processadores Rockchip, acessado via hardware, que permite regravar o bootloader/sistema mesmo quando a placa não consegue dar boot normalmente. |
| **Brick / Unbrick** | "Brickar" é inutilizar um dispositivo (deixá-lo "duro como um tijolo"); "unbrick" é o processo de recuperá-lo. |
| **eMMC clock pin** | Pino físico na placa ligado ao clock (sincronismo) do chip eMMC; curto-circuitá-lo momentaneamente força o processador a entrar em modo maskrom. |
| **root** | Usuário administrador do sistema Linux, com acesso total para instalar, remover e configurar qualquer coisa. |
| **Shell** | Programa de interface de linha de comando usado para interagir com o sistema (ex: `bash`). |
| **armbian-config** | Ferramenta de configuração do Armbian via terminal, usada para ajustar fuso horário, idioma (locale), rede, entre outros. |
| **Locale** | Configuração de idioma/região do sistema (formato de data, idioma dos menus, etc.). |
| **Ethernet** | Conexão de rede via cabo (em vez de Wi-Fi) — mais estável para a primeira configuração do sistema. |