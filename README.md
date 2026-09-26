# Instalando OpenWrt em TV Box RK3228a / RK3228b / RK3229 (armhf)

Guia passo a passo para transformar uma TV Box baseada nos SoCs **Rockchip RK3228a, RK3228b ou RK3229** em um roteador/AP rodando **OpenWrt Legacy 21.08.1 (kernel 4.4)**, usando boot via **cartão de memória** (não via pendrive).

> ⚠️ **Aviso:** este processo pode inutilizar (brickar) sua TV Box se feito incorretamente. Faça por sua conta e risco. Tenha sempre um método de recuperação (imagem stock, cabo USB-OTG, etc.) antes de começar.

---

## 1. Pré-requisitos

- TV Box com SoC **RK3228a**, **RK3228b** ou **RK3229**
- Memória **DDR2** ou **DDR3** (varia conforme o modelo da placa)
- Arquitetura de destino: **armhf**
- Cartão de memória (microSD) — **boot via pendrive não funciona nesses SoCs**, é obrigatório usar cartão
- Computador com Windows/Linux/macOS para gravar a imagem
- Cabo de rede (recomendado durante os testes iniciais)

---

## 2. Identifique o modelo da sua placa

Antes de baixar qualquer imagem, é fundamental confirmar qual chip e qual tipo de memória sua TV Box utiliza, pois isso influencia diretamente na compatibilidade da imagem:

- **RK3228a** — DDR2 ou DDR3 (varia por fabricante)
- **RK3228b** — DDR2 ou DDR3 (varia por fabricante)
- **RK3229** — DDR2 ou DDR3 (varia por fabricante)

Se você não sabe o modelo exato, abra a carcaça e verifique o chip gravado na placa, ou pesquise o modelo/número de série da TV Box.

---

## 3. Baixe os arquivos necessários

1. **Imagem OpenWrt Legacy 21.08.1 (kernel 4.4)**
   Baixe a ISO/imagem legacy compatível com os SoCs RK322x/RK3229 (armhf).

2. **MultiTool (imagem .img)**
   Baixe a imagem do MultiTool, usada para identificar e preparar o hardware antes da gravação definitiva.

3. **balenaEtcher**
   Baixe o instalador do balenaEtcher para o seu sistema operacional.

---

## 4. Grave o MultiTool no cartão de memória

1. Insira o cartão microSD no computador.
2. Abra o **balenaEtcher** (ou use o terminal, se preferir).
3. Selecione o arquivo `.img` do **MultiTool**.
4. Selecione o cartão de memória como destino.
5. Clique em **Flash!** e aguarde a conclusão.

> Alternativa via terminal (Linux):
> ```bash
> sudo dd if=multitool.img of=/dev/sdX bs=4M status=progress conv=fsync
> ```
> Substitua `/dev/sdX` pelo dispositivo correto do seu cartão (confira com `lsblk` antes de executar, para não sobrescrever o disco errado).

---

## 5. Primeiro boot com o MultiTool

1. Insira o cartão gravado com o MultiTool na TV Box.
2. Ligue a TV Box (boot pelo cartão de memória).
3. Conecte a TV Box ao Wi-Fi.
4. Dentro do sistema, instale/abra o aplicativo **AIDA64**.
5. Verifique nas especificações do AIDA64:
   - Modelo exato do processador (RK3228a / RK3228b / RK3229)
   - Tipo de memória RAM (DDR2 ou DDR3)
   - Quantidade de RAM e armazenamento

Esses dados confirmam se sua placa é compatível com a imagem que será gravada na próxima etapa.

---

## 6. Grave a imagem definitiva do OpenWrt

Com o processador e a memória confirmados como compatíveis:

1. Abra o **balenaEtcher** novamente.
2. Selecione a imagem **OpenWrt Legacy 21.08.1 (kernel 4.4)** baixada anteriormente.
3. Selecione o cartão de memória como destino (o mesmo utilizado no MultiTool).
4. Clique em **Flash!** e aguarde a conclusão da gravação.

> Alternativa via terminal (Linux):
> ```bash
> sudo dd if=openwrt-21.08.1-legacy.img of=/dev/sdX bs=4M status=progress conv=fsync
> ```

---

## 7. Boot final

1. Insira o cartão com a imagem do OpenWrt na TV Box.
2. Ligue o dispositivo (boot via cartão de memória).
3. Aguarde a inicialização do OpenWrt.
4. Acesse a interface **LuCI** pelo navegador (geralmente em `192.168.1.1`) para confirmar que tudo está funcionando.

---

## Resumo do fluxo

```
Identificar SoC (RK3228a/b ou RK3229) e memória (DDR2/DDR3)
        │
        ▼
Baixar: ISO OpenWrt Legacy 21.08.1 (kernel 4.4) + MultiTool + balenaEtcher
        │
        ▼
Gravar MultiTool no cartão de memória (balenaEtcher ou dd)
        │
        ▼
Boot na TV Box → conectar Wi-Fi → checar specs no AIDA64
        │
        ▼
Confirmar compatibilidade do processador/memória
        │
        ▼
Gravar imagem definitiva do OpenWrt no cartão
        │
        ▼
Boot final via cartão de memória → acessar LuCI
```

---

## Observações finais

- O boot via **pendrive não é suportado** nesses SoCs — use sempre cartão de memória.
- Sempre confira o dispositivo correto (`/dev/sdX`) antes de usar `dd`, para evitar apagar o disco errado.
- Guarde a imagem original/stock da sua TV Box antes de iniciar, para recuperação em caso de erro.