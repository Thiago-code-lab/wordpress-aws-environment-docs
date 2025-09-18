# 🖥️ Infraestrutura WordPress - AWS

## 📌 Etapa 1: Criação da VPC

Criada uma **VPC dedicada** chamada `WordPress-VPC` com o bloco CIDR `10.0.0.0/16`.

**Configurações habilitadas:**
- 2 zonas de disponibilidade (`us-east-1a` e `us-east-1b`)
- Subnets públicas e privadas
- Internet Gateway para comunicação externa
- Rotas configuradas automaticamente

### 🔖 Tags aplicadas
- `Project: WordPress`
- `CostCenter: valor`
- `Owner: Thiago`

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

## 📌 Etapa 3: Criação do Banco de Dados (RDS)

**Especificações técnicas:**
- **Motor**: MySQL (`8.0.42`, tipo `db.t3g.micro`)
- **Multi-AZ**: desabilitado (restrição do projeto para controle de custos)
- **DB Subnet Group**: `wordpress-db-subnet-group`, associado apenas às subnets privadas
- **Acesso**: Sem acesso público (`Public access: No`), controlado via **Security Group**
- **Status**: **Iniciado (Running)** para a fase de testes

## 📌 Etapa 4: Configuração dos Security Groups

Criados **3 grupos de segurança** para isolar camadas:

- **`ec2-sg-wordpress`** → associado às instâncias EC2
- **`SG-RDS-WordPress`** → associado ao RDS, permite tráfego na porta `3306` **apenas do grupo** `ec2-sg-wordpress`
- **`efs-sg-wordpress`** → associado ao EFS, permite tráfego na porta `2049` (NFS) **apenas do grupo** `ec2-sg-wordpress`

## 📌 Etapa 5: Criação do Sistema de Arquivos (EFS)

**Configuração do EFS:**
- **File System**: `wordpress-efs` (Regional)
- **Mount Targets**: criados dentro das subnets privadas (um por AZ)
- **Segurança**: acessível apenas pelo grupo `ec2-sg-wordpress`
- **Status**: **Criado e Disponível**

## 📌 Etapa 6: Criação do Launch Template

Criado o template `wordpress-launch-template-v2` para definir a configuração das instâncias EC2.

### **Configurações principais:**
- **AMI**: Amazon Linux 2
- **Instance Type**: `t2.micro`
- **Security Group**: `ec2-sg-wordpress`
- **Tags**: `Project`, `CostCenter`, `Owner` e `Name` aplicadas na instância e volume
- **User Data**: Script configurado para instalar Apache/PHP, montar o EFS e conectar ao RDS automaticamente

## ✅ Status Atual e Pendências

**Progresso completado:**
- Toda a infraestrutura base (VPC, RDS, EFS, SGs) e o Launch Template foram criados
- Uma **instância EC2 de teste** foi lançada com sucesso a partir do template em uma subnet pública para validação

**⚠️ Problema Atual:**
Ao tentar acessar o IP Público da instância de teste, ocorre um erro de **timeout** (`ERR_CONNECTION_TIMED_OUT`), impedindo a validação da instalação do WordPress.

## 🚀 Próximos Passos

### 1. **Reativar o Ambiente**

Antes de continuar o troubleshooting, é preciso reativar os serviços que foram pausados para economizar custos.

**Iniciar a Instância RDS:**
- Vá ao console do RDS e dê "Start" na instância `wordpress-db`
- Aguarde até que o status seja "Available"

**Recriar os NAT Gateways:**
- Crie um novo NAT Gateway em cada uma das suas **subnets públicas**
- Lembre-se de alocar um novo Elastic IP para cada um

**Atualizar as Tabelas de Rotas:**
- Vá nas tabelas de rotas das suas **subnets privadas**
- Atualize a rota `0.0.0.0/0` para apontar para os novos NAT Gateways correspondentes

### 2. **Resolver o Problema de Conexão (Troubleshooting)**

Com o ambiente ativo, o próximo passo é recriar o cenário do teste para investigar o erro de timeout.

**Lançar a Instância de Teste:**
- Lance uma nova instância a partir do `wordpress-launch-template-v2` em uma **subnet pública**
- Garanta que ela receba um IP Público

**Configurar Acesso Temporário:**
- Adicione as regras `HTTP` e `SSH` (com origem `My IP`) ao security group `ec2-sg-wordpress`

**Investigação:**
- **Ação Imediata**: Conectar na instância de teste via **SSH**
- Verificar o status do serviço Apache (`httpd`) com o comando `systemctl status httpd`
- Analisar os logs de inicialização (`/var/log/cloud-init-output.log`) para encontrar possíveis erros na execução do script `user-data`

### 3. **Continuar o Projeto (Após a solução do problema)**

- Validar o acesso e a instalação do WordPress pelo navegador
- Terminar a instância de teste e remover as regras temporárias do Security Group
- Criar o **Auto Scaling Group**
- Criar o **Application Load Balancer**
