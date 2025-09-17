# Projeto AWS WordPress - Documentação do Ambiente

## 📌 Etapa 1: Criação da VPC
- Criada uma **VPC dedicada** chamada `WordPress-VPC` com o bloco CIDR `10.0.0.0/16`.
- Foram habilitadas:
  - 2 zonas de disponibilidade (us-east-1a e us-east-1b).
  - Subnets públicas e privadas.
  - Internet Gateway para comunicação externa.
  - Rotas configuradas automaticamente.

### 🔖 Tags aplicadas
- `Project: WordPress`
- `CostCenter: valor`
- `Owner: Thiago`

---

## 📌 Etapa 2: Criação de Subnets
- Foram criadas **4 subnets** dentro da VPC:
  - **Públicas**:
    - `WordPress-VPC-subnet-public1-us-east-1a` → `10.0.1.0/24`
    - `WordPress-VPC-subnet-public2-us-east-1b` → `10.0.2.0/24`
  - **Privadas**:
    - `WordPress-VPC-subnet-private1-us-east-1a` → `10.0.3.0/24`
    - `WordPress-VPC-subnet-private2-us-east-1b` → `10.0.4.0/24`

📌 As **subnets públicas** serão usadas pelo EC2 (WordPress).  
📌 As **subnets privadas** serão usadas pelo RDS (MySQL).

---

## 📌 Etapa 3: Preparação do Banco de Dados (RDS)
- Motor escolhido: **MySQL**.
- Versão: `MySQL 8.0.42`.
- Modo de implantação: **Multi-AZ DB instance deployment (2 instances)** para alta disponibilidade.
- Criado **DB Subnet Group dedicado**:
  - Nome: `wordpress-db-subnet-group`.
  - Inclui apenas **subnets privadas** (`10.0.3.0/24` e `10.0.4.0/24`).
- Configuração de acesso:
  - **Public access**: `No` (banco não acessível diretamente da internet).
  - VPC: `WordPress-VPC`.
  - Segurança: acesso será liberado somente para a instância EC2 do WordPress.

---

## 📌 Etapa 4: Considerações de Segurança
- Mantida a `default` DB Subnet Group (não alterada).
- Criado um grupo de sub-rede separado para evitar exposição do RDS.
- Banco ficará isolado em subnets privadas, aumentando a segurança.

---

## ✅ Status Atual
- VPC criada com sucesso.  
- Subnets públicas e privadas configuradas.  
- Tags aplicadas para identificação do projeto.  
- Preparação do RDS (MySQL) documentada e em andamento.  
- Próximo passo: **lançar o RDS na subnet group privada** e configurar as **Security Groups** para liberar comunicação somente com o EC2 (WordPress).  
