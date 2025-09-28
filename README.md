# 🖥️ Infraestrutura WordPress - AWS

![Status](https://img.shields.io/badge/Status-100%25%20Conclu%C3%ADdo-brightgreen)
![AWS](https://img.shields.io/badge/AWS-Infraestrutura-orange)
![WordPress](https://img.shields.io/badge/WordPress-Docker-blue)

![Image](https://github.com/user-attachments/assets/0e5e06f0-90c2-4b97-a66e-6463086840ce)

## 📋 Visão Geral

Este projeto implementa uma infraestrutura completa e escalável para WordPress na AWS, seguindo as melhores práticas de:

- 🔒 **Segurança** - Grupos de segurança em camadas
- 📈 **Escalabilidade** - Auto Scaling automático
- 🏗️ **Alta Disponibilidade** - Multi-AZ deployment
- 🐳 **Containerização** - WordPress em Docker

---

## 🏗️ Arquitetura da Solução

```
┌─────────────────────────────────────────────────────────┐
│                    Internet Gateway                      │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────┴───────────────────────────────┐
│                Application Load Balancer                │
│              (Public Subnets - Multi-AZ)               │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────┴───────────────────────────────┐
│              Auto Scaling Group (EC2)                  │
│           WordPress Containers + EFS Mount              │
│              (Private Subnets - Multi-AZ)              │
└─────────────────────────┬───────────────────────────────┘
                          │
          ┌───────────────┴────────────────┐
          │                                │
┌─────────▼─────────┐              ┌───────▼────────┐
│   RDS MySQL       │              │      EFS       │
│ (Private Subnets) │              │ (Regional HA)  │
└───────────────────┘              └────────────────┘
```

---

## 🚀 Etapas da Implementação

### ✅ Etapa 1: Networking (VPC)

Criação da infraestrutura de rede base:

| Componente | Configuração |
|------------|-------------|
| **VPC** | `WordPress-VPC` - CIDR: `10.0.0.0/16` |
| **Availability Zones** | `us-east-1a` e `us-east-1b` |
| **Subnets** | 4 total (2 públicas + 2 privadas) |
| **Internet Gateway** | Acesso público para ALB |
| **NAT Gateways** | 2 unidades (uma por AZ) |
| **Route Tables** | Configuradas automaticamente |

---

### 🔒 Etapa 2: Segurança (Security Groups)

Implementação de arquitetura de segurança em camadas:

| Security Group | Função | Porta | Origem |
|----------------|--------|-------|---------|
| `alb-sg-wordpress` | Load Balancer | 80, 443 | `0.0.0.0/0` |
| `ec2-sg-wordpress` | Instâncias EC2 | 80 | `alb-sg-wordpress` |
| `rds-sg-wordpress` | Banco de dados | 3306 | `ec2-sg-wordpress` |
| `efs-sg-wordpress` | Sistema de arquivos | 2049 | `ec2-sg-wordpress` |

> 🛡️ **Princípio do Menor Privilégio**: Cada camada só aceita tráfego da camada anterior

---

### 💾 Etapa 3: Armazenamento (EFS)

Sistema de arquivos compartilhado para WordPress:

- **Nome**: `wordpress-efs`
- **Tipo**: Regional (Multi-AZ)
- **Mount Targets**: Subnets privadas
- **Proteção**: `efs-sg-wordpress`

---

### 🗄️ Etapa 4: Banco de Dados (RDS)

Instância MySQL isolada e segura:

- **Nome**: `wordpress-db`
- **Engine**: MySQL `db.t3.micro`
- **Localização**: Subnets privadas
- **Subnet Group**: Configurado automaticamente
- **Acesso Público**: ❌ Desabilitado
- **Banco Inicial**: `wordpress`

---

### 🐳 Etapa 5: Template de Lançamento

Launch Template com automação Docker:

- **Nome**: `wordpress-docker-lt`
- **SO**: Ubuntu 22.04 LTS
- **Automação**: Script user-data completo

#### 📜 Script de Inicialização

<details>
<summary>🔽 Clique para expandir o script user-data</summary>

```bash
#!/bin/bash
set -e

# Atualiza a lista de pacotes e instala pré-requisitos
apt-get update -y
apt-get install -y apt-transport-https ca-certificates curl software-properties-common nfs-common

# Instala o Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update -y
apt-get install -y docker-ce docker-ce-cli containerd.io

# Inicia e habilita o serviço do Docker
systemctl start docker
systemctl enable docker

# Adiciona o usuário ubuntu ao grupo docker
usermod -a -G docker ubuntu

# Instala o Docker Compose
DOCKER_COMPOSE_VERSION="v2.20.2"
curl -L "https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Monta o sistema de arquivos EFS
EFS_ID="<SEU_EFS_ID>"
EFS_REGION="us-east-1"
EFS_MOUNT_POINT="/mnt/efs/wordpress"
mkdir -p ${EFS_MOUNT_POINT}
mount -t nfs -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport ${EFS_ID}.efs.${EFS_REGION}.amazonaws.com:/ ${EFS_MOUNT_POINT}
echo "${EFS_ID}.efs.${EFS_REGION}.amazonaws.com:/ ${EFS_MOUNT_POINT} nfs4 defaults,_netdev 0 0" >> /etc/fstab

# Cria o arquivo docker-compose.yml
cat <<EOF > /home/ubuntu/docker-compose.yml
version: '3.8'
services:
  wordpress:
    image: wordpress:latest
    container_name: wordpress
    restart: always
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: "<SEU_ENDPOINT_RDS>:3306"
      WORDPRESS_DB_USER: "<SEU_USUARIO_RDS>"
      WORDPRESS_DB_PASSWORD: "<SUA_SENHA_RDS>"
      WORDPRESS_DB_NAME: "<SEU_NOME_DO_BANCO>"
    volumes:
      - ${EFS_MOUNT_POINT}:/var/www/html
EOF

# Concede permissões ao usuário ubuntu
chown -R ubuntu:ubuntu ${EFS_MOUNT_POINT}
chown ubuntu:ubuntu /home/ubuntu/docker-compose.yml

# Executa o Docker Compose
/usr/local/bin/docker-compose -f /home/ubuntu/docker-compose.yml up -d
```

</details>

---

### ⚖️ Etapa 6: Load Balancing & Auto Scaling

Componentes para alta disponibilidade:

| Componente | Configuração | Função |
|------------|-------------|---------|
| **Target Group** | `wordpress-tg` | Agrupa instâncias EC2 |
| **Application Load Balancer** | `wordpress-alb` | Distribui tráfego (subnets públicas) |
| **Auto Scaling Group** | `wordpress-asg` | Gerencia e escala instâncias |

**Configuração ASG:**
- 📊 **Capacidade Desejada**: 2 instâncias
- 📈 **Política de Escalonamento**: Baseada em CPU
- 🌍 **Distribuição**: Multi-AZ (subnets privadas)

---

## ✅ Status do Projeto

<div align="center">

<img width="1919" height="986" alt="Image" src="https://github.com/user-attachments/assets/575303ac-9b50-4b13-94cd-63b60587d3bc" />

<img width="1919" height="986" alt="Image" src="https://github.com/user-attachments/assets/263a4f16-b59c-49c6-beb5-91dc28cfc37d" />

<img width="1919" height="950" alt="Image" src="https://github.com/user-attachments/assets/0ab823f0-143f-4a1d-9741-538e7e37cfdc" />

</div>

✅ Infraestrutura provisionada  
✅ Ambiente testado e validado  
✅ WordPress online via ALB  
✅ Auto Scaling configurado  
✅ Alta disponibilidade Multi-AZ  

</div>

---

## 💰 Gerenciamento de Custos

### ⏸️ Para Pausar/Reduzir Custos

| Recurso | Ação Recomendada | Impacto no Custo |
|---------|------------------|------------------|
| **Auto Scaling Group** | Definir Min/Desired/Max = `0` | 🔴 **Alto** - Encerra EC2 |
| **Application Load Balancer** | Deletar | 🟡 **Médio** - Cobrança por hora |
| **RDS Instance** | Parar (Stop) | 🟡 **Médio** - Reduz compute |
| **NAT Gateways** | Deletar | 🟡 **Médio** - Cobrança por hora/tráfego |
| **Elastic IPs** | Liberar (Release) | 🟢 **Baixo** - IPs não utilizados |

> ⚠️ **Importante**: Alguns recursos (EFS, VPC, Security Groups) têm custo zero ou muito baixo quando não utilizados.

---

## 📊 Monitoramento e Observabilidade

### 📈 Métricas Principais
- **CPU Utilization** (EC2/RDS)
- **Network I/O** (ALB/EC2)
- **Database Connections** (RDS)
- **EFS Throughput**

### 🚨 Alertas Configurados
- Auto Scaling baseado em CPU
- Health Checks do Target Group
- RDS Connection Monitoring

---

## 🔧 Troubleshooting

<details>
<summary>🔽 Problemas Comuns e Soluções</summary>

### WordPress não carrega
1. Verificar Health Check do Target Group
2. Conferir logs do Docker: `docker logs wordpress`
3. Validar conectividade RDS

### Problemas de montagem EFS
1. Verificar Security Groups NFS (porta 2049)
2. Confirmar EFS Mount Targets nas AZs corretas
3. Testar conectividade: `ping <efs-id>.efs.us-east-1.amazonaws.com`

### Auto Scaling não funciona
1. Verificar métricas CloudWatch
2. Conferir políticas de escalonamento
3. Validar Launch Template

</details>

---

## 📝 Logs e Documentação

### 📂 Estrutura de Arquivos
```
/home/ubuntu/
├── docker-compose.yml     # Configuração do WordPress
└── /mnt/efs/wordpress/    # Arquivos WordPress (EFS)
```

### 📋 Comandos Úteis
```bash
# Verificar status do container
docker ps

# Ver logs do WordPress
docker logs wordpress

# Reiniciar aplicação
docker-compose down && docker-compose up -d

# Verificar montagem EFS
df -h | grep efs
```

---

<div align="center">

**📅 Última Atualização**: 28/09/2025  
**👨‍💻 Desenvolvido por**: Thiago Cardoso 
**🏷️ Versão**: 1.0.0  

---

![AWS](https://img.shields.io/badge/AWS-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![WordPress](https://img.shields.io/badge/WordPress-21759B?style=for-the-badge&logo=wordpress&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white)

</div>
