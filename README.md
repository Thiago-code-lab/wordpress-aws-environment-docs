# 🖥️ Infraestrutura WordPress - AWS

## 📌 Etapa 1: Criação da VPC
- Criada uma **VPC dedicada** chamada `WordPress-VPC` com o bloco CIDR `10.0.0.0/16`.
- Foram habilitadas:
  - 2 zonas de disponibilidade (`us-east-1a` e `us-east-1b`)
  - Subnets públicas e privadas
  - Internet Gateway para comunicação externa
  - Rotas configuradas automaticamente

### 🔖 Tags aplicadas
- `Project: WordPress`
- `CostCenter: valor`
- `Owner: Thiago`

---

## 📌 Etapa 2: Criação de Subnets
Foram criadas **4 subnets** dentro da VPC:

### 🌐 Subnets Públicas
- `WordPress-VPC-subnet-public1-us-east-1a` → `10.0.1.0/24`
- `WordPress-VPC-subnet-public2-us-east-1b` → `10.0.2.0/24`

### 🔒 Subnets Privadas
- `WordPress-VPC-subnet-private1-us-east-1a` → `10.0.3.0/24`
- `WordPress-VPC-subnet-private2-us-east-1b` → `10.0.4.0/24`

> 📌 **Funções**:
> - **Públicas** → usadas pelo **Application Load Balancer (ALB)**  
> - **Privadas** → usadas pelas **instâncias EC2 (WordPress)** e pelo **banco de dados (RDS)**

---

## 📌 Etapa 3: Criação do Banco de Dados (RDS)
- **Motor**: MySQL (`8.0.42`, tipo `db.t3g.micro`)
- **Multi-AZ**: desabilitado (restrição do projeto para controle de custos)
- **DB Subnet Group**: `wordpress-db-subnet-group`, associado apenas às subnets privadas
- **Acesso**: Sem acesso público (`Public access: No`), controlado via **Security Group**
- **Status**: **Parado (Stopped)** para economizar custos

---

## 📌 Etapa 4: Configuração dos Security Groups
Criados 3 **grupos de segurança** para isolar camadas:

- **`ec2-sg-wordpress`** → associado às instâncias EC2
- **`SG-RDS-WordPress`** → associado ao RDS, permite tráfego na porta `3306` **apenas do grupo `ec2-sg-wordpress`**
- **`efs-sg-wordpress`** → associado ao EFS, permite tráfego na porta `2049` (NFS) **apenas do grupo `ec2-sg-wordpress`**

---

## 📌 Etapa 5: Criação do Sistema de Arquivos (EFS)
- **File System**: `wordpress-efs` (Regional)
- **Mount Targets**: criados dentro das subnets privadas (um por AZ)
- **Segurança**: acessível apenas pelo grupo `ec2-sg-wordpress`
- **Status**: **Criado e Disponível**

---

## ✅ Status Atual e Ações de Pausa
Infraestrutura de **rede, segurança e dados** provisionada com sucesso.  
Para **economizar custos**, foram aplicadas as seguintes ações:

- A instância **RDS** está em **Stopped**
- Os **NAT Gateways** e seus **Elastic IPs** foram **excluídos**

---

## 🚀 Próximos Passos
1. **Reativar o ambiente**:
   - Recriar NAT Gateways
   - Atualizar rotas privadas
   - Start na instância RDS

2. **Criar o Launch Template**:
   - Inserir `user-data` para preparar ambiente WordPress nas EC2

3. **Criar o Auto Scaling Group**:
   - Distribuir instâncias EC2 em subnets privadas
   - Escalabilidade baseada em métricas de CPU

4. **Criar o Application Load Balancer**:
   - Publicar tráfego HTTP/HTTPS nas subnets públicas
   - Encaminhar requisições para as instâncias privadas
