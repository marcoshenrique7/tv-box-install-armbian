# Guia: Instalação de Armbian Legacy em TV Box RK3228/RK3229

Procedimento completo para identificar o modelo da placa, fazer backup da firmware original e instalar o Armbian Legacy (kernel 4.4) em TV Boxes com processador RK3228a, RK3228b ou RK3229.

> Este guia é baseado no tutorial em vídeo: [https://www.youtube.com/watch?v=OMuoUVIoBBo](https://www.youtube.com/watch?v=OMuoUVIoBBo)

> ⚠️ **Atenção:** esse processo substitui o Android pelo Linux (Armbian) e envolve risco real de **"brickar"** (inutilizar) a TV Box se algum passo for feito errado — especialmente na hora de gravar na eMMC. Faça o backup da firmware original (item 6) antes de prosseguir e siga a ordem dos passos com cuidado.

---

## 1. Verificação do modelo e da placa

- Confirme que a arquitetura da TV Box é **armhf**.
- Identifique se o processador é **RK3228a**, **RK3228b** ou **RK3229**.
- Verifique se a memória RAM é **DDR2** ou **DDR3** (isso influencia qual imagem baixar).
- **Importante:** essas placas **não dão boot por pendrive** — é obrigatório usar **cartão de memória (SD)**.
- Use um cartão SD com **no mínimo 8 GB** de capacidade.

---

## 2. Download dos arquivos necessários

| Arquivo | Descrição | Link |
| --- | --- | --- |
| Armbian Legacy (versões) | Imagem do sistema a ser instalado (usar 21.08.1, kernel 4.4) | [https://armbian.tnahosting.net/archive/rk322x-box/archive/](https://armbian.tnahosting.net/archive/rk322x-box/archive/) |
| Multitool (.img) | Ferramenta de boot/backup/flash para RK322x | [https://www.mediafire.com/file/2wzb3y4er4zdmld/multitool.img/file](https://www.mediafire.com/file/2wzb3y4er4zdmld/multitool.img/file) |
| balenaEtcher | Programa para gravar imagens em SD/USB | *\[adicionar link\]* |

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
3. Baixe e instale o aplicativo **AIDA64** diretamente na TV Box (Android) — não precisa ser baixado antes, só nesta etapa. Link: *\[adicionar link\]*
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
2. No cartão SD, crie uma pasta chamada **imagens**.
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

## Links úteis

- Tutorial base (vídeo): [https://www.youtube.com/watch?v=OMuoUVIoBBo](https://www.youtube.com/watch?v=OMuoUVIoBBo)
- Armbian Legacy (versões): [https://armbian.tnahosting.net/archive/rk322x-box/archive/](https://armbian.tnahosting.net/archive/rk322x-box/archive/)
- Multitool: [https://www.mediafire.com/file/2wzb3y4er4zdmld/multitool.img/file](https://www.mediafire.com/file/2wzb3y4er4zdmld/multitool.img/file)
- balenaEtcher: *\[adicionar link\]*
- AIDA64: *\[adicionar link\]*