# Cluster k3s com TV Boxes (RK322x)

Projeto de montagem de um cluster de baixo custo com **12 TV boxes reaproveitadas** (chip RK3228/RK3229, **1 GB de RAM cada**), usando **Armbian** como sistema, **k3s** (Kubernetes leve) para gerenciar os containers e **Ansible** para automatizar a configuração.

Esta pasta explica o que será feito, em que ordem, com quais ferramentas e por que essas escolhas foram feitas. O objetivo é que qualquer pessoa consiga repetir o projeto.

> **Status:** planejamento. A etapa mais importante a validar é se o k3s roda no kernel antigo (4.4) das boxes. Isso será testado na primeira box, antes de preparar as outras 11.

---

## Visão geral da arquitetura

- **1 master (servidor k3s):** uma máquina com **pelo menos 2 núcleos e 2 GB de RAM**, fora das TV boxes (PC antigo, notebook, Raspberry Pi 4 ou VM). Ela comanda o cluster.
- **12 workers (agentes k3s):** as TV boxes com Armbian, nomeadas de `tvbox01` a `tvbox12`.
- **Rede:** todos ligados por **cabo** a um switch, com IP fixo (ou reserva no DHCP do roteador).
- **Máquina de controle:** o computador de onde se roda o Ansible e o `kubectl` (pode ser o próprio master).

```
                 ┌──────────────┐
                 │  k3s master  │  (≥ 2 GB de RAM)
                 └──────┬───────┘
                        │ cabo de rede / switch
   ┌────────┬───────────┼───────────┬────────┐
┌──┴────┐ ┌─┴─────┐    ...     ┌────┴──┐ ┌───┴───┐
│tvbox01│ │tvbox02│            │tvbox11│ │tvbox12│  (workers, 1 GB cada)
└───────┘ └───────┘            └───────┘ └───────┘
```

## Materiais

- 12 TV boxes com chip RK3228a, RK3228b ou RK3229 e 1 GB de RAM.
- Cartões SD de 8 GB ou mais (um só pode ser reaproveitado, gravando uma box por vez).
- 1 máquina para o master, com 2 GB de RAM ou mais.
- Switch com pelo menos 13 portas e cabos de rede.
- 12 fontes de energia estáveis.
- Teclado e TV/monitor com HDMI para a configuração inicial.

---

## Passo a passo

### Etapa 1: Levantamento do hardware

**O que fazer:** identificar, em cada box, o chip (RK3228a, RK3228b ou RK3229), o tipo de RAM (DDR2 ou DDR3) e a memória interna (eMMC, eMCP ou NAND).

**O que será usado:** aplicativo **AIDA64** (no Android da box) e o **Multitool**, que avisa na tela inicial se a placa tem NAND.

**Por que:** o chip e a memória interna definem qual imagem usar e o risco da instalação. Placas com NAND são mais difíceis de suportar.

**Links:**
- Guia de instalação do Armbian (itens 1 a 4): [../README.md](../README.md)
- Fórum Armbian para TV boxes RK322x: https://forum.armbian.com/topic/34923-csc-armbian-for-rk322x-tv-box-boards/

### Etapa 2: Instalação do Armbian (começando por uma única box)

**O que fazer:** gravar o Multitool no cartão SD, fazer o **backup da firmware original** e gravar o Armbian na memória interna.

**O que será usado:**
- **Multitool** e **balenaEtcher**.
- Imagem **`Armbian_21.05.1_Rk322x-box_buster_legacy_4.4.194.img.xz`** (Debian Buster, kernel legacy 4.4.194).

**Por que:** o Android original não serve para rodar um servidor. O Armbian tem suporte da comunidade para RK322x. A imagem legacy foi a única em que o Wi-Fi funcionou nos testes.

**Links:**
- Passo a passo completo: [../README.md](../README.md)
- Imagens do Armbian para RK322x: https://armbian.tnahosting.net/archive/rk322x-box/archive/
- balenaEtcher: https://etcher.balena.io/#download-etcher
- Documentação oficial do Armbian: https://docs.armbian.com/getting-started/

### Etapa 3: Preparação do sistema em cada nó

**O que fazer:**
1. Corrigir o APT: o Debian Buster está arquivado, então os repositórios precisam apontar para `archive.debian.org`.
2. Acertar a data do sistema.
3. Desativar o swap (inclusive o zram do Armbian, se estiver ativo; confira com `swapon --show`).
4. Verificar os cgroups, em especial o de memória: `cat /proc/cgroups`.
5. Definir hostname único, IP fixo e SSH por chave.

**O que será usado:** `apt`, `armbian-config`, `nmtui` e comandos do Linux.

**Por que:** sem isso o k3s não instala ou não inicia. Os cgroups em kernel antigo são o **principal risco técnico** do projeto e por isso são verificados já na primeira box.

**Links:**
- Configuração do APT para Debian Buster: [../apt_install/APT-ARMBIAN-RK322X.md](../apt_install/APT-ARMBIAN-RK322X.md)
- Requisitos do k3s: https://docs.k3s.io/installation/requirements

### Etapa 4: Rede

**O que fazer:** ligar todas as boxes e o master ao switch por cabo, com IP fixo para cada uma. Exemplo de plano de endereços (ajuste para a sua rede):
- `k3s-master`: 192.168.1.10
- `tvbox01` a `tvbox12`: 192.168.1.101 a 192.168.1.112

**Por que cabo:** Wi-Fi é instável para um cluster. Além disso, o chip Wi-Fi dessas boxes só funciona na faixa de 2,4 GHz e com um driver específico.

**Links:**
- Wi-Fi no RK3228A (apenas como alternativa e para testes): [../wi-fi_settings/WI-FI_RK322x.md](../wi-fi_settings/WI-FI_RK322x.md)

### Etapa 5: Master e primeiro worker com k3s

**O que fazer:**
1. No master, instalar o servidor:
   ```bash
   curl -sfL https://get.k3s.io | sh -
   ```
2. Pegar o token do cluster em `/var/lib/rancher/k3s/server/node-token`.
3. Na primeira box, instalar o agente:
   ```bash
   curl -sfL https://get.k3s.io | K3S_URL=https://IP_DO_MASTER:6443 K3S_TOKEN=SEU_TOKEN sh -
   ```
4. No master, conferir se a box aparece como `Ready`:
   ```bash
   sudo k3s kubectl get nodes
   ```

**O que será usado:** k3s (servidor no master, agente na box).

**Por que k3s:** é um Kubernetes enxuto. O agente pede cerca de 512 MB de RAM, o que cabe em boxes de 1 GB, e ele roda em armhf (32 bits).

**Atenção:** nunca coloque o token no repositório do GitHub.

**Links:**
- Requisitos do k3s: https://docs.k3s.io/installation/requirements
- Repositório oficial do k3s: https://github.com/k3s-io/k3s
- Tutorial de k3s em ARM (Raspberry Pi), com cgroups e swap: https://oneuptime.com/blog/post/2026-01-27-k3s-raspberry-pi/view
- Tutorial de k3s para edge e home lab: https://tutorials.technology/tutorials/k3s-kubernetes-tutorial-2026.html
- Palestra sobre k3s em hardware de mídia (5 nós): https://www.nclouds.com/nsights/introduction-to-k3s-microcloud/

### Etapa 6: Teste de carga na primeira box

**O que fazer:**
1. Subir um **nginx** com 2 ou 3 réplicas.
2. Acessar pelo navegador.
3. Desligar a box que roda uma réplica e observar o k3s recriando-a em outro nó.
4. Observar o uso de memória com `free -h`.

**Por que:** é o teste que mostra se o cluster é viável antes de replicar o trabalho nas outras 11 boxes. **Se o k3s não rodar no kernel 4.4, o plano é testar outra imagem (veja a seção de melhorias abaixo) ou o Docker Swarm.**

### Etapa 7: Replicar nas outras 11 boxes, com Ansible

**O que fazer:**
1. Instalar o Armbian nas outras boxes (Etapas 2 e 3).
2. Configurar a chave SSH do computador de controle em todas.
3. Escrever o **inventário** do Ansible com os 12 IPs e os **playbooks** que preparam o sistema e instalam o agente do k3s.
4. Rodar os playbooks e conferir com `kubectl get nodes` até aparecerem 13 nós (1 master e 12 workers).

**O que será usado:** Ansible, com inventário e playbooks.

**Por que:** repetir à mão em 12 máquinas gera erros. Com o Ansible o processo fica **reproduzível**, e uma box que precise ser reinstalada volta ao mesmo estado.

**Links:**
- Repositório oficial do Ansible: https://github.com/ansible/ansible
- Playbooks Ansible para k3s com Armbian (referência): https://www.github.com/devoops0/k3s
- Scripts de instalação do k3s (referência): https://github.com/franciscojsc/scripts-install-k3s

### Etapa 8: Serviços e operação

**O que fazer:**
1. Instalar um monitoramento leve para acompanhar CPU, RAM e temperatura.
2. Subir serviços pequenos, como um site, um bot ou uma automação.
3. Definir **limites de memória** (`resources.limits`) em cada serviço.
4. Fazer backup do master (configuração e token) e guardar os arquivos do Ansible no Git, **sem segredos**.

**Por que:** com 1 GB por nó, um serviço sem limite de memória pode derrubar a box inteira.

### Etapa 9: Testes e medições

**O que fazer:** medir o uso de RAM por nó, o desempenho de um serviço de teste com 1, 3, 6 e 12 workers e o tempo que o k3s leva para recriar um serviço quando uma box é desligada.

**Por que:** são esses dados que mostram até onde esse hardware vai.

---

## Por que usar essa abordagem

- **Armbian em vez do Android original:** o Android serve para assistir vídeo, não para ser servidor. O Armbian é um Linux comum, com terminal, SSH e suporte a containers.
- **k3s em vez do Kubernetes completo:** o Kubernetes tradicional é pesado e trabalhoso para instalar, e o agente do k3s cabe nos 512 MB mínimos que a documentação indica. Ele continua compatível com o Kubernetes, então o aprendizado vale para o mercado.
- **k3s em vez do Docker Swarm:** o Swarm é mais leve e simples, mas é menos usado hoje e tem menos recursos. Ele fica como **plano B**.
- **Ansible em vez de configurar à mão:** com 12 máquinas, a automação evita erros e deixa o processo repetível por outras pessoas.
- **Cabo em vez de Wi-Fi:** mais estável para um cluster e sem depender do driver do chip Wi-Fi.
- **Master fora das boxes:** o servidor do k3s precisa de mais RAM do que as boxes têm.
- **TV boxes em vez de equipamento novo:** hardware barato, que gasta pouca energia e que, de outro modo, ficaria parado.

## Limitações e melhorias por causa do hardware

Todas as boxes têm **1 GB de RAM**, e isso define o que o projeto consegue entregar.

**Limitações**

- **Pouca memória:** o agente do k3s já consome boa parte do 1 GB, e sobra pouco para os seus serviços. Só dá para rodar coisas leves. Programas que precisam de 1 GB ou mais não rodam em nenhuma box.
- **Soma não é igual a potência:** os 12 GB totais estão divididos em pedaços de 1 GB, então o cluster não equivale a um servidor com 12 GB.
- **Master único:** se o master cair, o cluster perde o controle (os containers que já estão rodando continuam, mas nada novo é agendado). Alta disponibilidade exigiria 3 ou mais servidores, o que não cabe em boxes de 1 GB.
- **Arquitetura de 32 bits (armhf):** nem toda imagem de container tem versão armhf, o que limita os programas disponíveis.
- **Kernel antigo (4.4):** pode ter limitações com cgroups e containers. Isso ainda precisa ser testado.
- **Debian Buster arquivado:** não recebe mais atualizações de segurança. Mantenha as boxes numa rede interna, sem exposição direta à Internet.
- **Memória interna e energia:** modelos com NAND são mais problemáticos, e fontes de má qualidade causam travamentos e corrupção de dados.
- **Rede:** a velocidade da porta Ethernet dessas boxes pode ser limitada (muitas têm 100 Mbps). Confirme no seu modelo.

**Melhorias e estratégias para contornar**

- **Limites de memória** (`resources.limits`) em cada serviço, para um programa não derrubar a box.
- **Serviços leves e imagens pequenas,** com versão armhf confirmada antes do uso.
- **Plano B para o kernel:** como o cluster usa cabo e não depende do Wi-Fi, vale testar a imagem **current (kernel 5.10)** se o kernel 4.4 der problema com cgroups ou containers. Essa troca não foi testada.
- **Backup do master e do Ansible,** guardando o repositório com os playbooks e o inventário.
- **Monitoramento** para identificar boxes sobrecarregadas ou superaquecendo.
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