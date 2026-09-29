# Configuração do APT — Armbian RK322x

Guia para corrigir e configurar o APT em TV Boxes RK322x utilizando Armbian baseado no Debian Buster.

Este procedimento foi elaborado a partir da configuração de uma TV Box RK322x com uma versão antiga do Armbian/Debian Buster.

---

## 1. Verificar o sistema

Primeiro, verificar a versão do sistema e a arquitetura:

```bash
cat /etc/os-release
uname -m
```

Também é possível verificar a versão do kernel:

```bash
uname -r
```

---

## 2. Verificar a conexão de rede

Antes de corrigir o APT, é importante confirmar que a TV Box possui conexão de rede.

Verificar a rota padrão:

```bash
ip route
```

Testar o roteador:

```bash
ping -c 4 192.168.1.1
```

Testar conectividade externa por IP:

```bash
ping -c 4 1.1.1.1
```

> Observação: alguns servidores podem bloquear ou não responder a `ping`. Portanto, perda de pacotes no `ping` não significa necessariamente que a Internet esteja indisponível.

Um teste mais confiável é utilizar HTTP/HTTPS:

```bash
curl -4 https://example.com
```

Também pode ser utilizado:

```bash
curl -4 https://google.com
```

Se esses comandos retornarem conteúdo HTML ou uma resposta HTTP, a conexão está funcionando.

---

## 3. Verificar DNS

Verificar o arquivo:

```bash
cat /etc/resolv.conf
```

Testar a resolução de nomes:

```bash
getent hosts google.com
```

E:

```bash
getent ahostsv4 archive.debian.org
```

O segundo comando é especialmente útil para confirmar que o endereço do arquivo histórico do Debian pode ser resolvido por IPv4.

---

## 4. Verificar a data e hora do sistema

TV Boxes com imagens antigas podem iniciar com uma data incorreta.

Verificar:

```bash
date
```

Ou:

```bash
timedatectl
```

Uma data muito antiga pode causar problemas de HTTPS e certificados, por exemplo:

```text
SSL certificate problem: certificate is not yet valid
```

Se a data estiver incorreta, corrigir temporariamente:

```bash
sudo date -s "2026-09-29 10:37:00"
```

Substitua pela data e hora atuais.

Depois confirme:

```bash
date
```

> A data correta é importante principalmente para conexões HTTPS e validação de certificados.

---

## 5. Verificar os repositórios

Verificar o arquivo principal:

```bash
cat /etc/apt/sources.list
```

Verificar também repositórios adicionais:

```bash
ls /etc/apt/sources.list.d/
```

E, caso existam arquivos `.list`:

```bash
cat /etc/apt/sources.list.d/*.list
```

---

## 6. Fazer backup

Antes de alterar os repositórios:

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.backup
```

Se existirem repositórios adicionais:

```bash
sudo cp -r /etc/apt/sources.list.d /etc/apt/sources.list.d.backup
```

---

# 7. Configurar os repositórios do Debian Buster

O Debian Buster é uma versão antiga e seus repositórios foram movidos para o arquivo histórico do Debian.

Editar:

```bash
sudo nano /etc/apt/sources.list
```

Para a configuração utilizada neste procedimento, deixar:

```text
deb http://archive.debian.org/debian buster main contrib non-free
deb http://archive.debian.org/debian buster-updates main contrib non-free
```

### Não adicionar o Buster Backports neste procedimento

Inicialmente foi utilizado:

```text
deb http://archive.debian.org/debian buster-backports main contrib non-free
```

Porém, o repositório `buster-backports` apresentou erros de assinatura:

```text
NO_PUBKEY 0E984040386FA1D9
NO_PUBKEY 6ED0E7882643E131
```

A TV Box possui:

```text
debian-archive-keyring 2019.1
```

e o próprio repositório Buster disponibiliza:

```text
2019.1+deb10u1
```

Portanto, atualizar o keyring através dos repositórios Buster não resolve esse problema específico do `buster-backports`.

Como o objetivo deste procedimento é preparar a TV Box e atualizar os pacotes básicos, o `buster-backports` foi removido.

Salvar no Nano:

```text
CTRL + O
ENTER
CTRL + X
```

---

# 8. Verificar repositórios adicionais

Verificar:

```bash
ls /etc/apt/sources.list.d/
```

Depois:

```bash
cat /etc/apt/sources.list.d/*.list
```

Caso algum repositório antigo esteja causando erros, editar o arquivo correspondente:

```bash
sudo nano /etc/apt/sources.list.d/arquivo.list
```

Para desativar uma linha, adicionar `#` no início:

```text
# deb http://repositorio-antigo ...
```

É importante evitar repositórios antigos ou incompatíveis com Debian Buster.

---

# 9. Limpar as listas antigas

Depois de corrigir os repositórios:

```bash
sudo rm -rf /var/lib/apt/lists/*
```

Limpar o cache:

```bash
sudo apt clean
```

---

# 10. Atualizar os índices do APT

Como o Buster está arquivado, os metadados podem possuir datas antigas.

Utilizar:

```bash
sudo apt -o Acquire::Check-Valid-Until=false update
```

### Importante

O comando correto é:

```bash
Acquire::Check-Valid-Until=false
```

Não colocar espaço entre `::` e `Check-Valid-Until`.

Correto:

```bash
sudo apt -o Acquire::Check-Valid-Until=false update
```

Incorreto:

```bash
sudo apt -o Acquire:: Check-Valid-Until=false update
```

Se estiver funcionando corretamente, deverá aparecer algo semelhante a:

```text
Hit:1 http://archive.debian.org/debian buster InRelease
Hit:2 http://archive.debian.org/debian buster-updates InRelease

Reading package lists... Done
```

E, por exemplo:

```text
84 packages can be upgraded.
```

---

# 11. Chaves de segurança GPG

O APT utiliza chaves GPG para verificar a autenticidade dos repositórios.

Podem aparecer erros como:

```text
NO_PUBKEY XXXXXXXX
```

ou:

```text
The following signatures couldn't be verified
```

ou:

```text
EXPKEYSIG XXXXXXXX
```

Verificar o keyring instalado:

```bash
dpkg -l | grep debian-archive-keyring
```

Também pode ser utilizado:

```bash
apt-cache policy debian-archive-keyring
```

E:

```bash
apt-cache madison debian-archive-keyring
```

Na configuração utilizada neste procedimento, foi identificado:

```text
debian-archive-keyring 2019.1
```

e:

```text
debian-archive-keyring | 2019.1+deb10u1 | http://archive.debian.org/debian buster/main armhf Packages
```

Isso significa que o Buster fornece uma versão antiga do keyring.

### Importante

Não utilizar como solução padrão:

```bash
--allow-unauthenticated
```

nem:

```text
trusted=yes
```

Essas opções reduzem a segurança da verificação dos pacotes.

Também não é recomendado importar chaves aleatórias de servidores de chaves sem verificar sua procedência.

---

# 12. Valid-Until x GPG

São problemas diferentes.

O comando:

```bash
sudo apt -o Acquire::Check-Valid-Until=false update
```

ignora somente a validade temporal dos metadados do repositório.

Ele **não desativa a verificação das assinaturas GPG**.

A verificação criptográfica das assinaturas continua sendo realizada pelo APT.

---

# 13. Atualizar os pacotes

Depois que o `apt update` funcionar:

Verificar os pacotes disponíveis:

```bash
apt list --upgradable
```

Depois atualizar:

```bash
sudo apt upgrade
```

Para aceitar automaticamente:

```bash
sudo apt upgrade -y
```

O `apt upgrade` é suficiente para a atualização normal dos pacotes instalados.

O `full-upgrade` pode realizar alterações adicionais nas dependências:

```bash
sudo apt full-upgrade -y
```

Por isso, para uma preparação inicial da TV Box, é recomendado começar com:

```bash
sudo apt upgrade
```

---

# 14. Corrigir pacotes quebrados

Caso apareçam problemas de dependências:

```bash
sudo apt --fix-broken install
```

Depois:

```bash
sudo dpkg --configure -a
```

E novamente:

```bash
sudo apt -o Acquire::Check-Valid-Until=false update
```

Depois:

```bash
sudo apt upgrade
```

---

# 15. Testar o APT

Depois da atualização, instalar um pacote simples para confirmar que o sistema consegue baixar e instalar pacotes:

```bash
sudo apt install curl
```

Ou:

```bash
sudo apt install wget
```

Verificar novamente os pacotes pendentes:

```bash
apt list --upgradable
```

Se não houver pacotes pendentes, o sistema estará atualizado dentro dos repositórios disponíveis.

---

# 16. Principais erros

## 404 Not Found

Exemplo:

```text
http://deb.debian.org/debian buster
404 Not Found
```

### Solução

O Buster não está mais nos repositórios normais.

Utilizar:

```text
http://archive.debian.org/debian
```

---

## does not have a Release file

Verificar:

```bash
/etc/apt/sources.list
```

e:

```bash
/etc/apt/sources.list.d/
```

Desativar repositórios antigos ou indisponíveis.

---

## Release file expired

Executar:

```bash
sudo apt -o Acquire::Check-Valid-Until=false update
```

---

## NO_PUBKEY

Verificar o keyring:

```bash
dpkg -l | grep debian-archive-keyring
```

Verificar a versão disponível:

```bash
apt-cache policy debian-archive-keyring
```

Se o erro estiver relacionado ao `buster-backports`, a solução utilizada neste procedimento foi **remover o repositório `buster-backports`**, mantendo apenas:

```text
deb http://archive.debian.org/debian buster main contrib non-free
deb http://archive.debian.org/debian buster-updates main contrib non-free
```

---

## EXPKEYSIG

Verificar o keyring:

```bash
dpkg -l | grep debian-archive-keyring
```

Se houver uma versão disponível nos repositórios configurados:

```bash
sudo apt install --reinstall debian-archive-keyring
```

Depois:

```bash
sudo apt -o Acquire::Check-Valid-Until=false update
```

Se o problema estiver relacionado a um repositório arquivado específico, como `buster-backports`, verificar primeiro se esse repositório é realmente necessário.

---

## SSL certificate problem

Exemplo:

```text
SSL certificate problem: certificate is not yet valid
```

Verificar:

```bash
date
```

Se a data estiver incorreta, corrigir:

```bash
sudo date -s "AAAA-MM-DD HH:MM:SS"
```

Depois testar novamente:

```bash
curl -4 https://example.com
```

---

# 17. Sequência rápida

Depois de verificar a rede e corrigir os repositórios:

```bash
sudo rm -rf /var/lib/apt/lists/*
sudo apt clean
sudo apt -o Acquire::Check-Valid-Until=false update
apt list --upgradable
sudo apt upgrade
```

Se houver problemas de dependências:

```bash
sudo apt --fix-broken install
sudo dpkg --configure -a
```

Depois:

```bash
sudo apt -o Acquire::Check-Valid-Until=false update
sudo apt upgrade
```

Teste final:

```bash
sudo apt install curl
```

---

# 18. Configuração final utilizada na TV Box

A configuração que funcionou para a TV Box RK322x foi:

```text
deb http://archive.debian.org/debian buster main contrib non-free
deb http://archive.debian.org/debian buster-updates main contrib non-free
```

Atualização:

```bash
sudo apt -o Acquire::Check-Valid-Until=false update
```

Resultado obtido:

```text
Hit:1 http://archive.debian.org/debian buster InRelease
Hit:2 http://archive.debian.org/debian buster-updates InRelease

Reading package lists... Done
84 packages can be upgraded.
```

A partir desse ponto, os pacotes podem ser atualizados com:

```bash
sudo apt upgrade
```

---

# 19. Observação

O uso de:

```text
archive.debian.org
```

é necessário porque o Debian Buster é uma versão antiga e arquivada.

Esse procedimento permite continuar utilizando os pacotes disponíveis no arquivo histórico, mas **não significa que o Debian Buster voltou a ser uma versão suportada**.

Além disso, os pacotes disponíveis no arquivo histórico não representam as versões mais recentes dos softwares.

Sempre que houver suporte adequado ao hardware, recomenda-se utilizar uma versão mais recente do Armbian/Debian.

Este guia foi criado especificamente para facilitar a preparação de **TV Boxes RK322x com Armbian antigo**, permitindo corrigir a rede, data do sistema e APT antes de instalar e configurar os demais componentes.

**Configuração recomendada para este procedimento:**

```text
Debian Buster
        ↓
archive.debian.org
        ↓
buster + buster-updates
        ↓
apt update com Check-Valid-Until=false
        ↓
apt upgrade
        ↓
TV Box pronta para continuar a configuração
```
