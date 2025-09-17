# Infraestrutura WordPress - AWS

## 📌 Etapa 1: Criação da VPC

- Criada uma **VPC dedicada** chamada `WordPress-VPC` com o bloco CIDR `10.0.0.0/16`
- Foram habilitadas:
  - 2 zonas de disponibilidade (us-east-1a e us-east-1b)
  - Subnets públicas e privadas
  - Internet Gateway para comunicação externa
  - Rotas configuradas automaticamente

### 🔖 Tags aplicadas

- `Project: WordPress`
- `CostCenter: valor`
- `Owner: Thiago`

## 📌 Etapa 2: Criação de Subnets

Foram criadas **4 subnets** dentro da VPC:

### **Públicas**:
- `WordPress-VPC-subnet-public1-us-east-1a` → `10.0.1.0/24`
- `WordPress-VPC-subnet-public2-us-east-1b` → `10.0.2.0/24`

### **Privadas**:
- `WordPress-VPC-subnet-private1-us-east-1a` → `10.0.3.0/24`
- `WordPress-VPC-subnet-private2-us-east-1b` → `10.0.4.0/24`

> 📌 As **subnets públicas** serão usadas pelo **Application Load Balancer (ALB)**  
> 📌 As **subnets privadas** serão usadas pelas **instâncias EC2 (WordPress)** e pelo **banco de dados (RDS)**

## 📌 Etapa 3: Criação do Banco de Dados (RDS)

- **Motor**: MySQL (versão `8.0.42`, tipo `db.t3g.micro`)
- **Modo de implantação**: Multi-AZ desabilitado (conforme restrição do projeto para controle de custos)
- **DB Subnet Group**: Criado (`wordpress-db-subnet-group`) e associado apenas às **subnets privadas**
- **Acesso**: Sem acesso público (`Public access: No`). A segurança é controlada via Security Group
- **Status**: **Parado (Stopped)** para economizar custos

## 📌 Etapa 4: Configuração dos Grupos de Segurança (Security Groups)

Foram criados 3 grupos de segurança para isolar e proteger as camadas da aplicação:

- **`ec2-sg-wordpress`**: Criado para ser associado às instâncias EC2
- **`SG-RDS-WordPress`**: Associado à instância RDS. Sua regra de entrada permite tráfego na porta `3306` (MySQL) **apenas** a partir do grupo `ec2-sg-wordpress`
- **`efs-sg-wordpress`**: Associado ao sistema de arquivos EFS. Sua regra de entrada permite tráfego na porta `2049` (NFS) **apenas** a partir do grupo `ec2-sg-wordpress`

## 📌 Etapa 5: Criação do Sistema de Arquivos (EFS)

- **File System**: Criado um EFS do tipo Regional (`wordpress-efs`) para armazenar os arquivos do WordPress
- **Pontos de Acesso (Mount Targets)**: Criados dentro das **subnets privadas** (um por Zona de Disponibilidade)
- **Segurança**: O acesso é controlado pelo grupo de segurança `efs-sg-wordpress`, garantindo que apenas as instâncias EC2 possam se conectar
- **Status**: **Criado e Disponível**

## ✅ Status Atual e Ações de Pausa

A infraestrutura de rede (VPC, Subnets), segurança (Security Groups) e dados (RDS, EFS) foi completamente provisionada.

Para economizar custos, os seguintes recursos foram **pausados/excluídos temporariamente**:
- A instância **RDS** foi **parada (Stopped)**
- Os **NAT Gateways** e seus **Elastic IPs** foram **excluídos**

## 🚀 Próximos Passos e Pendências

### 1. **Reativar o ambiente**:
- Recriar os NAT Gateways
- Atualizar as tabelas de rotas privadas
- Iniciar (Start) a instância RDS

### 2. **Criar o Launch Template**:
Configurar o modelo de lançamento para as instâncias EC2, inserindo o script `user-data` que prepara o ambiente WordPress

### 3. **Criar o Auto Scaling Group**:
Configurar o grupo que irá gerenciar e escalar as instâncias EC2 nas subnets privadas

### 4. **Criar o Application Load Balancer**:
Configurar o balanceador de carga nas subnets públicas para distribuir o tráfego para as instâncias
