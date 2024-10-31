
# Projeto Docker Wordpress Compass

Esta documentação descreve a configuração de uma infraestrutura AWS para hospedar uma aplicação WordPress de maneira escalável e resiliente. Ela inclui a utilização de serviços como VPC, Subnets, Security Groups, NAT Gateway, Internet Gateway, Amazon RDS, Amazon EFS, Load Balancer e uma instância EC2 com Docker para o WordPress.

![Imagem 1](imagem/imagen1.png)

![Imagem 2](imagem/imagen2.png)


# Sumário
1.	Introdução
2.	Configuração da Rede (VPC e Subnets)
3.	Configuração do Internet Gateway e NAT Gateway
4.	Configuração de Tabelas de Roteamento (Route Tables)
5.	Configuração de Security Groups
6.	Configuração do Banco de Dados (Amazon RDS)
7.	Configuração do Sistema de Arquivos (Amazon EFS)
8.	Configuração do Load Balancer (Classic Load Balancer)
9.	Criação do Launch Template para as Instâncias EC2
10.	Configuração do Auto Scaling Group (ASG)
11.	Script de Inicialização (User Data) para Configuração do WordPress





# 1. Introdução
Esta documentação descreve a configuração de uma infraestrutura AWS para hospedar uma aplicação WordPress de maneira escalável e resiliente. Ela inclui a utilização de serviços como VPC, Subnets, Security Groups, NAT Gateway, Internet Gateway, Amazon RDS, Amazon EFS, Load Balancer e uma instância EC2 com Docker para o WordPress.

#2. Configuração da Rede (VPC e Subnets)

•	VPC ID: vpc-06ac2f4684c29af08
•	CIDR: 10.0.0.0/16

# 2. Subnets

•	Public-Subnet-AZ1 (ID: subnet-0807e832f1532a687)
  o	CIDR: 10.0.3.0/24
  o	Associada à tabela de rota pública para acesso à Internet.

•	Public-Subnet-AZ2 (ID: subnet-028708829b9109889)
  o	CIDR: 10.0.4.0/24
  o	Associada à tabela de rota pública.

•	Private-Subnet-AZ1 (ID: subnet-01826e970b7f3399d)
  o	CIDR: 10.0.1.0/24
  o	Associada à tabela de rota privada para uso com o NAT Gateway.

•	Private-Subnet-AZ2 (ID: subnet-0499d6e3637bf4e3e)
  o	CIDR: 10.0.2.0/24
  o	Associada à tabela de rota privada.

# 3. Configuração do Internet Gateway e NAT Gateway
  • Internet GatewayID: igw-02f745eb2bab24b13
  •Associado à VPC para fornecer conectividade de Internet às subnets públicas.
  • NAT Gateway	ID: nat-08cd2a1361584021e
  •	Localizado em Public-Subnet-AZ1 para permitir que instâncias em subnets privadas acessem a Internet para atualizações e dependências.
  
# 4. Configuração de Tabelas de Roteamento (Route Tables)

  • Public Route Table	ID: rtb-04c513ae89c759c12
  
  Rotas:
  o	0.0.0.0/0 direcionado ao Internet Gateway (igw-02f745eb2bab24b13)
  o	10.0.0.0/16 para tráfego local.

 • Private Route Table ID: rtb-0cbe68bf200070b14
  Rotas:
  o	0.0.0.0/0 direcionado ao NAT Gateway (nat-08cd2a1361584021e)
  o	10.0.0.0/16 para tráfego local.
  
# 5. Configuração de Security Groups

## EC2-WordPress-SG
•	Regras de entrada:
  o	Porta 80: Acesso HTTP do Load Balancer.
  o	Porta 443: Acesso HTTPS do Load Balancer.
  o	Porta 22: Acesso SSH (opcional e controlado).
  o	Porta 8080: Configuração customizada.
  o	Porta 11211: Acesso para Memcached.
  •	Regras de saída: Todos os destinos permitidos.
  
## WordPress-RDS-SG
  •	Porta 3306: Acesso MySQL para o WordPress.
  •	Permissão: EC2-WordPress-SG.
  
## WordPress-EFS-SG
  •	Porta 2049: Acesso NFS para EFS.
  •	Permissão: EC2-WordPress-SG.
  
## CLB-WordPress-SG
  •	Porta 80 e 443: Acesso HTTP/HTTPS da Internet.
  
# 6. Configuração do Banco de Dados (Amazon RDS)
  •	DB Instance Identifier: db-wordpress
  •	Engine: MySQL
  •	VPC Security Groups: WordPress-RDS-SG
  •	Endpoint: Gerenciado para conexões seguras do WordPress.

# 7. Configuração do Sistema de Arquivos (Amazon EFS)
  •	File System ID: fs-0554a7102bb9c22b9
•	Mount Targets:
  o	us-east-1a: 10.0.1.245
  o	us-east-1b: 10.0.2.197
  
# 8. Configuração do Load Balancer (Classic Load Balancer)
  •	Tipo: Classic Load Balancer
  •	Listeners:
    o	HTTP 80 para redirecionamento de tráfego da Internet.
    o	HTTPS 443 para tráfego seguro.
    •	Distribuição de tráfego: Entre instâncias nas zonas us-east-1a e us-east-1b.
    
# 9. Criação do Launch Template para as Instâncias EC2
    •	AMI ID: ami-0ddc798b3f1a5117e
    •	Tipo de instância: t2.micro
    •	Security Group: EC2-WordPress-SG
    •	Configuração: Docker e Docker Compose instalados para rodar o WordPress.
    
# 10. Configuração do Auto Scaling Group (ASG)
  •	Auto Scaling Group Name: WordPress-ASG
  •	Launch Template: WordPress-LaunchTemplate (Versão 13)
  •	Subnets: Associado às subnets públicas para que as instâncias estejam acessíveis via Internet.
  •	Políticas de Escala:
    o	Scale Out: Aumenta a capacidade quando a utilização de CPU excede 75%.
    o	Scale In: Reduz a capacidade quando a utilização de CPU cai abaixo de 30%.
    
# 11. Script de Inicialização (User Data) para Configuração do WordPress

## Explicação do Script

•	Configuração do Ambiente: Atualiza o sistema e instala Docker, Docker Compose e as ferramentas necessárias para EFS.
•	Autenticação e Credenciais: Recupera as credenciais do banco de dados do Secrets Manager da AWS para configurar o WordPress com acesso ao RDS.
•	Montagem do EFS: Detecta a zona de disponibilidade e monta o EFS no local correto para persistência de dados.
•	Docker Compose: Configura e inicia o WordPress usando Docker Compose, com volumes persistentes mapeados no EFS.
•	Health Check e Configuração Final: Cria um arquivo de saúde para o Load Balancer e ajusta o wp-config.php do WordPress com as credenciais do banco de dados.


![Imagem 3](imagem/imagen3.png)

![Imagem 4](imagem/imagen4.png)

![Imagem 5](imagem/imagen5.png)

![Imagem 6](imagem/imagen6.png)
