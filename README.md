
# Projeto de Infraestrutura na AWS

Este projeto demonstra a criação de uma infraestrutura básica na AWS usando a CLI da AWS. O objetivo é configurar uma VPC, subnets, um sistema de arquivos EFS, uma instância RDS MySQL e uma instância EC2 com Docker e WordPress.

## Criar a VPC e Subnets

```bash
# Criar uma VPC com um bloco CIDR de 10.0.0.0/16
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# Criar uma subnet na zona de disponibilidade us-east-1a
aws ec2 create-subnet   --vpc-id <VPC_ID>   --cidr-block 10.0.1.0/24   --availability-zone us-east-1a

# Criar uma subnet na zona de disponibilidade us-east-1b
aws ec2 create-subnet   --vpc-id <VPC_ID>   --cidr-block 10.0.2.0/24   --availability-zone us-east-1b
```

## Criar o Sistema de Arquivos EFS

```bash
# Criar um sistema de arquivos EFS
aws efs create-file-system --creation-token <unique-token>

# Criar pontos de montagem para o EFS nas subnets
aws efs create-mount-target   --file-system-id <FileSystemId>   --subnet-id <Subnet_ID_1>

aws efs create-mount-target   --file-system-id <FileSystemId>   --subnet-id <Subnet_ID_2>
```

## Criar um Grupo de Subnets para o RDS

```bash
# Criar um grupo de subnets para o RDS
aws rds create-db-subnet-group   --db-subnet-group-name my-db-subnet-group   --db-subnet-group-description "My DB Subnet Group"   --subnet-ids <Subnet_ID_1> <Subnet_ID_2>
```

## Criar uma Instância RDS MySQL

```bash
# Criar uma instância RDS MySQL
aws rds create-db-instance   --db-instance-identifier mydbinstance   --db-instance-class db.t3.micro   --engine mysql   --master-username admin   --master-user-password <DB_PASSWORD>   --allocated-storage 20   --vpc-security-group-ids <SECURITY_GROUP_ID>   --db-subnet-group-name my-db-subnet-group
```

## Criar um Par de Chaves

```bash
# Criar um par de chaves
aws ec2 create-key-pair --key-name MyKeyPair --query 'KeyMaterial' --output text > MyKeyPair.pem
chmod 400 MyKeyPair.pem
```

## Criar um Grupo de Segurança

```bash
# Criar um grupo de segurança
aws ec2 create-security-group   --group-name my-security-group   --description "My security group"   --vpc-id <VPC_ID>

# Autorizar o tráfego de entrada para SSH
aws ec2 authorize-security-group-ingress   --group-id <SECURITY_GROUP_ID>   --protocol tcp   --port 22   --cidr xxx.xxx.xxx.xxx/32

# Autorizar o tráfego de entrada para HTTP
aws ec2 authorize-security-group-ingress   --group-id <SECURITY_GROUP_ID>   --protocol tcp   --port 80   --cidr xxx.xxx.xxx.xxx/32

# Autorizar o tráfego de entrada para HTTPS
aws ec2 authorize-security-group-ingress   --group-id <SECURITY_GROUP_ID>   --protocol tcp   --port 443   --cidr xxx.xxx.xxx.xxx/32
```

## Criar uma Instância EC2

```bash
# Criar uma instância EC2
aws ec2 run-instances   --image-id <AMI_ID>   --count 1   --instance-type t3.micro   --key-name MyKeyPair   --security-group-ids <SECURITY_GROUP_ID>   --subnet-id <SUBNET_ID>   --associate-public-ip-address
```

## Configuração da Instância EC2

Dentro da EC2, execute o seguinte script:

```bash
#!/bin/bash

# 1. Atualizar o sistema
sudo yum update -y

# 2. Instalar Docker
sudo dnf install docker -y
sudo service docker start
sudo usermod -a -G docker ec2-user

# 3. Instalar Git
sudo yum install git -y

# 4. Instalar Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# 5. Instalar o cliente EFS
sudo yum install -y amazon-efs-utils

# 6. Criar um diretório para o EFS
mkdir ~/efs

# 7. Montar o EFS (substitua <EFS_ID> pelo ID do seu EFS)
sudo mount -t efs <EFS_ID> ~/efs

# 8. Criar um diretório para o WordPress
mkdir ~/wordpress && cd ~/wordpress

# 9. Criar o arquivo docker-compose.yml
cat <<EOL > docker-compose.yml
version: '3.7'

services:
  wordpress:
    image: wordpress:latest
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: <db_host>
      WORDPRESS_DB_USER: usuario_wp
      WORDPRESS_DB_PASSWORD: senha_segura_wp
      WORDPRESS_DB_NAME: wordpress_db
    volumes:
      - ~/efs:/var/www/html
    restart: always

volumes:
  db_data:
EOL

# 10. Iniciar o Docker Compose
docker-compose up -d

# 11. Adicionar o arquivo docker-compose.yml ao GitHub
git init
git add docker-compose.yml
git commit -m "Adicionar arquivo docker-compose para WordPress e MySQL"
git remote add origin https://github.com/<seu-usuario>/<seu-repositorio>.git
git push -u origin master
```

## Criar uma AMI da Instância EC2

```bash
# Criar uma AMI da instância EC2
aws ec2 create-image --instance-id <instance-id> --name "MyWordPressAMI" --no-reboot
```

## Criar um Modelo de Execução

```bash
# Criar um modelo de execução
aws ec2 create-launch-template --launch-template-name MyLaunchTemplate   --version-description "Version1"   --launch-template-data '{"ImageId":"<ami-id>", "InstanceType":"t3.micro", "KeyName":"<key-name>", "SecurityGroupIds":["<security-group-id>"], "UserData":"<base64_encoded_user_data>"}'
```

## Adicionar Instruções para a Inicialização da Máquina

```bash
# Instruções para a inicialização da máquina:
sudo yum update -y
sudo mount -t efs fs-08a19a5e85af1072b:/ /home/ec2-user/efs
cd ~/wordpress
docker-compose up -d
```

## Criar um Grupo de Auto Scaling

```bash
# Criar um grupo de auto scaling
aws autoscaling create-auto-scaling-group --auto-scaling-group-name MyAutoScalingGroup   --launch-template "LaunchTemplateName=MyLaunchTemplate,Version=1"   --min-size 3   --max-size 6   --desired-capacity 2   --vpc-zone-identifier "<subnet-1-id>,<subnet-2-id>"
```

## Criar um Load Balancer

```bash
# Criar um Load Balancer
aws elbv2 create-load-balancer --name <load-balancer-name>   --subnets <subnet-1-id> <subnet-2-id>   --security-groups <security-group-id>
```

## Criar um Grupo de Destino

```bash
# Criar um grupo de destino
aws elbv2 create-target-group --name <target-group-name>   --protocol HTTP   --port 80   --vpc-id <vpc-id>

# Registrar instâncias no grupo de destino
aws elbv2 register-targets --target-group-arn <target-group-arn>   --targets Id=<instance-id-1> Id=<instance-id-2>
```

## Criar um Listener para o Load Balancer

```bash
# Criar um listener para o Load Balancer
aws elbv2 create-listener --load-balancer-arn <load-balancer-arn>   --protocol HTTP --port 80   --default-actions Type=forward,TargetGroupArn=<target-group-arn>
```

## User Data para a Instância EC2

Este script será executado na inicialização da instância EC2.
