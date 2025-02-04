# 🖥️ Deploy de WordPress na AWS

## 👥 Integrantes

- Erykson Xavier
- Guilherme de Brito
---

## 📝 Sobre o Projeto

Este projeto realiza o deploy de um ambiente WordPress manualmente via **AWS Console**, incluindo:

- **Instância EC2** com Docker rodando WordPress.
- **Banco de Dados RDS (MySQL)** para armazenar os dados.
- **Amazon EFS** para armazenar arquivos estáticos.
- **Elastic Load Balancer (ELB)** para balanceamento de carga.

---

## 🔐 Configuração dos Security Groups

Antes de criar os recursos, configure os **Security Groups** no **AWS Console**.

### ** Regras de Security Groups Utilizadas**
| Tipo  | Protocolo | Porta | Origem      |
|-------|----------|------|-------------|
| SSH   | TCP      | 22   | 0.0.0.0/0   |
| HTTP  | TCP      | 80   | 0.0.0.0/0   |
| MYSQL/Aurora | TCP      | 3306 | 0.0.0.0/0   |
| NFS  | TCP      | 2049 | 0.0.0.0/0   |


##  Criando a Instância EC2

1. Acesse o **AWS Console** e vá para o serviço **EC2**.
2. Clique em **Launch Instance**.
3. Escolha o **sistema operacional** (Amazon Linux 2 ou Ubuntu).
4. Escolha o tipo da instância (**t3.micro** - Free Tier).
5. Selecione o **Security Group** configurado anteriormente.
6. No campo **User Data**, cole o seguinte script:

```bash
#!/bin/bash

sudo yum update -y
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
sudo chkconfig docker on
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo mv /usr/local/bin/docker-compose /bin/docker-compose
sudo yum install nfs-utils -y
sudo mkdir /mnt/efs/
sudo chmod +rwx /mnt/efs/
```

##  Criando Banco de dados

1. No AWS Console, vá até o serviço RDS.
2. Clique em Create Database e escolha MySQL.
Em Templates, selecione Free Tier.
3. Defina as credenciais do banco (Master username e password).
4. Escolha a mesma região e AZ da EC2.
5. Marque Public Access = No.
6. Em Connectivity, selecione o Security Group correto.
7. Defina um nome para o banco em Initial database name.
8. Clique em Create Database.

##  Criando EFS

1. No AWS Console, vá para EFS.
2. Clique em Create file system e defina um nome.
3. Vá até Network e altere o Security Group para permitir conexões.
4. Após criar, copie o DNS Name do EFS.

Montando o EFS na EC2
Na EC2, execute:

```bash
sudo mount -t nfs4 <DNS_DO_EFS>:/ /mnt/efs
```

Para tornar a montagem permanente:

```bash
echo "<DNS_DO_EFS>:/    /mnt/efs    nfs4    defaults,_netdev,rw    0   0" | sudo tee -a /etc/fstab

Confirme a montagem:

df -h
📄 Configurando o Docker e WordPress
1. Conectando-se à EC2 via SSH
ssh -i sua-chave.pem ec2-user@seu-ip-publico
2. Criando o arquivo docker-compose.yml

vim docker-compose.yml

3. Adicionando o seguinte conteúdo:
yaml

version: '3.3'
services:
  wordpress:
    image: wordpress:latest
    volumes:
      - /mnt/efs/wordpress:/var/www/html
    ports:
      - 80:80
    restart: always
    environment:
      WORDPRESS_DB_HOST: projetopb2.cen60c2qo92t.us-east-1.rds.amazonaws.com
      WORDPRESS_DB_USER: admin
      WORDPRESS_DB_PASSWORD: wordpress123
      WORDPRESS_DB_NAME: dbwordpress
      WORDPRESS_TABLE_CONFIG: wp_

4. Iniciando o contêiner

docker-compose up -d

5. Verificando se o WordPress está rodando

docker ps

6. Confirmando que os arquivos foram armazenados no EFS

ls /mnt/efs/wordpress
⚖️ Criando o Load Balancer (ELB)
No AWS Console, vá até Load Balancers.
Clique em Create Load Balancer e selecione Application Load Balancer.
Configure:
Listeners: Porta 80, protocolo HTTP.
Availability Zones: Selecione pelo menos duas AZs.
Na etapa Security Group, crie um novo ou use um existente.
Crie um Target Group e associe a sua EC2.
Finalize a criação e aguarde o status Active.
Para acessar o WordPress, copie o DNS do Load Balancer e cole no navegador.

✅ Testando o Ambiente
1. Acesse o WordPress pelo DNS do Load Balancer
Copie o DNS Name do Load Balancer e cole no navegador.

2. Verifique o Banco de Dados conectando-se ao MySQL

mysql -h <ENDPOINT_DO_RDS> -P 3306 -u admin -p
3. Testando o EFS

ls /mnt/efs/wordpress

🔗 Referências
Deploy WordPress com AWS RDS
WordPress Docker Official Images
Amazon EC2 Masterclass
Deploy Dockerized WordPress com AWS EFS

