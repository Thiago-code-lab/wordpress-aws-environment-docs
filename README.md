# 🖥️ Infraestrutura WordPress - AWS

📌 Etapa 1: Criação da VPC
Criada uma VPC dedicada chamada WordPress-VPC com o bloco CIDR 10.0.0.0/16.

Configurações habilitadas:

2 zonas de disponibilidade (us-east-1a e us-east-1b)

Subnets públicas e privadas

Internet Gateway para comunicação externa

Rotas e NAT Gateways para acesso à internet

🔖 Tags aplicadas
Project: WordPress

CostCenter: valor

Owner: Thiago

📌 Etapa 2: Criação de Subnets
Foram criadas 4 subnets dentro da VPC:

🌐 Subnets Públicas
WordPress-VPC-subnet-public1-us-east-1a → 10.0.1.0/24

WordPress-VPC-subnet-public2-us-east-1b → 10.0.2.0/24

🔒 Subnets Privadas
WordPress-VPC-subnet-private1-us-east-1a → 10.0.3.0/24

WordPress-VPC-subnet-private2-us-east-1b → 10.0.4.0/24

📌 Funções
Públicas → usadas pelo Application Load Balancer (ALB)

Privadas → usadas pelas instâncias EC2 (WordPress) e pelo banco de dados (RDS)

📌 Etapa 3: Criação do Banco de Dados (RDS)
Motor: MySQL (8.0.42, tipo db.t3g.micro)

Multi-AZ: desabilitado (restrição do projeto)

DB Subnet Group: wordpress-db-subnet-group, associado apenas às subnets privadas

Acesso: Sem acesso público (Public access: No), controlado via Security Group

Status: Iniciado (Available)

📌 Etapa 4: Configuração dos Security Groups
Criados 3 grupos de segurança para isolar camadas:

ec2-sg-wordpress → associado às instâncias EC2

SG-RDS-WordPress → associado ao RDS, permite tráfego na porta 3306 apenas do grupo ec2-sg-wordpress

efs-sg-wordpress → associado ao EFS, permite tráfego na porta 2049 (NFS) apenas do grupo ec2-sg-wordpress

📌 Etapa 5: Criação do Sistema de Arquivos (EFS)
File System: wordpress-efs (Regional)

Mount Targets: criados dentro das subnets privadas (um por AZ)

Segurança: acessível apenas pelo grupo ec2-sg-wordpress

Status: Criado e Disponível

📌 Etapa 6: Criação do Launch Template (Docker)
Mudança de Estratégia: A abordagem foi alterada para utilizar Docker e Docker Compose, seguindo novas diretrizes do projeto.

Criado o template wordpress-docker-lt para as instâncias EC2.

Configurações principais:

AMI: Ubuntu Server 22.04 LTS

Instance Type: t2.micro

Security Group: ec2-sg-wordpress

Tags: Project, CostCenter, Owner e Name aplicadas.

User Data: Script configurado para instalar Docker, Docker Compose, montar o EFS e iniciar o contêiner do WordPress conectado ao RDS.

✅ Status Atual e Pendências
A infraestrutura base (VPC, RDS, EFS, SGs) e o novo Launch Template com Docker foram criados com sucesso.

Uma instância EC2 de teste foi lançada, superando os desafios de permissão (SCP) anteriores.

Problema Atual: A instância, apesar de rodar o contêiner do WordPress, exibe o erro "Error establishing a database connection".

Diagnóstico: A investigação via SSH revelou que o problema é uma falha de sincronia entre a montagem do EFS no servidor e o serviço do Docker que não o enxergava corretamente. A última ação executada foi reiniciar o serviço do Docker e recriar o contêiner para forçar o reconhecimento do volume EFS.

🚀 Próximos Passos
Validação Final da Solução:

Ação Imediata: Acessar o IP Público da instância de teste no navegador para verificar se a última correção (reiniciar o Docker) resolveu o erro de conexão com o banco de dados.

Continuar o Projeto (Após a solução do problema):

Sucesso! Validar a página de instalação do WordPress.

Limpeza: Encerrar (Terminate) a instância de teste e remover as regras temporárias de HTTP/SSH do ec2-sg-wordpress.

Organizar Security Groups para Produção: Criar um novo SG para o Load Balancer (alb-sg-wordpress) e ajustar o ec2-sg-wordpress para aceitar tráfego apenas do alb-sg.

Criar o Auto Scaling Group usando o Launch Template wordpress-docker-launcher-template.

Criar o Application Load Balancer (ALB) para distribuir o tráfego.
