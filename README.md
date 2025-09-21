# 🖥️ Infraestrutura WordPress - AWS

## 📌 Etapa 1: Criação da VPC
Criada uma **VPC dedicada** chamada `WordPress-VPC` com o bloco CIDR `10.0.0.0/16`.

**Configurações habilitadas:**
- 2 zonas de disponibilidade (`us-east-1a` e `us-east-1b`)
- Subnets públicas e privadas
- Internet Gateway para comunicação externa
- Rotas e NAT Gateways para acesso à internet

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

### 📌 Funções
- **Públicas** → usadas pelo **Application Load Balancer (ALB)**
- **Privadas** → usadas pelas **instâncias EC2 (WordPress)** e pelo **banco de dados (RDS)**

---

## 📌 Etapa 3: Criação do Banco de Dados (RDS)
**Especificações:**
- Motor: **MySQL 8.0.42** (`db.t3g.micro`)
- Multi-AZ: **desabilitado** (restrição do projeto)
- DB Subnet Group: `wordpress-db-subnet-group`, associado apenas às subnets privadas
- Acesso: **Sem acesso público** (`Public access: No`), controlado via **Security Group**
- Status: **Iniciado (Available)**

---

## 📌 Etapa 4: Configuração dos Security Groups
Criados **4 grupos de segurança** para isolar as camadas:

- `alb-sg-wordpress` → associado ao ALB (HTTP/HTTPS público)
- `ec2-sg-wordpress` → associado às instâncias EC2
- `SG-RDS-WordPress` → associado ao RDS (porta 3306 apenas para `ec2-sg-wordpress`)
- `efs-sg-wordpress` → associado ao EFS (porta 2049 apenas para `ec2-sg-wordpress`)

---

## 📌 Etapa 5: Criação do Sistema de Arquivos (EFS)
**Configuração:**
- File System: `wordpress-efs` (Regional)
- Mount Targets: criados em **subnets privadas** (um por AZ)
- Segurança: acessível apenas pelo `ec2-sg-wordpress`
- Status: **Criado e Disponível**

---

## 📌 Etapa 6: Criação do Launch Template (Docker)
**Mudança de Estratégia:**  
A instalação tradicional foi substituída por uma abordagem moderna com **Docker** e **Docker Compose**.

**Launch Template:**
- Nome: `wordpress-docker-lt`
- AMI: **Ubuntu Server 22.04 LTS**
- Instance Type: `t2.micro`
- Security Group: `ec2-sg-wordpress`
- Tags: `Project`, `CostCenter`, `Owner`, `Name`
- User Data: Script para instalar **Docker**, **Docker Compose**, montar o **EFS** e iniciar o contêiner **WordPress** conectado ao **RDS**

---

## ✅ Status Atual
- Infraestrutura de rede (VPC, Subnets, NATs, IGW, RTs) criada e validada
- Security Groups configurados em camadas
- Banco de Dados (RDS MySQL) disponível
- EFS provisionado e com mount targets
- Launch Template com Docker configurado

---

## ⚠️ Desafio Atual
Na **instância EC2 de teste** criada a partir do Launch Template:

- O script **user-data falhou** durante a inicialização
- **Diagnóstico:** apesar da instância estar em uma subnet pública, **não recebeu IP Público**
- Sem IP Público, não conseguiu baixar pacotes da internet (ex: Docker), quebrando a automação
- Resultado: erro ao acessar via navegador, impossibilitando validar a instalação do WordPress

---

## 🚀 Próximos Passos
1. **Corrigir atribuição de IP Público:**
   - Garantir que a **subnet pública** esteja com *Auto-assign public IPv4 = Enabled*
   - No lançamento da instância de teste, confirmar a opção *Auto-assign Public IP: Enable*

2. **Reexecutar o teste:**
   - Relançar a instância de teste do template corrigido
   - Confirmar instalação automática do WordPress via navegador

3. **Validação Final:**
   - Se WordPress carregar, encerrar instância de teste
   - Remover regras temporárias (HTTP/SSH) do SG `ec2-sg-wordpress`

4. **Produção:**
   - Criar **Auto Scaling Group** com o template
   - Criar **Application Load Balancer** associado às subnets públicas
   - Redirecionar tráfego do ALB para instâncias privadas no ASG
   - Ajustar SGs (ALB → EC2; EC2 → RDS/EFS)

---

## 📌 Observação Importante
Este erro de **falta de IP Público** foi a causa principal da falha no user-data.  
A correção desse ponto é fundamental para permitir a instalação automática do Docker, EFS e WordPress.

