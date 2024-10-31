# Documentação da Infraestrutura WordPress na AWS

Documentação da Infraestrutura WordPress na AWS

Sumário

Introdução

Configuração da Rede (VPC e Subnets)

Configuração do Internet Gateway e NAT Gateway

Configuração de Tabelas de Roteamento (Route Tables)

Configuração de Security Groups

Configuração do Banco de Dados (Amazon RDS)

Configuração do Sistema de Arquivos (Amazon EFS)

Configuração do Load Balancer (Classic Load Balancer)

Criação do Launch Template para as Instâncias EC2

Configuração do Auto Scaling Group (ASG)

Script de Inicialização (User Data) para Configuração do WordPress

1. Introdução

Esta documentação descreve a configuração de uma infraestrutura AWS para hospedar uma aplicação WordPress de maneira escalável e resiliente. Ela inclui a utilização de serviços como VPC, Subnets, Security Groups, NAT Gateway, Internet Gateway, Amazon RDS, Amazon EFS, Load Balancer e uma instância EC2 com Docker para o WordPress.

2. Configuração da Rede (VPC e Subnets)

VPC

ID: vpc-06ac2f4684c29af08

CIDR: 10.0.0.0/16

Subnets

Public-Subnet-AZ1 (ID: subnet-0807e832f1532a687)

CIDR: 10.0.3.0/24

Associada à tabela de rota pública para acesso à Internet.

Public-Subnet-AZ2 (ID: subnet-028708829b9109889)

CIDR: 10.0.4.0/24

Associada à tabela de rota pública.

Private-Subnet-AZ1 (ID: subnet-01826e970b7f3399d)

CIDR: 10.0.1.0/24

Associada à tabela de rota privada para uso com o NAT Gateway.

Private-Subnet-AZ2 (ID: subnet-0499d6e3637bf4e3e)

CIDR: 10.0.2.0/24

Associada à tabela de rota privada.

3. Configuração do Internet Gateway e NAT Gateway

Internet Gateway

ID: igw-02f745eb2bab24b13

Associado à VPC para fornecer conectividade de Internet às subnets públicas.

NAT Gateway

ID: nat-08cd2a1361584021e

Localizado em Public-Subnet-AZ1 para permitir que instâncias em subnets privadas acessem a Internet para atualizações e dependências.

4. Configuração de Tabelas de Roteamento (Route Tables)

Public Route Table

ID: rtb-04c513ae89c759c12

Rotas:

0.0.0.0/0 direcionado ao Internet Gateway (igw-02f745eb2bab24b13)

10.0.0.0/16 para tráfego local.

Private Route Table

ID: rtb-0cbe68bf200070b14

Rotas:

0.0.0.0/0 direcionado ao NAT Gateway (nat-08cd2a1361584021e)

10.0.0.0/16 para tráfego local.

5. Configuração de Security Groups

EC2-WordPress-SG

Regras de entrada:

Porta 80: Acesso HTTP do Load Balancer.

Porta 443: Acesso HTTPS do Load Balancer.

Porta 22: Acesso SSH (opcional e controlado).

Porta 8080: Configuração customizada.

Porta 11211: Acesso para Memcached.

Regras de saída: Todos os destinos permitidos.

WordPress-RDS-SG

Porta 3306: Acesso MySQL para o WordPress.

Fonte: EC2-WordPress-SG.

WordPress-EFS-SG

Porta 2049: Acesso NFS para EFS.

Fonte: EC2-WordPress-SG.

CLB-WordPress-SG

Porta 80 e 443: Acesso HTTP/HTTPS da Internet.

6. Configuração do Banco de Dados (Amazon RDS)

DB Instance Identifier: db-wordpress

Engine: MySQL

VPC Security Groups: WordPress-RDS-SG

Endpoint: Gerenciado para conexões seguras do WordPress.

7. Configuração do Sistema de Arquivos (Amazon EFS)

File System ID: fs-0554a7102bb9c22b9

Mount Targets:

us-east-1a: 10.0.1.245

us-east-1b: 10.0.2.197

8. Configuração do Load Balancer (Classic Load Balancer)

Tipo: Classic Load Balancer

Listeners:

HTTP 80 para redirecionamento de tráfego da Internet.

HTTPS 443 para tráfego seguro.

Distribuição de tráfego: Entre instâncias nas zonas us-east-1a e us-east-1b.

9. Criação do Launch Template para as Instâncias EC2

AMI ID: ami-0ddc798b3f1a5117e

Tipo de instância: t2.micro

Security Group: EC2-WordPress-SG

Configuração: Docker e Docker Compose instalados para rodar o WordPress.

10. Configuração do Auto Scaling Group (ASG)

Auto Scaling Group Name: WordPress-ASG

Launch Template: WordPress-LaunchTemplate (Versão 13)

Subnets: Associado às subnets públicas para que as instâncias estejam acessíveis via Internet.

Políticas de Escala:

Scale Out: Aumenta a capacidade quando a utilização de CPU excede 75%.

Scale In: Reduz a capacidade quando a utilização de CPU cai abaixo de 30%.

11. Script de Inicialização (User Data) para Configuração do WordPress

Explicação do Script

Explicação Resumida do Script

Configuração do Ambiente: Atualiza o sistema e instala Docker, Docker Compose e as ferramentas necessárias para EFS.

Autenticação e Credenciais: Recupera as credenciais do banco de dados do Secrets Manager da AWS para configurar o WordPress com acesso ao RDS.

Montagem do EFS: Detecta a zona de disponibilidade e monta o EFS no local correto para persistência de dados.

Docker Compose: Configura e inicia o WordPress usando Docker Compose, com volumes persistentes mapeados no EFS.

Health Check e Configuração Final: Cria um arquivo de saúde para o Load Balancer e ajusta o wp-config.php do WordPress com as credenciais do banco de dados.

- Esse script cobre todos os passos essenciais para configurar o WordPress em Docker com suporte a RDS e EFS.
