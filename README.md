# PROJETO 2 -  WORDPRESS + DOCKER

## - TECNOLOGIAS UTILIZADAS:

#### ° Amazon Web Services com os serviços de EC2 - Auto Scaling Groups, Load Balancer, Security Groups e Launch Template; RDS (Banco de Dados); EFS (Para manter arquivos estáticos do Wordpress); VPC (para gerar as subnets e criar um interface de rede para a aplicação) e CloudWatch (para criar os alarmes que serão utilizados para as políticas de scaling).

#### ° Docker para rodar o container de Wordpress por meio de um docker-compose.yml.

#### ° PuTTy para acessar a instância EC2 via SSH e verificar o funcionamento do banco de dados e do EFS.

## - Primeiro passo (Configuração do ambiente AWS):
### Criação de uma VPC para o projeto:

####  ° Criei uma VPC com 2 subnets públicas e 2 subnets privadas, para ter acesso a mais zonas de disponibilidade.

![alt text](image-4.png)

### Criação de um Security Group específico para cada serviço do projeto.

#### ° SG-LoadBalancer: Inbound Rules adicionar HTTP para origem "All", para permitir acesso à página web.

![alt text](image-47.png)

#### ° SG-LaunchTemplate: Inbound Rules adicionar HTTP para origem do SG-LoadBalancer, MySQL para origem do SG-RDS e SSH para origem "All" para permitir conexão entre instâncias e banco de dados e para acessar a instância via chave SSH .ppk.

![alt text](image-48.png)

#### ° SG-EFS: Inbound Rules adicionar NFS para origem do SG-LaunchTemplate para permitir conexão entre EFS e EC2.

![alt text](image-49.png)

#### °SG-RDS: Inbound Rules adicionar MySQL para origem do SG-LaunchTemplate para permitir conexão entre o banco de dados e a instância.

![alt text](image-50.png)

#### ° SG-ASG: Inbound Rules adicionar HTTP para SG-LoadBalancer e MySQL para SG-RDS para permitir conexão entre o ASG, banco de dados e http para o LB.

![alt text](image-51.png)

#### ° Essas configurações permitirão a comunicação entre os serviços e evitam a exposição de recursos privados.

### Criação do banco de dados RDS (MySQL)

#### ° Acessar o serviço Aurora and RDS e em seguida clicar para criar um database.

#### ° Nas configurações, deve ser selecionado o MySQL (de preferência escolha a versão mais recente para evitar conflitos); escolhi a template Free-Tier (Single AZ) com o tipo de instância db-t3.micro; após escolher o tipo de instância, escolha um nome para o DB, o MasterUsername e a Master Password; na seção de conectividade/network selecione a mesma VPC da EC2 que será criada, subnet group default, o SG criado para o RDS e coloque o acesso privado ao DB.

#### ° Em conectividade essas são as configurações (Sem acesso público e selecionar a VPC do projeto, sem utilizar subnet pública no RDS):

![alt text](image-52.png)

#### ° No final da página de configurações existe um detalhe importante. Na aba de configurações adicionais você deve inserir um nome incial para o DB, esse nome será colocado nas variáveis de ambiente do Userdata.

![alt text](image-2.png)

#### ° DB criado e funcionando:

![alt text](image-3.png)

### Criação do EFS (Elastic File System)

#### ° Pesquise por EFS, na aba de "File Systems" selecione "Create File System".
#### ° Escolha um nome para esse EFS e selecione a mesma VPC utilizada no RDS e na EC2 e selecione o SG-EFS. Clique para criar e aguarde a criação do EFS.

![alt text](image-6.png)

![alt text](image-7.png)

![alt text](image-8.png)

#### ° Comandos que podem ser utilizados no Userdata para montagem do EFS.

### Criação e configuração do Userdata

#### ° Depois de ter criado o EFS e o RDS podemos seguir com a criação e configuração do Userdata, pois durante o script será necessário preencher com dados do DB e do Elastic File System.

```
#!/bin/bash

sudo yum update -y
sudo yum install -y docker
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ec2-user

DOCKER_COMPOSE_VERSION=$(curl -s https://api.github.com/repos/docker/compose/releases/latest | grep -Po '"tag_name": "\K[^"]+')
sudo curl -L "https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

sudo yum install -y amazon-efs-utils

sudo mkdir -p /mnt/efs
echo "fs-047913fad21bca3be:/ /mnt/efs efs _netdev,tls 0 0" | sudo tee -a /etc/fstab
sudo mount -t efs -o tls fs-047913fad21bca3be:/ /mnt/efs

echo "<?php
http_response_code(200);
?>" | sudo tee /mnt/efs/healthcheck.php

mkdir -p /home/ec2-user/wordpress
cd /home/ec2-user/wordpress

cat <<EOF > docker-compose.yml
version: '3.3'

services:
  wordpress:
    image: wordpress:latest
    container_name: wordpress
    restart: always
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: db-wordpress.c7mgqcao61ww.us-east-1.rds.amazonaws.com (SEU ENDPOINT DO RDS)
      WORDPRESS_DB_USER: admin (NOME DO MASTERUSER)
      WORDPRESS_DB_PASSWORD: SenhaForte123 (MASTERPASSWORD)
      WORDPRESS_DB_NAME: wordpressdocker (INITIAL DB NAME - PREENCHIDO NAS CONFIGS ADICIONAIS DO DB)
    volumes:
      - /mnt/efs:/var/www/html
    networks:
      - wp_network

networks:
  wp_network:
    driver: bridge
EOF

sudo docker-compose up -d
```

#### Esse é o Userdata que será utilizado na EC2.
#### Explicando o Userdata:
#### ° O código começa atualizando os pacotes (yum update), instalando (yum install), ativando (systemctl enable), iniciando (systemctl start) o Docker e dando permissões (usermod -aG docker ec2-user) para o usuário conseguir acessar e configurar o Docker.

#### ° Na próxima linha o código executa a instalação do docker-compose, que será utilizado para conectar o Wordpress com o DB e EFS.

#### ° Logo após, o yum install é chamado novamente para instalar as utilidades da Amazon para o EFS. Essa instalação é seguida pela criação do diretório para montagem e armazenamento dos arquivos do EFS.

#### ° A criação do arquivo healthcheck.php (com código de resposta HTTP 200 OK) serve para direcionar o ping path no Health Check do Load Balancer. Com esse arquivo, o Load Balancer consegue fazer a verificação da página, evitando conflitos caso estiver na página de install ou de login.

#### ° Por último, o código apresenta o docker-compose que configura o ambiente para conectar o Wordpress com o DB e com o EFS. O último comando executa o docker-compose (docker-compose up).

### Criação do Launch Template (para ser utilizado no Auto Scaling Group)

#### ° No serviço de EC2, acesse a aba de Launch Templates.

#### ° Clique para criar uma Launch Template. Nas configurações faça como uma criação de instância normal, selecione a VPC usada no RDS e EFS, tipo t2.micro, AMI 2023 Linux, escolha o SG-LaunchTemplate, inclua a chave SSH ou crie uma e no final em "Detalhes avançados" coloque o Userdata criado anteriormente.

![alt text](image-9.png)

![alt text](image-10.png)


### Criação do Load Balancer

#### ° Entre no serviço de Load Balancers e selecione para criar um LB, escolha o Classic Load Balancer.

#### ° Nas configurações escolha: internet-facing, a mesma VPC usada nos outros serviços, escolha as duas subnets públicas dessa VPC, o SG-LoadBalancer, em Listener e Instance protocol coloque o protocólo HTTP na porta 80, em Health Checks coloque o ping path = "/healthcheck.php" (arquivo criado no Userdata) e o protocólo em HTTP porta 80, em "Configurações avançadas de Health Check" mude os intervalos de verificação caso julgue necessário, por último clique para criar o Load Balancer.

![alt text](image-12.png)

![alt text](image-13.png)

![alt text](image-19.png)

### Criação do Auto Scaling Group

#### ° No serviço de EC2 selecione a aba Auto Scaling Groups e clique para criar um novo.

#### ° Nas configurações: escolha um nome para o ASG, em Launch Template escolha a template criada anteriormente, escolha a mesma VPC, subnets e SG-ASG, associe o Auto Scaling Group com o Load Balancer criado no passo anterior, nas configurações de quantidade de instância e condições de scaling in e scaling out coloque para ter no mínimo 2 EC2, desejado 2 e máximo 3 instâncias. Por último revise as configurações e clique para criar o Auto Scaling Group.

![alt text](image-14.png)
#### ° Escolha a Launch Template criada anteriormente.

![alt text](image-20.png)
#### ° Escolha a VPC do projeto e as subnets públicas dessa VPC.

![alt text](image-21.png)
#### ° Selecione a quantidade desejada de instâncias = 2, Min = 2 e Máx = 3. Esse máx será atingindo conforme as métricas e alarmes do Cloudwatch que serão criados no próximo passo. Por isso não precisa criar Scaling Policies agora.

#### ° Escolha o Classic Load Balancer criado anteriormente.
![alt text](image-15.png)

#### ° Se você for na aba de instâncias, as duas EC2 já vão estar criadas e você poderá acessá-las pelo navegador utilizando o DNS do Load Balancer (que foi associado ao Auto Scaling Group). Caso queira acessá-las, pode conectar-se via SSH utilizando o software PuTTy; quando já conectado, ao dar o comando "docker ps" você verificará que o container do Wordpress estará rodando normalmente.

![alt text](image-22.png)

### Criação de uma métrica de alarme no Cloudwatch

#### ° No serviço de Cloudwatch clique em "Alarms" e selecione para criar um novo alarme.

![alt text](image-23.png)

#### ° Selecione a métrica EC2 e "By Auto Scaling Group", marque a check box de CPUUtilization e faça as configurações:

#### ° Aqui nessa configuração, será usada a média da utilização de CPU nos últimos 5 minutos, se caso o alarme perceber o comportamento de uso maior que 55%, a política do ASG usando esse alarme adicionará uma EC2. (O restante das configurações são default)

![alt text](image-39.png)

#### ° Agora devemos criar outra para remover uma EC2 quando a utilização for inferior a 25%. (DETALHE QUE ELE REMOVERÁ APENAS SE TIVER 3 E O USO DIMINUIR, POIS A CAPACIDADE DESEJADA DO ASG É SEMPRE 2 EC2's).

![alt text](image-40.png)


#### ° Depois de criar os alarmes, devemos ir para o Auto Scaling Group na aba de Automatic Scaling e criar duas políticas dinâmicas de escalonamento (uma para adicionar instâncias e outra para remover instâncias com base nos alarmes):

![alt text](image-26.png)

![alt text](image-41.png)

![alt text](image-42.png)

![alt text](image-43.png)

#### ° Você pode criar primeiro as políticas dinâmicas e depois os alarmes (anexando eles às políticas) caso queira. Nessa ordem que eu fiz, você cria primeiro os alarmes e depois as políticas dinâmicas (associando as políticas aos alarmes já criados).

#### ° Para testar essas métricas e a criação ou remoção de instâncias, utilize um comando para dar "stress" no CPU de uma EC2: 
```
sudo yum install -y stress
stress --cpu $(nproc --all)
```

#### ° Depois de dar "stress" o alarme foi ativado e a terceira EC2 foi criada:

![alt text](image-45.png)

#### ° Após finalizar o "stress" uma das 3 EC2 foi finalizada:

![alt text](image-46.png)

## - CONCLUSÃO

### TESTE DE TERMINAR UMA DAS DUAS INSTÂNCIAS E VERIFICAR O FUNCIONAMENTO DO WORDPRESS:

#### ° Após terminar uma das EC2, o site continuou funcionando normalmente.
![alt text](image-54.png)

![alt text](image-55.png)

#### ° Logo após alguns minutos uma nova instância foi criada para manter o mínimo de 2 estabelecido no Auto Scaling Group

![alt text](image-56.png)

### - Wordpress acessível pelo DNS do LoadBalancer

![alt text](image-33.png)

![alt text](image-34.png)

![alt text](image-35.png)

#### ° Para acessar o banco de dados e verificar sua funcionalidade via SSH da EC2: 
```
sudo docker run --rm -it mysql:8.0 mysql -h <ENDEREÇO_DO_RDS> -u <USUÁRIO> -p
```
#### ° O usuário "testador" criado está presente no Banco de Dados:
![alt text](image-36.png)

#### ° Testando o funcionamento do EFS:

#### ° Montagem feita com sucesso e nesse diretório "upgrade" estarão as mídias que o usuário fez upload no Wordpress:

![alt text](image-37.png)

#### ° Ao entrar conectar-se na EC2 e utilizar o comando "docker ps" é possível ver o contâiner do Wordpress rodando:

![alt text](image-57.png)

#### ° Portanto, é possível verificar que a aplicação funcionou corretamente: gerando as EC2's pelo Auto Scaling Group (Launch Template com o Userdata contendo o docker-compose.yml e configurações que conectam a instância ao RDS e ao EFS); conectando esse ASG ao Load Balancer; acessando o Wordpress pelo DNS do Load Balancer; alarmando o uso de CPU das EC2's por meio do CloudWatch e escalonando a criação ou remoção de instâncias pelas métricas do ASG.



