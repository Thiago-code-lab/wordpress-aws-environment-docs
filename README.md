# 🖥️ Infraestrutura WordPress - AWS

## 📋 Visão Geral

Este documento descreve a implementação de uma infraestrutura completa para WordPress na AWS, utilizando boas práticas de segurança, alta disponibilidade e escalabilidade automática.

---

## 🏗️ Progresso da Implementação

### 📌 Etapa 1: Criação da VPC
**Status:** ✅ **Concluído**

Foi criada uma **VPC dedicada** chamada `WordPress-VPC` com o bloco CIDR `10.0.0.0/16` através do assistente "VPC and more".

**Recursos Provisionados:**
- 2 Zonas de Disponibilidade (`us-east-1a` e `us-east-1b`)
- 4 Subnets (2 públicas e 2 privadas)
- 1 Internet Gateway para acesso público
- 2 NAT Gateways (um por AZ) para acesso à internet pelas subnets privadas
- Tabelas de Rota configuradas e associadas automaticamente

---

### 📌 Etapa 2: Security Groups
**Status:** ✅ **Concluído**

Foram criados **4 grupos de segurança** para implementar uma arquitetura de rede segura em camadas:

| Security Group | Função | Regras de Entrada |
|---|---|---|
| `alb-sg-wordpress` | Load Balancer | HTTP (80) e HTTPS (443) - `0.0.0.0/0` |
| `ec2-sg-wordpress` | Instâncias EC2 | HTTP (80) - apenas de `alb-sg-wordpress` |
| `rds-sg-wordpress` | Banco de dados | MySQL (3306) - apenas de `ec2-sg-wordpress` |
| `efs-sg-wordpress` | Sistema de arquivos | NFS (2049) - apenas de `ec2-sg-wordpress` |

---

### 📌 Etapa 3: Sistema de Arquivos (EFS)
**Status:** ✅ **Concluído**

Foi criado um **File System** (`wordpress-efs`) do tipo Regional.

**Configuração:**
- Mount Targets criados nas **subnets privadas**
- Segurança garantida pela associação com o `efs-sg-wordpress`

---

### 📌 Etapa 4: Banco de Dados (RDS)
**Status:** ✅ **Concluído**

Foi criada uma instância **RDS MySQL** (`wordpress-db`) do tipo `db.t3.micro`.

**Configuração:**
- Instância isolada nas subnets privadas através de **DB Subnet Group**
- Protegida pelo `rds-sg-wordpress`
- Configurada sem acesso público
- Banco de dados inicial `wordpress` criado

---

### 📌 Etapa 5: Launch Template (Docker)
**Status:** ✅ **Concluído**

Foi criado um **Launch Template** (`wordpress-docker-lt`) baseado em **Ubuntu 22.04**.

**Automação:**
- Script `user-data` completo que:
  - Instala e configura Docker e Docker Compose
  - Monta o EFS
  - Inicia o contêiner do WordPress conectado ao RDS e EFS

---

### 📌 Etapa 6: Target Group e Load Balancer
**Status:** ✅ **Concluído**

Foram criados os componentes para balanceamento de carga:

**Recursos:**
- **Target Group** (`wordpress-tg`): Agrupa as instâncias do WordPress
- **Application Load Balancer** (`wordpress-alb`):
  - Configurado como `Internet-facing`
  - Posicionado nas subnets públicas
  - Protegido pelo `alb-sg-wordpress`
  - Listener configurado para encaminhar tráfego para o `wordpress-tg`

---

## ✅ Status Atual e Pendências

### **Progresso Atual**
Quase toda a infraestrutura foi provisionada com sucesso, incluindo:
- ✅ Rede e segurança
- ✅ Camada de dados
- ✅ Template de automação
- ✅ Balanceamento de carga

### **⚠️ Desafio Atual**
O último passo, a criação do **Auto Scaling Group**, não pôde ser concluído.

**Erro Encontrado:**
```
You are not authorized to perform this operation
```

**Causa Provável:**
Política de segurança da conta (SCP) que impede o serviço de Auto Scaling de utilizar o Launch Template ou seus componentes.

---

## 🚀 Próximos Passos

### 1. Diagnosticar e Criar o Auto Scaling Group
- [ ] Tentar criar o ASG novamente e analisar mensagem de erro específica
- [ ] Investigar permissões ausentes (ex: `iam:PassRole`)
- [ ] Verificar restrições em recursos que o ASG tenta criar (ex: volumes EBS)

### 2. Validação Final
- [ ] Verificar se o ASG lança 2 instâncias
- [ ] Confirmar se as instâncias ficam "healthy" no Target Group
- [ ] Acessar o DNS do Load Balancer para confirmar carregamento do WordPress

---

## ⏸️ Ações para Pausar Custos

Para evitar cobranças desnecessárias, os seguintes recursos devem ser gerenciados:

| Recurso | Ação | Motivo |
|---|---|---|
| Application Load Balancer | `Delete` (Excluir) | Cobrança por hora |
| Instância RDS | `Stop` (Parar) | Reduzir custos computacionais |
| NAT Gateways | `Delete` (Excluir) | Cobrança por hora + tráfego |
| Elastic IPs | `Release` (Liberar) | Cobrança por IP não utilizado |

---

## 📚 Recursos Criados

### Componentes de Rede
- VPC: `WordPress-VPC` (10.0.0.0/16)
- Subnets: 2 públicas + 2 privadas
- Internet Gateway + 2 NAT Gateways

### Componentes de Segurança
- 4 Security Groups com regras em camadas
- Isolamento de tráfego por função

### Componentes de Dados
- RDS MySQL: `wordpress-db` (db.t3.micro)
- EFS: `wordpress-efs` (Regional)

### Componentes de Computação
- Launch Template: `wordpress-docker-lt` (Ubuntu 22.04 + Docker)
- Target Group: `wordpress-tg`
- Application Load Balancer: `wordpress-alb`

---

**📅 Última Atualização:** [21/09/2025]  
**🔧 Responsável:** [Thiago Cardoso]
