# Guia de Implantação do WordPress com Docker na AWS

## 1. Criar VPC com 2 Subnets em AZs diferentes

### Criação da VPC
- Nome da VPC: docker-project-vpc
- Navegue até VPC > Your VPC > Create VPC
- IPv4 CIDR: 192.168.0.0/24
- Habilitar resoluções de DNS automáticas

### Criação das Subnets

#### Subnet na zona A
- Navegue até VPC > Subnets > Create Subnet
- Selecione sua VPC (docker-project-vpc)
- Nome da subnet: sub-az-a
- Availability Zone: us-east-1a
- IPv4 CIDR block: 192.168.0.0/27

#### Subnet na zona B
- Navegue até VPC > Subnets > Create Subnet
- Selecione sua VPC (docker-project-vpc)
- Nome da subnet: sub-az-b
- Availability Zone: us-east-1b
- IPv4 CIDR block: 192.168.0.32/27

### Security Groups

#### 1. Security Group para RDS (rds-sg)
- Inbound Rules:
  - MySQL/Aurora (3306) - Source: EC2 Security Group
- Outbound Rules:
  - All Traffic (0.0.0.0/0)

#### 2. Security Group para EFS (efs-sg)
- Inbound Rules:
  - NFS (2049) - Source: EC2 Security Group
- Outbound Rules:
  - All Traffic (0.0.0.0/0)

#### 3. Security Group para EC2 (ec2-sg)
- Inbound Rules:
  - HTTP (80) - Source: Load Balancer Security Group
  - SSH (22) - Source: Your IP
- Outbound Rules:
  - All Traffic (0.0.0.0/0)

#### 4. Security Group para Load Balancer (alb-sg)
- Inbound Rules:
  - HTTP (80) - Source: 0.0.0.0/0
  - HTTPS (443) - Source: 0.0.0.0/0
- Outbound Rules:
  - All Traffic (0.0.0.0/0)

## 2. Criar e Configurar o RDS

- Navegue até RDS > Databases > Create database
- Escolha Standard create
- Engine options: MySQL
- DB instance identifier: wordpress
- Master username: <username>
- Gerenciamento de credenciais: Self managed
- Master password: <password>
- Instance configuration: db.t3.micro
- Connectivity:
  - Don't connect to an EC2 compute resource
  - VPC: docker-project-vpc
  - Security Group: rds-sg
  - Availability Zone: No preference

Importante: Após a criação do RDS, conecte-se ao endpoint e crie um banco de dados para o WordPress:
```sql
CREATE DATABASE wordpress;
```

## 3. Criação do EFS

- Navegue até EFS > Create file system
- Name: efs
- VPC: docker-project-vpc
- Security Group: efs-sg
- Performance Mode: General Purpose
- Throughput Mode: Bursting

Comando para montar o EFS:
```bash
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport <efs-endpoint>.amazonaws.com:/ efs
```

## 4. Criação das Instâncias EC2

- Amazon Machine Image (AMI): Ubuntu Server 22.04 LTS
- Instance Type: t2.micro
- Network Settings:
  - VPC: docker-project-vpc
  - Subnet: sub-az-a (para primeira instância)
  - Subnet: sub-az-b (para segunda instância)
  - Security Group: ec2-sg
- Advanced Details > User Data:

```bash
#!/bin/bash

# Atualizar e instalar docker e NFS
sudo apt update -y
sudo apt install -y docker.io nfs-common

# Iniciar o serviço Docker
sudo systemctl start docker
sudo systemctl enable docker

# Montar o EFS
sudo mkdir /mnt/efs
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport <efs-endpoint>.amazonaws.com:/ /mnt/efs

# Instalação do Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Arquivo Docker Compose para WordPress e MySQL (RDS)
sudo tee /mnt/efs/docker-compose.yml > /dev/null <<EOF
version: '3.1'

services:
  wordpress:
    image: wordpress:latest
    restart: always
    ports:
      - 80:80
    environment:
      WORDPRESS_DB_HOST: <rds-endpoint>
      WORDPRESS_DB_USER: <user>
      WORDPRESS_DB_PASSWORD: <pass>
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - /mnt/efs:/var/www/html

  db:
    image: mysql:latest
    restart: always
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: <user>
      MYSQL_PASSWORD: <pass>
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    volumes:
      - db:/var/lib/mysql

volumes:
  wordpress:
  db:
EOF

if ! grep -q "<efs-endpoint>.amazonaws.com:" /etc/fstab; then
    echo "<efs-endpoint>.amazonaws.com:/ /mnt/efs nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport 0 0" | sudo tee -a /etc/fstab
fi

# Iniciar o Docker Compose
cd /mnt/efs
sudo docker-compose up -d
```

## 5. Criar AMI e Launch Template

### AMI
- Selecione uma das instâncias EC2 criadas
- Actions > Image and Templates > Create Image
- Defina um nome para a AMI

### Launch Template
- EC2 > Launch Templates > Create Launch Template
- Nome do template: wordpress-template
- Selecione a AMI criada anteriormente
- Instance type: t2.micro
- Security Group: ec2-sg
- User Data: mesmo script usado nas instâncias EC2

## 6. Configurar Load Balancer e Auto Scaling

### Target Group
- EC2 > Target Groups > Create Target Group
- Target type: Instances
- Target group name: wordpress-tg
- VPC: docker-project-vpc
- Protocol: HTTP
- Port: 80
- Health check path: /
- Register targets: selecione as duas instâncias EC2 criadas

### Application Load Balancer
- EC2 > Load Balancers > Create Load Balancer
- Tipo: Application Load Balancer
- Nome: wordpress-alb
- Scheme: Internet-facing
- VPC: docker-project-vpc
- Mappings: selecione us-east-1a e us-east-1b
- Security Group: alb-sg
- Listeners:
  - HTTP: 80
  - Target Group: wordpress-tg

### Auto Scaling Group
- EC2 > Auto Scaling Groups > Create Auto Scaling Group
- Nome: wordpress-asg
- Launch Template: wordpress-template
- VPC: docker-project-vpc
- Subnets: sub-az-a, sub-az-b
- Load Balancer: Attach to an existing load balancer
- Target Group: wordpress-tg
- Group Size:
  - Desired: 2
  - Minimum: 2
  - Maximum: 4
- Scaling Policies: Target tracking scaling policy
  - Metric: Average CPU Utilization
  - Target value: 70%