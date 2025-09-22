# 🖥️ Infraestrutura WordPress - AWS

## 📋 Visão Geral

Este documento descreve a implementação de uma infraestrutura completa para WordPress na AWS, utilizando boas práticas de segurança, alta disponibilidade e escalabilidade automática.

---

## 🏗️ Progresso da Implementação

### 📌 Etapa 1: Criação da VPC
**Status:** ✅ **Concluído**

Foi criada uma VPC dedicada chamada **WordPress-VPC** com o bloco CIDR `10.0.0.0/16` através do assistente "VPC and more".

**Recursos Provisionados:**
- 2 Zonas de Disponibilidade (`us-east-1a` e `us-east-1b`)
- 4 Subnets (2 públicas e 2 privadas)
- 1 Internet Gateway para acesso público
- 2 NAT Gateways (um por AZ) para acesso à internet pelas subnets privadas
- Tabelas de Rota configuradas e associadas automaticamente

---

### 📌 Etapa 2: Security Groups
**Status:** ✅ **Concluído**

Foram criados 4 grupos de segurança para implementar uma arquitetura de rede segura em camadas:

| Security Group | Função | Regras de Entrada |
|---|---|---|
| `alb-sg-wordpress` | Load Balancer | HTTP (80) e HTTPS (443) - `0.0.0.0/0` |
| `ec2-sg-wordpress` | Instâncias EC2 | HTTP (80) - apenas de `alb-sg-wordpress` |
| `rds-sg-wordpress` | Banco de dados | MySQL (3306) - apenas de `ec2-sg-wordpress` |
| `efs-sg-wordpress` | Sistema de arquivos | NFS (2049) - apenas de `ec2-sg-wordpress` |

---

### 📌 Etapa 3: Sistema de Arquivos (EFS)
**Status:** ✅ **Concluído**

Foi criado um File System (`wordpress-efs`) do tipo Regional.

**Configuração:**
- Mount Targets criados nas subnets privadas
- Segurança garantida pela associação com o `efs-sg-wordpress`

---

### 📌 Etapa 4: Banco de Dados (RDS)
**Status:** ✅ **Concluído**

Foi criada uma instância RDS MySQL (`wordpress-db`) do tipo `db.t3.micro`.

**Configuração:**
- Instância isolada nas subnets privadas através de DB Subnet Group
- Protegida pelo `rds-sg-wordpress`
- Configurada sem acesso público
- Banco de dados inicial `wordpress` criado

---

### 📌 Etapa 5: Launch Template (Docker)
**Status:** ✅ **Concluído**

Foi criado um Launch Template (`wordpress-docker-lt`) baseado em Ubuntu 22.04.

**Automação:**
Script user-data completo que:
- Instala e configura Docker e Docker Compose
- Monta o EFS
- Inicia o contêiner do WordPress conectado ao RDS e EFS

#### 📜 Script User Data

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
      WORDPRESS_DB_HOST: <SEU_ENDPOINT_RDS>
      WORDPRESS_DB_USER: admin
      WORDPRESS_DB_PASSWORD: ***
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - ${EFS_MOUNT_POINT}:/var/www/html
EOF

# Concede permissões ao usuário ubuntu
chown -R ubuntu:ubuntu ${EFS_MOUNT_POINT}
chown ubuntu:ubuntu /home/ubuntu/docker-compose.yml

# Executa o Docker Compose
/usr/local/bin/docker-compose -f /home/ubuntu/docker-compose.yml up -d
```

---

### 📌 Etapa 6: Target Group, Load Balancer e Auto Scaling
**Status:** ✅ **Concluído**

Foram criados os componentes finais para alta disponibilidade e escalabilidade:

- **Target Group** (`wordpress-tg`): Criado para agrupar as instâncias
- **Application Load Balancer** (`wordpress-alb`): Criado nas subnets públicas para distribuir o tráfego
- **Auto Scaling Group** (`wordpress-asg`): Criado para gerenciar as instâncias nas subnets privadas, com uma capacidade desejada de 2 instâncias e política de escalonamento baseada em CPU

---

## ✅ Status do Projeto: 90% Concluído

Toda a infraestrutura foi provisionada e validada com sucesso. O site WordPress está online, acessível através do Application Load Balancer, com as instâncias sendo gerenciadas por um Auto Scaling Group em um ambiente resiliente e distribuído em duas Zonas de Disponibilidade.

---

## 🚀 Próximos Passos (Opcional)

- [ ] Configurar HTTPS no ALB com um certificado do AWS Certificate Manager (ACM)
- [ ] Configurar um domínio personalizado (ex: via Route 53) para apontar para o DNS do ALB
- [ ] Implementar monitoramento contínuo da aplicação via CloudWatch

---

## ⏸️ Ações para Pausar Custos

Para evitar cobranças, os seguintes recursos devem ser gerenciados:

| Recurso | Ação | Motivo |
|---|---|---|
| Auto Scaling Group | Editar e definir Min/Desired/Max para 0 | Encerra as instâncias EC2 |
| Application Load Balancer | Delete (Excluir) | Cobrança por hora |
| Instância RDS | Stop (Parar) | Reduzir custos computacionais |
| NAT Gateways | Delete (Excluir) | Cobrança por hora + tráfego |
| Elastic IPs | Release (Liberar) | Cobrança por IP não utilizado |

---

**📅 Última Atualização:** 22/09/2025
