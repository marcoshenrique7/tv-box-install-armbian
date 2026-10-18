# Cluster k3s com TV Boxes (RK322x) via Wi-Fi

Variante do projeto em que as **12 TV boxes** (chip RK3228/RK3229, **1 GB de RAM cada**) se conectam ao cluster por **Wi-Fi**, em vez de cabo. O sistema é o **Armbian**, o gerenciador de containers é o **k3s** (Kubernetes leve) e a configuração é automatizada com **Ansible**.

Esta pasta explica o que será feito, em que ordem, com quais ferramentas e por que essas escolhas foram feitas. O objetivo é que qualquer pessoa consiga repetir o projeto.

> **Status:** planejamento. Há duas coisas a validar na primeira box, antes de preparar as outras 11: se o **k3s roda no kernel antigo (4.4)** e se o **Wi-Fi é estável o bastante** para o cluster.

> **Atenção:** o Wi-Fi é a parte mais frágil desta variante. Ele só foi testado em boxes **RK3228A com chip SSV 6051p/6256p**, e só funciona na imagem **legacy 4.4.194**. Veja a seção de limitações.

---

## Visão geral da arquitetura

- **1 master (servidor k3s):** uma máquina com **pelo menos 2 núcleos e 2 GB de RAM**, fora das TV boxes (PC antigo, notebook, Raspberry Pi 4 ou VM). Recomenda-se ligá-la **por cabo** ao roteador, para ela não ser mais um ponto de instabilidade.
- **12 workers (agentes k3s):** as TV boxes com Armbian, nomeadas de `tvbox01` a `tvbox12`, conectadas por **Wi-Fi 2,4 GHz**.
- **Roteador / ponto de acesso:** de preferência dedicado ao cluster, com suporte a pelo menos 13 dispositivos conectados e com uma rede de 2,4 GHz **visível** (não oculta).
- **IP fixo:** reserva de DHCP no roteador, pelo endereço MAC da interface Wi-Fi (`wlan0`) de cada box.
- **Máquina de controle:** o computador de onde se roda o Ansible e o `kubectl` (pode ser o próprio master).

```
                 ┌──────────────┐
                 │  k3s master  │  (≥ 2 GB de RAM, por cabo)
                 └──────┬───────┘
                        │ cabo
                 ┌──────┴───────┐
                 │ Roteador / AP│  Wi-Fi 2,4 GHz
                 └──────┬───────┘
        ┌───────────┬───┴───────┬───────────┐
   ┌────┴──┐   ┌────┴──┐     ...      ┌────┴──┐
   │tvbox01│   │tvbox02│              │tvbox12│  (workers, 1 GB cada)
   └───────┘   └───────┘              └───────┘
```

## Materiais

- 12 TV boxes com chip **RK3228A** e Wi-Fi **SSV 6051p/6256p** (ou adaptador USB Wi-Fi compatível com Linux nas que não tiverem).
- Cartões SD de 8 GB ou mais (um só pode ser reaproveitado, gravando uma box por vez).
- 1 máquina para o master, com 2 GB de RAM ou mais.
- Roteador ou ponto de acesso de 2,4 GHz que aguente 13 ou mais dispositivos.
- Um cabo de rede para o master e **um cabo de rede para a configuração inicial das boxes** (o guia recomenda Ethernet no primeiro boot).
- 12 fontes de energia estáveis.
- Teclado e TV/monitor com HDMI para a configuração inicial.

---

## Passo a passo

### Etapa 1: Levantamento do hardware

**O que fazer:** identificar, em cada box, o chip (RK3228a, RK3228b ou RK3229), o tipo de RAM (DDR2 ou DDR3), a memória interna (eMMC, eMCP ou NAND) e **o chip de Wi-Fi**.

**O que será usado:** aplicativo **AIDA64** (no Android da box) e o **Multitool**, que avisa na tela inicial se a placa tem NAND.

**Por que:** o chip e a memória interna definem qual imagem usar e o risco da instalação. O chip de Wi-Fi define se a box conecta sem adaptador extra: o guia de Wi-Fi do projeto só vale para o SSV 6051p/6256p.

**Links:**
- Guia de instalação do Armbian (itens 1 a 4): [../README.md](../README.md)
- Fórum Armbian para TV boxes RK322x: https://forum.armbian.com/topic/34923-csc-armbian-for-rk322x-tv-box-boards/

### Etapa 2: Instalação do Armbian (começando por uma única box)

**O que fazer:** gravar o Multitool no cartão SD, fazer o **backup da firmware original** e gravar o Armbian na memória interna.

**O que será usado:**
- **Multitool** e **balenaEtcher**.
- Imagem **`Armbian_21.05.1_Rk322x-box_buster_legacy_4.4.194.img.xz`** (Debian Buster, kernel legacy 4.4.194).

**Por que:** o Android original não serve para rodar um servidor, e o Armbian tem suporte da comunidade para RK322x. Nesta variante a imagem **legacy é obrigatória**, porque o kernel 5.10 não tem driver para o chip de Wi-Fi SSV.

**Links:**
- Passo a passo completo: [../README.md](../README.md)
- Imagens do Armbian para RK322x: https://armbian.tnahosting.net/archive/rk322x-box/archive/
- balenaEtcher: https://etcher.balena.io/#download-etcher
- Documentação oficial do Armbian: https://docs.armbian.com/getting-started/

### Etapa 3: Preparação do sistema em cada nó

**O que fazer:**
1. Fazer o primeiro boot **com cabo de rede** e criar o usuário.
2. Corrigir o APT: o Debian Buster está arquivado, então os repositórios precisam apontar para `archive.debian.org`.
3. Acertar a data do sistema.
4. Desativar o swap (inclusive o zram do Armbian, se estiver ativo; confira com `swapon --show`).
5. Verificar os cgroups, em especial o de memória: `cat /proc/cgroups`.
6. Definir hostname único e SSH por chave.

**O que será usado:** `apt`, `armbian-config` e comandos do Linux.

**Por que:** sem isso o k3s não instala ou não inicia. Os cgroups em kernel antigo são o **principal risco técnico** do projeto e por isso são verificados já na primeira box.

**Links:**
- Configuração do APT para Debian Buster: [../apt_install/APT-ARMBIAN-RK322X.md](../apt_install/APT-ARMBIAN-RK322X.md)
- Requisitos do k3s: https://docs.k3s.io/installation/requirements

### Etapa 4: Rede por Wi-Fi

**O que fazer:**
1. Em cada box, rodar `sudo rk322x-config` e escolher SoC `RK3228A`, flash `emmc`, LED `led-conf1` e, no driver de Wi-Fi, a família **ssv6x5x**. Depois reiniciar.
2. Conferir se a interface `wlan0` existe com `ip link`.
3. Conectar na rede de 2,4 GHz com `nmcli` ou `nmtui`, deixando marcada a opção de **conectar automaticamente** a cada boot.
4. Anotar o **MAC do `wlan0`** de cada box e criar a **reserva de DHCP** no roteador para dar um IP fixo a cada uma.
5. Depois de confirmar que o Wi-Fi funciona, **retirar o cabo**, para o k3s não escolher a interface de cabo por engano.
6. Testar a qualidade do link: `ping -c 50 IP_DO_MASTER` (observe a perda de pacotes e a variação do tempo de resposta).

Exemplo de plano de endereços (ajuste para a sua rede):
- `k3s-master`: 192.168.1.10
- `tvbox01` a `tvbox12`: 192.168.1.101 a 192.168.1.112

**Por que Wi-Fi nesta variante:** menos cabos e menos portas de switch, e as boxes podem ficar em lugares diferentes.

**Links:**
- Wi-Fi no RK3228A com chip SSV (guia completo): [../wi-fi_settings/WI-FI_RK322x.md](../wi-fi_settings/WI-FI_RK322x.md)

### Etapa 5: Master e primeiro worker com k3s

**O que fazer:**
1. No master, instalar o servidor:
   ```bash
   curl -sfL https://get.k3s.io | sh -
   ```
2. Pegar o token do cluster em `/var/lib/rancher/k3s/server/node-token`.
3. Na primeira box (já no Wi-Fi), instalar o agente:
   ```bash
   curl -sfL https://get.k3s.io | K3S_URL=https://IP_DO_MASTER:6443 K3S_TOKEN=SEU_TOKEN sh -
   ```
4. No master, conferir se a box aparece como `Ready`:
   ```bash
   sudo k3s kubectl get nodes
   ```
5. Se a rede entre os nós falhar por a interface errada ter sido escolhida, o k3s tem uma opção para indicar a interface do Wi-Fi (`--flannel-iface wlan0`). Confira a opção na documentação antes de usar, pois isso ainda não foi testado neste projeto.

**O que será usado:** k3s (servidor no master, agente na box).

**Por que k3s:** é um Kubernetes enxuto. O agente pede cerca de 512 MB de RAM, o que cabe em boxes de 1 GB, e ele roda em armhf (32 bits).

**Atenção:** nunca coloque o token no repositório do GitHub.

**Links:**
- Requisitos do k3s: https://docs.k3s.io/installation/requirements
- Repositório oficial do k3s: https://github.com/k3s-io/k3s
- Tutorial de k3s em ARM (Raspberry Pi), com cgroups e swap: https://oneuptime.com/blog/post/2026-01-27-k3s-raspberry-pi/view
- Tutorial de k3s para edge e home lab: https://tutorials.technology/tutorials/k3s-kubernetes-tutorial-2026.html
- Palestra sobre k3s em hardware de mídia (5 nós): https://www.nclouds.com/nsights/introduction-to-k3s-microcloud/

### Etapa 6: Teste de carga e de estabilidade na primeira box

**O que fazer:**
1. Subir um **nginx** com 2 ou 3 réplicas e acessar pelo navegador.
2. Desligar a box que roda uma réplica e observar o k3s recriando-a em outro nó.
3. Observar o uso de memória com `free -h`.
4. **Deixar o cluster ligado por várias horas** e verificar se a box fica alternando entre `Ready` e `NotReady` (sinal de Wi-Fi instável).

**Por que:** é o teste que mostra se o cluster é viável antes de replicar o trabalho nas outras 11 boxes. **Se o k3s não rodar no kernel 4.4, ou se o Wi-Fi for instável demais, a alternativa é voltar para cabo** (veja `README.md` desta pasta) ou testar o Docker Swarm.

### Etapa 7: Replicar nas outras 11 boxes, com Ansible

**O que fazer:**
1. Instalar o Armbian nas outras boxes (Etapas 2, 3 e 4).
2. Configurar a chave SSH do computador de controle em todas.
3. Escrever o **inventário** do Ansible com os 12 IPs e os **playbooks** que preparam o sistema e instalam o agente do k3s.
4. Rodar os playbooks e conferir com `kubectl get nodes` até aparecerem 13 nós (1 master e 12 workers).

**O que será usado:** Ansible, com inventário e playbooks.

**Por que:** repetir à mão em 12 máquinas gera erros. Com o Ansible o processo fica **reproduzível**, e uma box que precise ser reinstalada volta ao mesmo estado. As credenciais do Wi-Fi não devem ir para o repositório: use o Ansible Vault ou um arquivo de exemplo (`.example`) sem a senha real.

**Links:**
- Repositório oficial do Ansible: https://github.com/ansible/ansible
- Playbooks Ansible para k3s com Armbian (referência): https://www.github.com/devoops0/k3s
- Scripts de instalação do k3s (referência): https://github.com/franciscojsc/scripts-install-k3s

### Etapa 8: Serviços e operação

**O que fazer:**
1. Instalar um monitoramento leve para acompanhar CPU, RAM, temperatura e a **qualidade do sinal Wi-Fi**.
2. Subir serviços pequenos, como um site, um bot ou uma automação.
3. Definir **limites de memória** (`resources.limits`) em cada serviço.
4. Fazer backup do master (configuração e token) e guardar os arquivos do Ansible no Git, **sem segredos**.

**Por que:** com 1 GB por nó, um serviço sem limite de memória pode derrubar a box inteira.

### Etapa 9: Testes e medições

**O que fazer:** medir o uso de RAM por nó, o desempenho de um serviço de teste com 1, 3, 6 e 12 workers, o tempo que o k3s leva para recriar um serviço quando uma box é desligada e a **perda de pacotes e a latência** da rede Wi-Fi com todas as boxes ativas.

**Por que:** são esses dados que mostram até onde esse hardware vai. Se der para repetir um dos testes com cabo em uma box, a comparação **Wi-Fi contra cabo** vira um resultado valioso para o trabalho.

---

## Por que usar essa abordagem

- **Armbian em vez do Android original:** o Android serve para assistir vídeo, não para ser servidor. O Armbian é um Linux comum, com terminal, SSH e suporte a containers.
- **k3s em vez do Kubernetes completo:** o Kubernetes tradicional é pesado e trabalhoso para instalar, e o agente do k3s cabe nos 512 MB mínimos que a documentação indica. Ele continua compatível com o Kubernetes, então o aprendizado vale para o mercado.
- **k3s em vez do Docker Swarm:** o Swarm é mais leve e simples, mas é menos usado hoje e tem menos recursos. Ele fica como **plano B**.
- **Ansible em vez de configurar à mão:** com 12 máquinas, a automação evita erros e deixa o processo repetível por outras pessoas.
- **Wi-Fi em vez de cabo:** menos cabos, menos portas de switch e mais liberdade para posicionar as boxes. Em troca, o cluster fica **menos estável e mais lento**, como explicado abaixo. Por isso este projeto sugere comparar com a versão por cabo.
- **Master por cabo e fora das boxes:** o servidor do k3s precisa de mais RAM do que as boxes têm e, sendo o centro do cluster, não deve depender de Wi-Fi.
- **TV boxes em vez de equipamento novo:** hardware barato, que gasta pouca energia e que, de outro modo, ficaria parado.

## Limitações e melhorias por causa do hardware

Todas as boxes têm **1 GB de RAM**, e isso define o que o projeto consegue entregar. O uso de Wi-Fi acrescenta limitações próprias.

**Limitações de hardware (1 GB de RAM)**

- **Pouca memória:** o agente do k3s já consome boa parte do 1 GB, e sobra pouco para os seus serviços. Só dá para rodar coisas leves. Programas que precisam de 1 GB ou mais não rodam em nenhuma box.
- **Soma não é igual a potência:** os 12 GB totais estão divididos em pedaços de 1 GB, então o cluster não equivale a um servidor com 12 GB.
- **Master único:** se o master cair, o cluster perde o controle (os containers que já estão rodando continuam, mas nada novo é agendado). Alta disponibilidade exigiria 3 ou mais servidores, o que não cabe em boxes de 1 GB.
- **Arquitetura de 32 bits (armhf):** nem toda imagem de container tem versão armhf, o que limita os programas disponíveis.
- **Kernel antigo (4.4):** pode ter limitações com cgroups e containers. Isso ainda precisa ser testado.
- **Debian Buster arquivado:** não recebe mais atualizações de segurança. Mantenha as boxes numa rede interna, sem exposição direta à Internet.
- **Memória interna e energia:** modelos com NAND são mais problemáticos, e fontes de má qualidade causam travamentos e corrupção de dados.

**Limitações específicas do Wi-Fi**

- **Só funciona em parte das boxes:** o guia de Wi-Fi vale para RK3228A com chip SSV 6051p/6256p. Outras boxes podem não ter Wi-Fi utilizável e precisariam de adaptador USB.
- **Sem alternativa de kernel:** o Wi-Fi só funcionou no kernel legacy 4.4.194. Se esse kernel der problema com cgroups ou containers, **não dá para trocar para o kernel 5.10** sem perder o Wi-Fi (a menos que se use um adaptador USB, o que não foi testado).
- **Apenas 2,4 GHz:** a faixa é mais congestionada e mais lenta que a de 5 GHz. Com 12 boxes dividindo o mesmo canal, a velocidade real por nó será baixa.
- **Estabilidade:** perda de pacotes e quedas de sinal podem fazer os nós alternarem entre `Ready` e `NotReady` e fazer o k3s recriar serviços sem necessidade.
- **Latência maior:** o tráfego entre os nós e com o master fica mais lento e menos previsível, o que afeta os resultados dos testes de desempenho.
- **Rede oculta não é suportada:** a rede de 2,4 GHz precisa estar visível.
- **Senha do Wi-Fi nas boxes:** a senha fica guardada em cada nó e **nunca deve ir para o GitHub**.

**Melhorias e estratégias para contornar**

- **Roteador dedicado** ao cluster, com canal fixo e pouca interferência de outras redes, e boxes próximas dele.
- **Reserva de DHCP** pelo MAC do `wlan0`, para o IP de cada box nunca mudar.
- **Limites de memória** (`resources.limits`) em cada serviço, para um programa não derrubar a box.
- **Serviços leves e imagens pequenas,** com versão armhf confirmada antes do uso.
- **Monitoramento** da perda de pacotes, da latência e do sinal Wi-Fi.
- **Comparação com cabo:** repetir os testes principais em uma ou mais boxes por cabo, para medir o custo real do Wi-Fi.
- **Adaptador USB Wi-Fi** nas boxes sem Wi-Fi funcional (o guia cita chipsets Mediatek MT7601U e Realtek RTL8188, que costumam funcionar sem driver extra). Isso ainda não foi testado neste projeto.
- **Backup do master e do Ansible,** guardando o repositório com os playbooks e o inventário.
- **Evolução futura:** trocar o master por uma máquina melhor ou adicionar mais servidores, se for preciso mais disponibilidade.

---

## Repositórios e links úteis

- k3s (repositório oficial): https://github.com/k3s-io/k3s
- Requisitos do k3s: https://docs.k3s.io/installation/requirements
- Ansible (repositório oficial): https://github.com/ansible/ansible
- Playbooks Ansible para k3s com Armbian (referência): https://www.github.com/devoops0/k3s
- Scripts de instalação do k3s (referência): https://github.com/franciscojsc/scripts-install-k3s
- Tutorial de k3s em ARM (Raspberry Pi): https://oneuptime.com/blog/post/2026-01-27-k3s-raspberry-pi/view
- Tutorial de k3s para edge e home lab: https://tutorials.technology/tutorials/k3s-kubernetes-tutorial-2026.html
- Palestra sobre k3s em hardware de mídia: https://www.nclouds.com/nsights/introduction-to-k3s-microcloud/
- Imagens do Armbian para RK322x: https://armbian.tnahosting.net/archive/rk322x-box/archive/
- Documentação oficial do Armbian: https://docs.armbian.com/getting-started/
- Fórum Armbian (RK322x): https://forum.armbian.com/topic/34923-csc-armbian-for-rk322x-tv-box-boards/
- balenaEtcher: https://etcher.balena.io/#download-etcher