# ☁️ CirrusCloud

**Uma plataforma de nuvem privada construída do zero sobre OpenStack, com painel visual próprio, orquestração via Heat e Terraform, e infraestrutura totalmente documentada.**

![Status](https://img.shields.io/badge/status-em%20desenvolvimento-blue)
![OpenStack](https://img.shields.io/badge/OpenStack-DevStack-red)
![Licença](https://img.shields.io/badge/uso-aprendizado%2Fpessoal-lightgrey)

---

## 📖 Sobre o projeto

O **CirrusCloud** nasceu como um projeto de aprendizado prático de infraestrutura de nuvem: em vez de só ler sobre como o OpenStack funciona, a proposta foi **instalar, quebrar, consertar e customizar** um ambiente real, do zero, até virar uma plataforma com identidade própria — nome, visual, dashboard customizado e orquestração de infraestrutura como código.

Hoje o projeto é uma instalação completa do OpenStack (via DevStack) rodando dentro de uma VM local, com:

- 🎨 **Painel visual próprio** (tema customizado sobre o Horizon, o painel padrão do OpenStack)
- 📊 **Dashboard interno redesenhado**, com dados reais de uso/cota
- 🏗️ **Duas ferramentas de orquestração**: Heat (nativa do OpenStack) e Terraform (multi-nuvem)
- 🔒 **Camada de segurança**: firewall, HTTPS com TLS hardening
- 🌐 **Acesso público** via túnel Cloudflare
- 🐛 **Uma lista extensa de bugs reais encontrados e corrigidos**, documentados para quem for reproduzir o processo

Este documento serve tanto como **guia de instalação/reconstrução** quanto como **material de estudo** — cada seção técnica vem acompanhada de uma explicação do "porquê", não só do "como".

---

## 📑 Sumário

- [O que é OpenStack e DevStack](#-o-que-é-openstack-e-devstack)
- [Arquitetura do ambiente](#-arquitetura-do-ambiente)
- [Pré-requisitos](#-pré-requisitos)
- [Instalação do zero](#-instalação-do-zero)
- [Acesso e credenciais](#-acesso-e-credenciais)
- [Identidade visual (tema CirrusCloud)](#-identidade-visual-tema-cirruscloud)
- [Dashboard customizado](#-dashboard-customizado)
- [Segurança](#-segurança)
- [Problemas conhecidos e correções](#-problemas-conhecidos-e-correções)
- [Orquestração com Heat](#-orquestração-com-heat)
- [Orquestração com Terraform](#-orquestração-com-terraform)
- [Acesso público (Cloudflare Tunnel)](#-acesso-público-cloudflare-tunnel)
- [Roadmap / pendências](#-roadmap--pendências)
- [Créditos e aprendizados](#-créditos-e-aprendizados)

---

## 🧠 O que é OpenStack e DevStack

**OpenStack** é uma plataforma de código aberto para construir e gerenciar nuvens privadas e públicas — é, essencialmente, o "motor" por trás de serviços parecidos com AWS, Azure ou Google Cloud, só que você mesmo hospeda e opera. Ele não é um programa único, e sim um **conjunto de serviços independentes** que conversam entre si via API, cada um responsável por uma parte da infraestrutura:

| Serviço | Apelido | Responsabilidade |
|---|---|---|
| **Keystone** | "O porteiro" | Autenticação e autorização (usuários, projetos, tokens) |
| **Nova** | "O motor" | Cria e gerencia máquinas virtuais |
| **Neutron** | "A rede" | Redes virtuais, roteadores, IPs |
| **Glance** | "O catálogo" | Armazena e distribui imagens de sistema operacional |
| **Cinder** | "O armazém de blocos" | Discos/volumes anexáveis às VMs |
| **Swift** | "O armazém de objetos" | Armazenamento tipo S3 (arquivos, backups) |
| **Heat** | "O maestro" | Orquestração — cria infraestrutura inteira a partir de um arquivo de template |
| **Horizon** | "A vitrine" | Painel web para operar tudo visualmente |

Instalar o OpenStack "de verdade" (produção) é um processo complexo, geralmente feito com ferramentas como Kolla-Ansible, envolvendo múltiplos servidores físicos. Para aprendizado e testes, existe o **DevStack**: um conjunto de scripts que instala e configura todos esses serviços numa única máquina, de forma automatizada. É o que usamos aqui.

> ⚠️ **Importante:** o DevStack é uma instalação de *desenvolvimento*. Ele não tem a robustez, redundância e segurança necessárias para produção real com dados sensíveis — é ótimo para aprender e prototipar, não para hospedar um serviço crítico.

---

## 🏗️ Arquitetura do ambiente

![Arquitetura CirrusCloud](./architecture.svg)

<details>
<summary>Versão em texto (ASCII)</summary>

```
┌─────────────────────────────────────────────────────────┐
│  Notebook físico (host)                                  │
│                                                            │
│   ┌──────────────────────────────────────────────────┐   │
│   │  Multipass (gerenciador de VMs)                    │   │
│   │                                                     │   │
│   │   ┌────────────────────────────────────────────┐  │   │
│   │   │  VM: devstack-vm (Ubuntu 24.04, 12GB RAM)    │  │   │
│   │   │                                                │  │   │
│   │   │   ┌──────────────┐   ┌──────────────────┐    │  │   │
│   │   │   │  DevStack    │   │  Apache + Horizon │    │  │   │
│   │   │   │  (OpenStack) │◄──┤  (tema CirrusCloud)│    │  │   │
│   │   │   └──────────────┘   └──────────────────┘    │  │   │
│   │   │           │                                    │  │   │
│   │   │   ┌───────┴────────┐                          │  │   │
│   │   │   │ VMs internas    │  (criadas via Heat,      │  │   │
│   │   │   │ (libvirt/KVM)   │   Terraform ou Horizon)  │  │   │
│   │   │   └────────────────┘                          │  │   │
│   │   └────────────────────────────────────────────┘  │   │
│   └──────────────────────────────────────────────────┘   │
│                                                            │
│   cloudflared (túnel) ──────► Internet ──► cirruscloud.eu.org │
└─────────────────────────────────────────────────────────┘
```

**Em resumo:** o notebook roda uma VM (via Multipass) que hospeda o DevStack completo. Dentro dessa VM, o OpenStack pode criar *outras* VMs (as instâncias de verdade que você usa no dia a dia) usando virtualização aninhada (KVM). O painel Horizon é servido via Apache com HTTPS, e um túnel Cloudflare expõe isso à internet sem precisar abrir portas no roteador.

</details>

---

## ✅ Pré-requisitos

- Notebook com pelo menos **16GB de RAM** (recomendado — a VM sozinha usa 12GB)
- [Multipass](https://multipass.run/) instalado
- Conexão de internet estável (a instalação baixa vários pacotes)
- Paciência — a primeira instalação leva de 20 a 40 minutos, e reinstalações parciais são comuns

---

## 🚀 Instalação do zero

### 1. Criar a VM

```bash
multipass launch 24.04 --name devstack-vm --cpus 2 --memory 12G --disk 60G
```

> 💡 Por que Ubuntu 24.04? O DevStack testa oficialmente contra versões **LTS** do Ubuntu. Versões não-LTS (como 25.10) ou muito antigas causam falhas de instalação de pacotes.

### 2. Preparar o usuário `stack`

Dentro da VM (`multipass shell devstack-vm`):

```bash
sudo useradd -s /bin/bash -d /opt/stack -m stack
sudo chmod +x /opt/stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack

# Evita um erro comum de lock do apt durante a instalação
sudo systemctl mask unattended-upgrades
```

### 3. Clonar o DevStack e configurar

```bash
sudo su - stack
git clone https://opendev.org/openstack/devstack
cd devstack

cat > local.conf << 'EOF'
[[local|localrc]]
ADMIN_PASSWORD=SuaSenhaForte123!
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD
HOST_IP=127.0.0.1
LOGFILE=/opt/stack/logs/stack.log
NEUTRON_CREATE_INITIAL_NETWORKS=False
EOF
```

> 💡 `NEUTRON_CREATE_INITIAL_NETWORKS=False` evita um erro conhecido de criação automática de rede que falha por timing na primeira instalação (ver [Problemas conhecidos](#-problemas-conhecidos-e-correções)).

### 4. Rodar a instalação

```bash
./stack.sh
```

Ao final, se tudo correu bem, você verá a URL do Horizon e as credenciais padrão.

### 5. Criar rede básica manualmente

Como desativamos a criação automática, criamos manualmente:

```bash
openstack network create --external --provider-physical-network public --provider-network-type flat rede-publica
openstack subnet create --network rede-publica --subnet-range 172.24.4.0/24 --no-dhcp subnet-publica

openstack network create rede-privada
openstack subnet create --network rede-privada --subnet-range 192.168.100.0/24 subnet-privada

openstack router create meu-roteador
openstack router add subnet meu-roteador subnet-privada
openstack router set --external-gateway rede-publica meu-roteador
```

---

## 🔑 Acesso e credenciais

| Item | Valor padrão |
|---|---|
| Painel web | `https://<IP-DA-VM>/dashboard` |
| Usuário admin | `admin` |
| Senha | a que você definiu em `ADMIN_PASSWORD` |
| CLI | `source ~/devstack/openrc admin admin` |

Para descobrir o IP atual da VM (pode mudar após reinícios):
```bash
multipass list
```

---

## 🎨 Identidade visual (tema CirrusCloud)

O Horizon suporta um sistema de **temas** — pastas que sobrescrevem templates e estilos padrão sem tocar no código-fonte original. O tema do CirrusCloud fica em:

```
/opt/stack/horizon/openstack_dashboard/themes/syscloud/
├── templates/auth/login.html   → estrutura da tela de login
└── static/
    ├── _variables.scss          → paleta de cores (SCSS)
    └── _styles.scss             → todo o CSS customizado
```

Ele é ativado adicionando ao `local_settings.py`:

```python
AVAILABLE_THEMES = [
    ('default', 'Default', 'themes/default'),
    ('material', 'Material', 'themes/material'),
    ('syscloud', 'CirrusCloud', 'themes/syscloud'),
]
DEFAULT_THEME = 'syscloud'
```

Depois de qualquer mudança no tema, é necessário **recompilar os arquivos estáticos**:

```bash
source /opt/stack/data/venv/bin/activate
cd /opt/stack/horizon
python manage.py collectstatic --noinput --clear
python manage.py compress --force
sudo systemctl restart apache2
```

> 💡 Por que esse passo? O Horizon usa `django-compressor`, que combina e minifica todos os arquivos CSS/JS num único bundle para performance. Sem recompilar, o navegador continua servindo a versão em cache antiga.

**Paleta atual:** tons de azul-marinho (`#0a0e1a` → `#0d1526`) com destaque em azul (`#3b82f6`).

---

## 📊 Dashboard customizado

A página "Visão Geral" padrão do Horizon (gráficos de rosca genéricos) foi **totalmente reescrita** para parecer um dashboard de produto real, mantendo os dados **reais** onde possível:

| Seção | Dado |
|---|---|
| Cards de cota (Instâncias, vCPUs, RAM...) | ✅ Real — vem da API de cotas do OpenStack |
| Resumo de Limites (barras de progresso) | ✅ Real |
| Ações Rápidas | ✅ Links funcionais para telas reais |
| Serviços Habilitados | ⚠️ Estático (lista fixa dos serviços instalados) |
| Instâncias Recentes | ✅ Real — tabela nativa do Horizon |
| Uso do Projeto Atual (donut) | ✅ Real — calculado a partir da cota de instâncias |
| **Avisos de Cota** | ✅ Real e dinâmico — mostra alerta automático quando qualquer recurso passa de 80% de uso |

> 💡 **Por que alguns elementos de dashboards "bonitos" (gráficos de linha temporal, sparklines) não têm dado real aqui?** Eles exigiriam um serviço de **telemetria** (Ceilometer/Gnocchi), que não faz parte da instalação padrão do DevStack e representa um projeto à parte, de instalação e configuração significativas.

Arquivo principal:
```
/opt/stack/horizon/openstack_dashboard/dashboards/project/overview/templates/overview/usage.html
```

---

## 🔒 Segurança

Mesmo sendo um ambiente de aprendizado, aplicamos camadas reais de proteção — também como exercício de "pentest no próprio ambiente" (nunca em sistemas de terceiros sem autorização).

### Firewall
```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 6080/tcp   # console VNC
sudo ufw default deny incoming
sudo ufw enable
```

### HTTPS com hardening de TLS
Certificado autoassinado + configuração restritiva:

```apache
SSLProtocol -all +TLSv1.2 +TLSv1.3
SSLCipherSuite HIGH:!aNULL:!MD5:!3DES:!RSA+AES:!RSA+CAMELLIA:!RSA+ARIA
SSLHonorCipherOrder on
SSLCompression off
Header always set Strict-Transport-Security "max-age=63072000"
```

> 💡 **Por que remover cifras `RSA` puras?** Elas não oferecem *forward secrecy* — se a chave privada do servidor for comprometida no futuro, tráfego capturado no passado poderia ser descriptografado retroativamente. Cifras `ECDHE`/`DHE` geram uma chave de sessão nova a cada conexão, mitigando esse risco.

### Verificação
```bash
nmap -sV <IP>                                # portas expostas
nmap --script ssl-enum-ciphers -p 443 <IP>   # qualidade do TLS
```

---

## 🐛 Problemas conhecidos e correções

Esta é a seção mais valiosa para quem for reproduzir ou manter este ambiente. O DevStack **recria os bancos de dados do zero** toda vez que `stack.sh` é executado por completo, o que causa uma cascata previsível de problemas.

<details>
<summary><b>❌ "No project network is available for allocation"</b></summary>

**Causa:** a tabela `ml2_geneve_allocations` do Neutron fica vazia após recriação do banco — sem segmentos de rede disponíveis, nenhuma rede nova pode ser criada.

```bash
mysql -u root -p'SENHA' neutron -e "
INSERT INTO ml2_geneve_allocations (geneve_vni, allocated)
WITH RECURSIVE seq AS (
  SELECT 1 AS n UNION ALL SELECT n + 1 FROM seq WHERE n < 1000
)
SELECT n, 0 FROM seq;
"
```
</details>

<details>
<summary><b>❌ "Access denied" / "403 ACCESS_REFUSED" / "401 Unauthorized"</b></summary>

**Causa:** ao trocar a senha administrativa, serviços já instalados mantêm a senha antiga em arquivos de configuração específicos (ex: `/etc/nova/nova-cpu.conf`).

```bash
mysql -u root -p'SENHA_ANTIGA' -e "ALTER USER 'root'@'localhost' IDENTIFIED BY 'SENHA_NOVA'; FLUSH PRIVILEGES;"
sudo rabbitmqctl change_password stackrabbit 'SENHA_NOVA'
sudo sed -i 's/senha_antiga/senha_nova/g' /etc/nova/nova-cpu.conf
sudo systemctl restart devstack@n-cpu
```
</details>

<details>
<summary><b>❌ "My hypervisor has existing instances, but I appear to be a new service"</b></summary>

**Causa:** o libvirt mantém definições de VMs antigas que o banco recriado do Nova não reconhece mais.

```bash
sudo virsh list --all
sudo virsh undefine <nome-da-instancia-orfa>
sudo systemctl restart devstack@n-cpu
```
</details>

<details>
<summary><b>❌ "Host 'devstack-vm' is not mapped to any cell"</b></summary>

**Causa:** o host de computação não foi re-registrado no mapeamento de células.

```bash
source /opt/stack/data/venv/bin/activate
nova-manage cell_v2 discover_hosts --verbose
```
</details>

<details>
<summary><b>❌ Serviço "zumbi" (Bad Gateway / 503 em algum serviço)</b></summary>

**Causa:** processo antigo do systemd sobrevive a um restart parcial, escutando numa porta diferente da que o Apache espera.

```bash
sudo ss -tlnp | grep <porta_esperada>
sudo kill -9 <PID_antigo>
sudo systemctl restart devstack@<servico>
```
</details>

<details>
<summary><b>❌ Instâncias caem sozinhas / VMs alternam ACTIVE↔SHUTOFF</b></summary>

**Causa:** falta de RAM física real na VM host.

```bash
# No host, fora da VM:
multipass stop devstack-vm
multipass set local.devstack-vm.memory=12G
multipass start devstack-vm
```
</details>

<details>
<summary><b>❌ Tema visual "some" (volta ao padrão OpenStack)</b></summary>

**Causa:** `stack.sh` sobrescreve `local_settings.py` e `horizon.conf`, apagando a ativação do tema e o bloco HTTPS.

Ver seções [Identidade visual](#-identidade-visual-tema-cirruscloud) — reaplicar o bloco `AVAILABLE_THEMES`/`DEFAULT_THEME` e o `<VirtualHost *:443>` do Apache, depois recompilar estáticos.
</details>

---

## 🎼 Orquestração com Heat

**Heat** é o serviço nativo do OpenStack para "infraestrutura como código" — você descreve, em um arquivo YAML, os recursos que quer (redes, servidores, volumes, IPs) e o Heat cria tudo na ordem certa, respeitando dependências.

### Exemplo mínimo

```yaml
heat_template_version: 2018-08-31

description: Template de teste simples - cria uma rede

resources:
  rede_teste:
    type: OS::Neutron::Net
    properties:
      name: rede-heat-teste

outputs:
  rede_id:
    value: { get_resource: rede_teste }
```

```bash
openstack stack create -t template.yaml minha-stack
openstack stack list
openstack stack output show minha-stack --all
```

### Painel visual (Heat Dashboard)

Instalado via `pip install heat-dashboard` — adiciona a seção "Orquestração" ao Horizon, incluindo visualização gráfica (topologia) da infraestrutura criada.

> 🐛 **Bug encontrado e corrigido:** a topologia visual não renderizava porque a função `d3_data()` (responsável por montar os dados do gráfico) chamava a API do Heat passando apenas o *nome* da stack, sem o ID — causando um erro 404 silencioso. Corrigido trocando para o identificador completo (`nome/id`), que é como o Heat realmente espera a consulta.

---

## 🌍 Orquestração com Terraform

Diferente do Heat (exclusivo do OpenStack), o **Terraform** é uma ferramenta de infraestrutura como código **multi-nuvem** — o mesmo conceito e sintaxe parecida funcionam para AWS, Azure, Google Cloud, etc. Aprender a usá-lo aqui é transferível para qualquer outro provedor no futuro.

### Instalação

```bash
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform -y
```

### Estrutura de um projeto Terraform

```hcl
# provider.tf — quem/onde autenticar
provider "openstack" {
  user_name   = "admin"
  password    = "SuaSenha"
  tenant_name = "admin"
  auth_url    = "http://127.0.0.1/identity/v3"
  region      = "RegionOne"
  domain_name = "Default"
  insecure    = true
}
```

```hcl
# main.tf — o que deve existir
resource "openstack_networking_network_v2" "rede" {
  name           = "rede-terraform"
  admin_state_up = "true"
}

resource "openstack_compute_instance_v2" "minha_vm" {
  name        = "vm-terraform-01"
  image_name  = "ubuntu-22.04"
  flavor_name = "m1.small"
  key_pair    = "minha-chave"
  network { uuid = openstack_networking_network_v2.rede.id }
}
```

### Fluxo de trabalho

```bash
terraform init     # baixa o plugin do provider (uma vez por projeto)
terraform plan      # mostra o que vai mudar, sem aplicar nada
terraform apply     # aplica de fato (pede confirmação)
terraform destroy   # remove tudo que o Terraform criou
```

> 💡 **Conceito-chave — o "state":** o Terraform mantém um arquivo (`terraform.tfstate`) com o que ele *acha* que existe. A cada `plan`/`apply`, ele compara esse estado com a infraestrutura real e calcula o diff — só muda o que precisa mudar.

### ⚠️ Conflito de variáveis de ambiente

As variáveis `OS_*` carregadas por `source ~/devstack/openrc` conflitam com a configuração de `provider.tf`. Solução: um wrapper no `.bashrc` que limpa essas variáveis só nas chamadas ao `terraform`:

```bash
terraform() {
  env -u OS_USERNAME -u OS_PASSWORD -u OS_PROJECT_NAME -u OS_AUTH_URL \
      -u OS_REGION_NAME -u OS_USER_DOMAIN_ID -u OS_PROJECT_DOMAIN_ID \
      -u OS_IDENTITY_API_VERSION -u OS_AUTH_TYPE \
      $(which terraform) "$@"
}
```

---

## 🌐 Acesso público (Cloudflare Tunnel)

Para acessar o painel de qualquer lugar (sem VPN, sem abrir porta no roteador), usamos o **Cloudflare Tunnel** — um cliente instalado na própria VM que cria uma conexão de *saída* para a Cloudflare, que então redireciona o tráfego público até o painel local.

### Por que não simplesmente abrir uma porta no roteador?
Port forwarding tradicional exige acesso administrativo ao roteador (nem sempre disponível, como em rede corporativa) e expõe seu IP residencial diretamente à internet. O túnel evita os dois problemas.

### Configuração (túnel nomeado/permanente)

1. Criar conta gratuita na Cloudflare
2. Criar um túnel em **Zero Trust → Networking → Tunnels**
3. Instalar como serviço:
```bash
sudo cloudflared service install <TOKEN>
```
4. Configurar uma rota (**Public Hostname**) apontando para `https://localhost`

O serviço fica rodando permanentemente:
```bash
sudo systemctl status cloudflared
```

---

## 🗺️ Roadmap / pendências

- [ ] Configurar SSH real via chave para acesso via Xshell/terminal externo
- [ ] Resolver perda de configuração (rede, imagens, keypairs) após reinício da VM — candidato natural para automatizar via Terraform
- [ ] Aprovação do domínio próprio (`cirruscloud.eu.org`)
- [ ] Testar Swift (armazenamento de objetos) de ponta a ponta
- [ ] Avaliar instalação de telemetria (Ceilometer/Gnocchi) para gráficos históricos reais no dashboard

---

## 🙏 Créditos e aprendizados

Este projeto foi construído de forma **incremental e iterativa** — cada erro encontrado (e foram muitos) virou uma oportunidade de entender melhor como o OpenStack funciona por baixo do capô: desde a relação entre Nova/Neutron/Glance até detalhes de como o Django Compressor, o systemd e o libvirt se encaixam nesse quebra-cabeça.

Se você está reproduzindo este processo: **os erros fazem parte**. A seção [Problemas conhecidos](#-problemas-conhecidos-e-correções) existe justamente porque quase nenhuma etapa deu certo de primeira — e tudo bem.

---

*Última atualização: agosto de 2026.*

---

## 📎 Apêndice — Arquivos completos (reprodutibilidade)

Esta seção contém o **conteúdo integral e consolidado** dos arquivos customizados, para quem quiser reproduzir o visual exatamente como está hoje, sem precisar reconstruir peça por peça.

> ⚠️ Os arquivos abaixo representam o **estado final consolidado** (a versão azul-marinho, que substituiu uma tentativa anterior em roxo/fúcsia ao longo do desenvolvimento). Cole cada um no caminho indicado.

### A.1 — Tema: `_variables.scss`

Caminho: `/opt/stack/horizon/openstack_dashboard/themes/syscloud/static/_variables.scss`

```scss
@import "../default/variables";

// Paleta Azul-marinho (definitiva)
$syscloud-navy-dark: #0a0e1a;
$syscloud-navy: #0d1526;
$syscloud-blue-accent: #3b82f6;
$syscloud-blue-light: #60a5fa;

// Navbar customizado
$navbar-default-bg: $syscloud-navy-dark !default;
$navbar-default-color: #fff !default;
$navbar-default-link-color: #cfe3ff !default;
$navbar-default-link-hover-color: #fff !default;
$navbar-default-link-hover-bg: $syscloud-navy !default;
$navbar-default-brand-color: #fff !default;
$navbar-default-border: $syscloud-navy !default;
```

### A.2 — Tema: `login.html`

Caminho: `/opt/stack/horizon/openstack_dashboard/themes/syscloud/templates/auth/login.html`

```html
{% extends "auth/login.html" %}
{% load i18n %}

{% block content %}
<div class="syscloud-login-wrapper">
  <div class="syscloud-left">
    <div class="syscloud-logo">
      <svg width="48" height="48" viewBox="0 0 48 48" fill="none" xmlns="http://www.w3.org/2000/svg">
        <path d="M35 20c-.3-5-4.6-9-9.8-9-4 0-7.5 2.4-9 5.9C11.3 17.4 8 20.8 8 25c0 4.4 3.6 8 8 8h19c3.9 0 7-3.1 7-7 0-3.5-2.6-6.5-6-6.9z" stroke="#3b82f6" stroke-width="2" fill="none"/>
      </svg>
      Cirrus<span class="accent">Cloud</span>
    </div>
    <p class="syscloud-tagline-small">Infraestrutura em Nuvem</p>

    <h1 class="syscloud-hero-title">Infraestrutura em nuvem.<br>Simples, segura e escalável.</h1>
    <p class="syscloud-hero-desc">Plataforma completa para construir, gerenciar e escalar sua infraestrutura com performance e segurança.</p>

    <div class="syscloud-features">
      <div class="feature">
        <div class="feature-icon">
          <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="#3b82f6" stroke-width="2"><path d="M3 17l6-6 4 4 8-8"/><path d="M17 7h4v4"/></svg>
        </div>
        <div class="feature-text">
          <div class="feature-title">Escalabilidade</div>
          <div class="feature-desc">Recursos sob demanda para acompanhar o crescimento da sua empresa.</div>
        </div>
      </div>
      <div class="feature">
        <div class="feature-icon">
          <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="#3b82f6" stroke-width="2"><path d="M12 2l8 3v6c0 5-3.5 9-8 11-4.5-2-8-6-8-11V5z"/></svg>
        </div>
        <div class="feature-text">
          <div class="feature-title">Segurança</div>
          <div class="feature-desc">Proteção de dados e infraestrutura com os mais altos padrões do mercado.</div>
        </div>
      </div>
      <div class="feature">
        <div class="feature-icon">
          <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="#3b82f6" stroke-width="2"><circle cx="12" cy="12" r="9"/><path d="M12 12l4-4"/></svg>
        </div>
        <div class="feature-text">
          <div class="feature-title">Performance</div>
          <div class="feature-desc">Alta disponibilidade e performance para suas aplicações e serviços.</div>
        </div>
      </div>
    </div>

    <div class="syscloud-illustration">
      <svg width="280" height="220" viewBox="0 0 280 220" fill="none" xmlns="http://www.w3.org/2000/svg">
        <path d="M180 90c-1-16-15-29-32-29-13 0-24 8-29 19-14 1-25 12-25 26 0 14 12 26 26 26h60c13 0 23-10 23-23 0-11-8-20-19-22z" stroke="#3b82f6" stroke-width="2" opacity="0.6"/>
        <rect x="90" y="100" width="100" height="110" rx="4" stroke="#3b82f6" stroke-width="2" opacity="0.8"/>
        <line x1="100" y1="120" x2="180" y2="120" stroke="#3b82f6" stroke-width="1.5" opacity="0.5"/>
        <line x1="100" y1="140" x2="180" y2="140" stroke="#3b82f6" stroke-width="1.5" opacity="0.5"/>
        <line x1="100" y1="160" x2="180" y2="160" stroke="#3b82f6" stroke-width="1.5" opacity="0.5"/>
        <line x1="100" y1="180" x2="180" y2="180" stroke="#3b82f6" stroke-width="1.5" opacity="0.5"/>
        <circle cx="112" cy="120" r="2.5" fill="#3b82f6"/>
        <circle cx="112" cy="140" r="2.5" fill="#3b82f6"/>
        <circle cx="112" cy="160" r="2.5" fill="#3b82f6"/>
        <circle cx="112" cy="180" r="2.5" fill="#3b82f6"/>
      </svg>
    </div>
  </div>

  <div class="syscloud-right">
    <div class="syscloud-card">
      <h2>Bem-vindo(a)</h2>
      <p class="syscloud-subtitle">Acesse sua conta para continuar</p>

      <form method="POST" action="{% url 'login' %}" autocomplete="off" ng-controller="hzLoginController">
        {% csrf_token %}
        <fieldset hz-login-finder>
          {% if logout_reason %}
            <div class="alert alert-danger">{{ logout_reason }}</div>
          {% endif %}
          {% if next %}
            <input type="hidden" name="{{ redirect_field_name }}" value="{{ next }}" />
          {% endif %}
          {% include "horizon/common/_form_fields.html" %}
        </fieldset>
        <button id="loginBtn" type="submit" class="syscloud-btn">Entrar</button>
      </form>

      <p class="syscloud-footer-text">&copy; 2026 CirrusCloud. Todos os direitos reservados.</p>
    </div>
  </div>
</div>
{% endblock %}
```

### A.3 — Tema: `_styles.scss` (consolidado)

Caminho: `/opt/stack/horizon/openstack_dashboard/themes/syscloud/static/_styles.scss`

```scss
@import "variables";

/* ===== Login (azul-marinho) ===== */
.syscloud-login-wrapper {
  display: flex;
  min-height: 100vh;
  width: 100%;
  background: linear-gradient(160deg, $syscloud-navy-dark 0%, $syscloud-navy 100%);
  color: #fff;
  font-family: Arial, sans-serif;
}

.syscloud-left {
  flex: 1.3;
  padding: 60px 60px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  position: relative;
}

.syscloud-logo {
  display: flex;
  align-items: center;
  gap: 12px;
  font-size: 32px;
  font-weight: bold;
  margin-bottom: 4px;
}
.syscloud-logo .accent { color: $syscloud-blue-accent; }

.syscloud-tagline-small {
  color: #94a3b8;
  font-size: 14px;
  margin: 4px 0 32px 60px;
}

.syscloud-hero-title {
  font-size: 30px;
  font-weight: 700;
  color: #fff;
  line-height: 1.3;
  margin-bottom: 16px;
}

.syscloud-hero-desc {
  color: #94a3b8;
  font-size: 15px;
  max-width: 420px;
  margin-bottom: 32px;
}

.syscloud-features .feature {
  display: flex;
  gap: 14px;
  margin-bottom: 20px;
  align-items: flex-start;
}

.feature-icon {
  background: rgba(59, 130, 246, 0.12);
  border-radius: 8px;
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.feature-title {
  color: #fff;
  font-weight: 600;
  font-size: 15px;
  margin-bottom: 2px;
}

.feature-desc {
  color: #94a3b8;
  font-size: 13px;
}

.syscloud-illustration {
  position: absolute;
  right: 40px;
  bottom: 20px;
  opacity: 0.85;
}

.syscloud-right {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 40px;
}

.syscloud-card {
  background: rgba(13, 21, 38, 0.9);
  border: 1px solid rgba(59, 130, 246, 0.15);
  border-radius: 14px;
  padding: 40px;
  width: 100%;
  max-width: 420px;
}

.syscloud-card h2 { color: #fff; text-align: center; margin-bottom: 8px; }
.syscloud-subtitle { text-align: center; color: #94a3b8; margin-bottom: 30px; font-size: 14px; }

.syscloud-card input {
  width: 100%;
  padding: 10px 14px;
  margin-bottom: 16px;
  border-radius: 6px;
  border: 1px solid rgba(255,255,255,0.2);
  background: rgba(255,255,255,0.05);
  color: #fff;
}
.syscloud-card label { color: #cfe3ff; font-size: 13px; }

.syscloud-btn {
  width: 100%;
  padding: 12px;
  background: linear-gradient(90deg, $syscloud-blue-accent, $syscloud-blue-light);
  border: none;
  border-radius: 8px;
  color: #fff;
  font-weight: bold;
  font-size: 15px;
  cursor: pointer;
  margin-top: 10px;
}

.syscloud-footer-text { text-align: center; color: #64748b; font-size: 12px; margin-top: 24px; }

/* ===== Navbar / Sidebar internos ===== */
.navbar-default { background: $syscloud-navy-dark; border-color: $syscloud-navy; }
.navbar-default .navbar-nav > li > a,
.navbar-default .context-project,
.navbar-default .user-name,
.navbar-default a, .navbar-default span { color: #ffffff !important; }
.navbar-default .navbar-nav > li > a:hover { color: $syscloud-blue-light !important; }

body, #content_body, .container-fluid { background-color: $syscloud-navy-dark !important; }

#sidebar { background-color: $syscloud-navy-dark !important; }
#sidebar-accordion .list-group-item.openstack-panel,
#sidebar-accordion li.panel > a,
#sidebar-accordion li.openstack-panel-group > a {
  background-color: $syscloud-navy-dark !important;
  color: #94a3b8 !important;
  border-color: #1f2937 !important;
}
#sidebar-accordion .list-group-item.openstack-panel:hover,
#sidebar-accordion li.panel > a:hover {
  background-color: rgba(59,130,246,0.1) !important;
  color: $syscloud-blue-light !important;
}
#sidebar-accordion .list-group-item.openstack-panel.active {
  background-color: rgba(59,130,246,0.15) !important;
  color: $syscloud-blue-light !important;
  border-left: 3px solid $syscloud-blue-accent !important;
}
#sidebar-accordion > li.panel > a { color: #fff !important; font-weight: 600; }

.page-breadcrumb, .breadcrumb { background: transparent !important; color: #64748b !important; }
.breadcrumb a { color: $syscloud-blue-light !important; }
.breadcrumb .active { color: #cbd5e1 !important; }
h1, .page-header h1 { color: #fff !important; }

/* ===== Tabelas ===== */
.table { background-color: #111827 !important; color: #e2e8f0 !important; }
.table > thead > tr > th {
  background-color: $syscloud-navy-dark !important;
  color: #cbd5e1 !important;
  border-bottom: 2px solid $syscloud-blue-accent !important;
}
.table > tbody > tr { background-color: #111827 !important; color: #e2e8f0 !important; }
.table > tbody > tr:hover { background-color: rgba(59,130,246,0.08) !important; }
.table-striped > tbody > tr:nth-of-type(odd) { background-color: #0f172a !important; }
.table > tbody > tr > td { border-color: #1f2937 !important; }
.table a { color: $syscloud-blue-light !important; }
.table tr.empty, .table tr.empty td { background-color: #0f172a !important; color: #94a3b8 !important; }

.btn-primary { background-color: $syscloud-blue-accent; border-color: $syscloud-blue-accent; }
.btn-primary:hover { background-color: $syscloud-blue-light; border-color: $syscloud-blue-light; }

/* ===== Painel "Utilização" (usage summary) ===== */
.quota-heading { color: #fff !important; }
.usage_info_wrapper {
  background: #111827 !important;
  border-color: #1f2937 !important;
  color: #e2e8f0 !important;
  padding: 16px;
  border-radius: 8px;
}
.usage_info_wrapper label, .usage_info_wrapper .small { color: #cbd5e1 !important; }
#activity dt { color: #94a3b8 !important; }
#activity dd { color: #fff !important; font-weight: 600; }
.usage_info_wrapper input { background: $syscloud-navy-dark !important; color: #fff !important; border-color: #1f2937 !important; }

/* ===== Acessibilidade ===== */
a:hover { text-decoration: underline !important; }
a:focus, button:focus, input:focus { outline: 2px solid $syscloud-blue-light !important; outline-offset: 2px; }

/* ===== Dashboard interno CirrusCloud (classes cc-*) ===== */
.cc-dashboard { padding: 8px 4px; }
.cc-dash-header h1 { font-size: 26px; font-weight: 700; color: #fff !important; margin-bottom: 4px; }
.cc-dash-header p { color: #94a3b8 !important; margin-bottom: 24px; }

.cc-stat-cards { display: grid; grid-template-columns: repeat(auto-fit, minmax(160px, 1fr)); gap: 16px; margin-bottom: 24px; }
.cc-stat-card { background: #111827 !important; border: 1px solid #1f2937 !important; border-radius: 10px; padding: 18px; }
.cc-stat-icon { width: 36px; height: 36px; border-radius: 8px; background: rgba(59,130,246,0.15) !important; color: $syscloud-blue-light !important; display: flex; align-items: center; justify-content: center; margin-bottom: 12px; }
.cc-stat-name { color: #94a3b8 !important; font-size: 13px; margin-bottom: 4px; }
.cc-stat-value { font-size: 24px; font-weight: 700; color: #fff !important; }
.cc-stat-total { font-size: 13px; font-weight: 400; color: #64748b !important; }

.cc-dash-row { display: flex; gap: 20px; margin-bottom: 20px; align-items: stretch; }
.cc-panel { background: #111827 !important; border: 1px solid #1f2937 !important; border-radius: 10px; flex: 1; min-width: 260px; }
.cc-panel-wide { flex: 2.2; }
.cc-panel-full { flex: 1 1 100%; }
.cc-panel-header { padding: 16px 20px; border-bottom: 1px solid #1f2937 !important; }
.cc-panel-header h3 { font-size: 15px; font-weight: 600; color: #fff !important; margin: 0; }
.cc-panel-body { padding: 18px 20px; }
.cc-section-title { font-size: 13px; color: #64748b !important; margin: 12px 0 10px; text-transform: uppercase; letter-spacing: 0.03em; }

.cc-limit-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 14px; margin-bottom: 10px; }
.cc-limit-bar-bg { background: #1f2937 !important; border-radius: 6px; height: 6px; overflow: hidden; margin-bottom: 6px; }
.cc-limit-bar-fill { background: linear-gradient(90deg, $syscloud-blue-accent, $syscloud-blue-light); height: 100%; border-radius: 6px; }
.cc-limit-label { display: flex; justify-content: space-between; font-size: 12px; color: #e2e8f0 !important; }

.cc-quick-actions { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
.cc-action-btn { display: flex; flex-direction: column; align-items: center; gap: 8px; padding: 16px 8px; background: #0f172a !important; border: 1px solid #1f2937 !important; border-radius: 8px; color: $syscloud-blue-light !important; text-decoration: none; font-size: 12px; text-align: center; }
.cc-action-btn:hover { background: rgba(59,130,246,0.12) !important; text-decoration: none; }

.cc-service-list { display: grid; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr)); gap: 10px; }
.cc-service-item { display: flex; justify-content: space-between; padding: 10px 14px; background: #0f172a !important; border-radius: 6px; font-size: 13px; color: #cbd5e1 !important; }
.cc-badge-ok { color: #4ade80 !important; display: inline-flex; align-items: center; gap: 4px; }
.cc-badge-ok::before { content: "✓"; font-weight: 700; }
.cc-dot-warn { background: #fbbf24; }

.cc-donut-wrap { display: flex; justify-content: center; padding: 20px 0; }
.cc-donut { width: 160px; height: 160px; border-radius: 50%; display: flex; align-items: center; justify-content: center; }
.cc-donut-hole { width: 110px; height: 110px; background: #111827; border-radius: 50%; display: flex; flex-direction: column; align-items: center; justify-content: center; }
.cc-donut-pct { font-size: 24px; font-weight: 700; color: #fff; }
.cc-donut-label { font-size: 12px; color: #94a3b8; }

.cc-notif-item { display: flex; gap: 10px; padding: 10px 0; border-bottom: 1px solid #1f2937; }
.cc-notif-item:last-child { border-bottom: none; }
.cc-notif-dot { width: 8px; height: 8px; border-radius: 50%; margin-top: 5px; flex-shrink: 0; }
.cc-dot-ok { background: #4ade80; }
.cc-dot-info { background: #60a5fa; }
.cc-notif-title { color: #e2e8f0; font-size: 13px; font-weight: 600; }
.cc-notif-time { color: #94a3b8; font-size: 12px; }
```

### A.4 — Dashboard: `usage.html`

Caminho: `/opt/stack/horizon/openstack_dashboard/dashboards/project/overview/templates/overview/usage.html`

```html
{% extends 'base.html' %}
{% load i18n horizon %}
{% block title %}{% trans "Dashboard" %}{% endblock %}

{% block main %}
<div class="cc-dashboard">

  <div class="cc-dash-header">
    <h1>Dashboard</h1>
    <p>Visão geral da sua infraestrutura na CirrusCloud.</p>
  </div>

  <div class="cc-stat-cards">
    {% for section in charts %}
      {% for chart in section.charts %}
        <div class="cc-stat-card">
          <div class="cc-stat-icon">
            <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <rect x="3" y="3" width="7" height="7" rx="1"/>
              <rect x="14" y="3" width="7" height="7" rx="1"/>
              <rect x="3" y="14" width="7" height="7" rx="1"/>
              <rect x="14" y="14" width="7" height="7" rx="1"/>
            </svg>
          </div>
          <div class="cc-stat-name">{{ chart.name }}</div>
          <div class="cc-stat-value">{{ chart.used_display }}
            {% if chart.quota|quotainf != '-1' %}
              <span class="cc-stat-total">de {{ chart.quota_display }}</span>
            {% endif %}
          </div>
        </div>
      {% endfor %}
    {% endfor %}
  </div>

  <div class="cc-dash-row">
    <div class="cc-panel cc-panel-wide">
      <div class="cc-panel-header"><h3>Resumo de Limites</h3></div>
      <div class="cc-panel-body">
        {% for section in charts %}
          <h4 class="cc-section-title">{{ section.title }}</h4>
          <div class="cc-limit-grid">
            {% for chart in section.charts %}
              <div class="cc-limit-item">
                <div class="cc-limit-bar-bg">
                  <div class="cc-limit-bar-fill" style="width: {% quotapercent chart.used chart.quota %}%;"></div>
                </div>
                <div class="cc-limit-label">
                  <span>{{ chart.name }}</span>
                  <span class="cc-limit-numbers">
                    {% if chart.quota|quotainf != '-1' %}
                      {{ chart.used_display }} / {{ chart.quota_display }}
                    {% else %}
                      {{ chart.used_display }} (sem limite)
                    {% endif %}
                  </span>
                </div>
              </div>
            {% endfor %}
          </div>
        {% endfor %}
      </div>
    </div>

    <div class="cc-panel">
      <div class="cc-panel-header"><h3>Ações Rápidas</h3></div>
      <div class="cc-panel-body cc-quick-actions">
        <a href="{% url 'horizon:project:instances:index' %}" class="cc-action-btn">
          <svg width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="2" y="3" width="20" height="14" rx="2"/><path d="M8 21h8M12 17v4"/></svg>
          <span>Nova Instância</span>
        </a>
        <a href="{% url 'horizon:project:volumes:index' %}" class="cc-action-btn">
          <svg width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><ellipse cx="12" cy="5" rx="9" ry="3"/><path d="M3 5v14c0 1.7 4 3 9 3s9-1.3 9-3V5"/></svg>
          <span>Novo Volume</span>
        </a>
        <a href="{% url 'horizon:project:networks:index' %}" class="cc-action-btn">
          <svg width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="5" r="2"/><circle cx="5" cy="19" r="2"/><circle cx="19" cy="19" r="2"/><path d="M12 7v6M12 13l-7 4M12 13l7 4"/></svg>
          <span>Nova Rede</span>
        </a>
        <a href="{% url 'horizon:project:images:index' %}" class="cc-action-btn">
          <svg width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="3" width="18" height="18" rx="2"/><circle cx="8.5" cy="8.5" r="1.5"/><path d="M21 15l-5-5L5 21"/></svg>
          <span>Carregar Imagem</span>
        </a>
      </div>
    </div>
  </div>

  <div class="cc-dash-row">
    <div class="cc-panel cc-panel-full">
      <div class="cc-panel-header"><h3>Serviços Habilitados</h3></div>
      <div class="cc-panel-body">
        <div class="cc-service-list">
          <div class="cc-service-item"><span>Nova (Compute)</span><span class="cc-badge-ok">Ativo</span></div>
          <div class="cc-service-item"><span>Neutron (Rede)</span><span class="cc-badge-ok">Ativo</span></div>
          <div class="cc-service-item"><span>Cinder (Volume)</span><span class="cc-badge-ok">Ativo</span></div>
          <div class="cc-service-item"><span>Glance (Imagem)</span><span class="cc-badge-ok">Ativo</span></div>
          <div class="cc-service-item"><span>Keystone (Identidade)</span><span class="cc-badge-ok">Ativo</span></div>
          <div class="cc-service-item"><span>Heat (Orquestração)</span><span class="cc-badge-ok">Ativo</span></div>
          <div class="cc-service-item"><span>Swift (Objeto)</span><span class="cc-badge-ok">Ativo</span></div>
        </div>
      </div>
    </div>
  </div>

  {% if simple_tenant_usage_enabled %}
    <div class="cc-dash-row">
      <div class="cc-panel cc-panel-full">
        <div class="cc-panel-header"><h3>Instâncias Recentes</h3></div>
        <div class="cc-panel-body">{{ table.render }}</div>
      </div>
    </div>
  {% endif %}

  <div class="cc-dash-row">
    <div class="cc-panel">
      <div class="cc-panel-header"><h3>Uso do Projeto Atual</h3></div>
      <div class="cc-panel-body cc-donut-wrap">
        {% for section in charts %}
          {% for chart in section.charts %}
            {% if chart.name == "Instances" or chart.name == "Instâncias" %}
              <div class="cc-donut" style="background: conic-gradient(#3b82f6 0% {% quotapercent chart.used chart.quota %}%, #1f2937 {% quotapercent chart.used chart.quota %}% 100%);">
                <div class="cc-donut-hole">
                  <span class="cc-donut-pct">{% quotapercent chart.used chart.quota %}%</span>
                  <span class="cc-donut-label">instâncias</span>
                </div>
              </div>
            {% endif %}
          {% endfor %}
        {% endfor %}
      </div>
    </div>

    <div class="cc-panel">
      <div class="cc-panel-header"><h3>Avisos de Cota</h3></div>
      <div class="cc-panel-body">
        {% for section in charts %}
          {% for chart in section.charts %}
            {% quotapercent chart.used chart.quota as pct %}
            {% if pct >= 80 %}
              <div class="cc-notif-item">
                <span class="cc-notif-dot cc-dot-warn"></span>
                <div><div class="cc-notif-title">{{ chart.name }} próximo do limite</div><div class="cc-notif-time">{{ chart.used_display }} de {{ chart.quota_display }} utilizados ({{ pct }}%)</div></div>
              </div>
            {% endif %}
          {% endfor %}
        {% endfor %}
        <div class="cc-notif-item">
          <span class="cc-notif-dot cc-dot-ok"></span>
          <div><div class="cc-notif-title">Ambiente OpenStack ativo</div><div class="cc-notif-time">Todos os serviços principais operando normalmente</div></div>
        </div>
      </div>
    </div>
  </div>

</div>
{% endblock %}
```

### A.5 — Ativação do tema em `local_settings.py`

Adicionar ao final de `/opt/stack/horizon/openstack_dashboard/local/local_settings.py`:

```python
AVAILABLE_THEMES = [
    ('default', 'Default', 'themes/default'),
    ('material', 'Material', 'themes/material'),
    ('syscloud', 'CirrusCloud', 'themes/syscloud'),
]
DEFAULT_THEME = 'syscloud'
```

### A.6 — Heat: unidades systemd (instalação manual)

Necessárias quando o Heat é instalado **sem** rerun completo do `stack.sh` (ver seção [Orquestração com Heat](#-orquestração-com-heat)).

`/etc/systemd/system/devstack@heat-api.service`
```ini
[Unit]
Description=Devstack devstack@heat-api.service
[Service]
Type=notify
NotifyAccess=all
Restart=always
KillMode=process
User=stack
ExecStart=/bin/uwsgi --procname-prefix heat-api --ini /etc/heat/heat-api-uwsgi.ini --venv /opt/stack/data/venv
[Install]
WantedBy=multi-user.target
```

`/etc/systemd/system/devstack@heat-api-cfn.service`
```ini
[Unit]
Description=Devstack devstack@heat-api-cfn.service
[Service]
Type=notify
NotifyAccess=all
Restart=always
KillMode=process
User=stack
ExecStart=/bin/uwsgi --procname-prefix heat-api-cfn --ini /etc/heat/heat-api-cfn-uwsgi.ini --venv /opt/stack/data/venv
[Install]
WantedBy=multi-user.target
```

`/etc/systemd/system/devstack@heat-engine.service`
```ini
[Unit]
Description=Devstack devstack@heat-engine.service
[Service]
Type=simple
Restart=always
User=stack
ExecStart=/opt/stack/data/venv/bin/heat-engine --config-file /etc/heat/heat.conf
[Install]
WantedBy=multi-user.target
```

Depois de criar os três arquivos:
```bash
sudo systemctl daemon-reload
sudo systemctl enable devstack@heat-api devstack@heat-api-cfn devstack@heat-engine
sudo systemctl start devstack@heat-api devstack@heat-api-cfn devstack@heat-engine
```

> 💡 Nota importante: repare no caminho `/bin/uwsgi` (não `/opt/stack/data/venv/bin/uwsgi`) — esse foi um erro comum cometido durante o desenvolvimento. O binário `uwsgi` do sistema é o correto; o venv é passado apenas como parâmetro `--venv`.

### A.7 — Bug corrigido no Heat Dashboard

Arquivo: `/opt/stack/data/venv/lib/python3.12/site-packages/heat_dashboard/content/stacks/api.py`

Localizar a linha:
```python
resources = heat.resources_list(request, stack.stack_name)
```

E substituir por:
```python
resources = heat.resources_list(request, '{}/{}'.format(stack.stack_name, stack.id))
```

Depois, limpar cache de bytecode e reiniciar:
```bash
sudo find /opt/stack/data/venv/lib/python3.12/site-packages/heat_dashboard -name "*.pyc" -delete
sudo find /opt/stack/data/venv/lib/python3.12/site-packages/heat_dashboard -name "__pycache__" -type d -exec rm -rf {} + 2>/dev/null
sudo systemctl restart apache2
```
