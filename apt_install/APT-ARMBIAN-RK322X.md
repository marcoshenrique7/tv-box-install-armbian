# Configuração do APT — Armbian RK322x

Guia para corrigir e configurar o APT em TV Boxes RK322x utilizando Armbian baseado no Debian Buster.

## 1. Verificar o sistema

```bash
cat /etc/os-release
uname -m
```

---

## 2. Verificar os repositórios

```bash
cat /etc/apt/sources.list
```

Verificar também repositórios adicionais:

```bash
ls /etc/apt/sources.list.d/
cat /etc/apt/sources.list.d/*.list
```

---

## 3. Fazer backup

Antes de alterar os repositórios:

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.backup
```

Se existirem repositórios adicionais:

```bash
sudo cp -r /etc/apt/sources.list.d /etc/apt/sources.list.d.backup
```

---

## 4. Configurar os repositórios do Debian Buster

O Debian Buster é uma versão antiga e seus repositórios foram movidos para o arquivo histórico do Debian.

Editar:

```bash
sudo nano /etc/apt/sources.list
```

Utilizar:

```text
deb http://archive.debian.org/debian buster main contrib non-free
deb http://archive.debian.org/debian buster-updates main contrib non-free
deb http://archive.debian.org/debian buster-backports main contrib non-free
```

Salvar no Nano:

```text
CTRL + O
ENTER
CTRL + X
```

---

## 5. Verificar repositórios adicionais

Verificar:

```bash
ls /etc/apt/sources.list.d/
```

E:

```bash
cat /etc/apt/sources.list.d/*.list
```

Caso algum repositório antigo esteja causando erro, editar o arquivo:

```bash
sudo nano /etc/apt/sources.list.d/arquivo.list
```

Para desativar uma linha, adicionar `#` no início:

```text
# deb http://repositorio-antigo ...
```

---

## 6. Limpar as listas antigas

```bash
sudo rm -rf /var/lib/apt/lists/*
sudo apt clean
```

---

## 7. Atualizar os repositórios

Como o Buster está arquivado, pode ser necessário ignorar a validade temporal dos metadados:

```bash
sudo apt -o Acquire::Check-Valid-Until=false update
```

Os repositórios devem aparecer utilizando:

```text
archive.debian.org
```

---

# Chaves de segurança (GPG)

Durante o `apt update`, podem aparecer erros relacionados às chaves de assinatura:

```text
NO_PUBKEY XXXXXXXX
```

```text
The following signatures couldn't be verified
```

ou:

```text
EXPKEYSIG XXXXXXXX
```

Esses erros estão relacionados às chaves utilizadas para verificar a autenticidade dos repositórios.

## Verificar as chaves instaladas

```bash
apt-key list
```

Também:

```bash
ls -l /etc/apt/trusted.gpg.d/
```

## Verificar o Debian Archive Keyring

O pacote responsável pelas chaves dos arquivos históricos do Debian é:

```text
debian-archive-keyring
```

Verificar se está instalado:

```bash
dpkg -l | grep debian-archive-keyring
```

Se o APT estiver funcionando:

```bash
sudo apt install --reinstall debian-archive-keyring
```

Depois:

```bash
sudo apt -o Acquire::Check-Valid-Until=false update
```

> **Importante:** não utilizar `--allow-unauthenticated` ou `trusted=yes` como solução padrão. Essas opções reduzem a segurança do sistema.

---

# Valid-Until x GPG

São problemas diferentes.

O comando:

```bash
sudo apt -o Acquire::Check-Valid-Until=false update
```

ignora somente a validade temporal dos metadados do repositório.

Ele **não desativa a verificação das assinaturas GPG**.

A verificação das assinaturas deve continuar habilitada.

---

# Atualizar os pacotes

Depois que o `apt update` funcionar:

```bash
sudo apt upgrade -y
```

Ou:

```bash
sudo apt full-upgrade -y
```

---

# Corrigir pacotes quebrados

Caso existam problemas de dependências:

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

---

# Testar o APT

Instalar um pacote simples para confirmar que está funcionando:

```bash
sudo apt install curl
```

Ou:

```bash
sudo apt install wget
```

Verificar pacotes que ainda podem ser atualizados:

```bash
apt list --upgradable
```

---

# Principais erros

## 404 Not Found

Exemplo:

```text
http://deb.debian.org/debian buster
404 Not Found
```

### Solução

Utilizar:

```text
http://archive.debian.org/debian
```

---

## does not have a Release file

Verificar:

```bash
/etc/apt/sources.list
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

Verificar:

```bash
dpkg -l | grep debian-archive-keyring
```

E tentar:

```bash
sudo apt install --reinstall debian-archive-keyring
```

---

## EXPKEYSIG

Tentar atualizar o keyring:

```bash
sudo apt install --reinstall debian-archive-keyring
```

Depois:

```bash
sudo apt -o Acquire::Check-Valid-Until=false update
```

---

# Sequência rápida

Quando a configuração dos repositórios já estiver correta:

```bash
sudo rm -rf /var/lib/apt/lists/*
sudo apt clean
sudo apt -o Acquire::Check-Valid-Until=false update
sudo apt upgrade -y
```

Caso existam pacotes quebrados:

```bash
sudo apt --fix-broken install
sudo dpkg --configure -a
```

Teste final:

```bash
sudo apt install curl
```

---

# Observação

O uso do `archive.debian.org` é necessário porque o Debian Buster é uma versão antiga e arquivada.

Esse procedimento permite continuar utilizando os pacotes disponíveis no arquivo histórico, mas não significa que o Debian Buster voltou a ser uma versão suportada.

Sempre que houver suporte adequado ao hardware, recomenda-se utilizar uma versão mais recente do Armbian/Debian.

Este guia foi criado para facilitar a preparação de **TV Boxes RK322x com Armbian antigo**, permitindo corrigir o APT antes de instalar e configurar os demais componentes do sistema.
