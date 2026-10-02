# Configuração do APT — Armbian RK322x

Procedimento utilizado para corrigir e atualizar o APT em uma TV Box RK322x com Armbian baseado no Debian Buster.

## 1. Verificar o sistema

```bash
cat /etc/os-release
uname -m
uname -r
```

## 2. Verificar a conexão de rede

Verificar a rota:

```bash
ip route
```

Testar o roteador:

```bash
ping -c 4 192.168.1.1
```

Testar conexão externa:

```bash
ping -c 4 8.8.8.8
```

Também testar o acesso HTTPS:

```bash
curl -4 https://example.com
```

> O `ping` externo pode apresentar perda de pacotes mesmo com a Internet funcionando. Por isso, o teste com `curl` também deve ser utilizado.

## 3. Corrigir a data do sistema

Verificar a data:

```bash
date
```

A data incorreta causou problemas de certificado HTTPS.

Corrigir:

```bash
sudo date -s "2026-09-29 10:37:00"
```

> Substitua pela data e hora atuais.

Confirmar:

```bash
date
```

## 4. Fazer backup do APT

Antes de alterar os repositórios:

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.backup
```

## 5. Corrigir os repositórios

Editar:

```bash
sudo nano /etc/apt/sources.list
```

Deixar:

```text
deb http://archive.debian.org/debian buster main contrib non-free
deb http://archive.debian.org/debian buster-updates main contrib non-free
```

### Buster Backports

Inicialmente foi utilizado:

```text
deb http://archive.debian.org/debian buster-backports main contrib non-free
```

Porém, esse repositório apresentou erros de chave:

```text
NO_PUBKEY 0E984040386FA1D9
NO_PUBKEY 6ED0E7882643E131
```

Como o `buster-backports` não era necessário para a preparação da TV Box, ele foi removido.

> Não utilizar `--allow-unauthenticated` ou `trusted=yes` como solução para esse problema.

Salvar no Nano:

```text
CTRL + O
ENTER
CTRL + X
```

## 6. Limpar as listas antigas

```bash
sudo rm -rf /var/lib/apt/lists/*
sudo apt clean
```

## 7. Atualizar os índices do APT

Como o Debian Buster está arquivado, foi necessário ignorar a validade temporal dos metadados:

```bash
sudo apt -o Acquire::Check-Valid-Until=false update
```

O resultado obtido foi:

```text
Hit:1 http://archive.debian.org/debian buster InRelease
Hit:2 http://archive.debian.org/debian buster-updates InRelease

Reading package lists... Done
84 packages can be upgraded.
```

### Importante

O parâmetro:

```text
Acquire::Check-Valid-Until=false
```

ignora somente a validade temporal dos metadados do repositório.

Ele **não desativa a verificação das assinaturas GPG**.

## 8. Verificar os pacotes disponíveis

```bash
apt list --upgradable
```

## 9. Atualizar os pacotes

```bash
sudo apt upgrade
```

Para aceitar automaticamente:

```bash
sudo apt upgrade -y
```

## 10. Caso ocorram problemas de dependências

Executar:

```bash
sudo apt --fix-broken install
sudo dpkg --configure -a
```

Depois atualizar novamente:

```bash
sudo apt -o Acquire::Check-Valid-Until=false update
sudo apt upgrade
```

## Configuração final

A configuração que funcionou na TV Box foi:

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

### Observação

O `archive.debian.org` é utilizado porque o Debian Buster é uma versão antiga e arquivada. Os pacotes disponíveis nesse repositório não representam necessariamente as versões mais recentes.

Este procedimento foi utilizado especificamente para preparar uma TV Box RK322x com uma versão antiga do Armbian/Debian Buster.
