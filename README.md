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
