# Projeto Fargate ECS
## Descrição

Criação de micro serviços que são alocados em containers hospedados pela Fargate, e esses seriam administrados pela ferramenta ECS. O primeiro é uma ferramenta que permite rodar os contêiner quando necessário utilizando outros servers e toda a gestão da OS dos contêiners é trabalho do fargate, dessa maneira o usuário só precisa se preocupar com o deploy e pagará apenas o que seu serviço consumir por hora enquanto estiver ativo. Já o ECS, é um serviço de administração em que será possível visualizar quanto ainda tem de espaço disponível, se algum contêiner parou de funcionar, retornar informações uteis de funcionamento...

## Configuração

Como nesse projeto iremos utilizar os recursos da AWS, é necessário baixar o aws cli e configurar suas chaves de acesso e a região que será utilizada,nesse caso **"us-east-1"**.  Esse console permite interagir e gerenciar serviços AWS além de automatizar tarefas e administrar recursos. Este [guia](https://docs.aws.amazon.com/cli/latest/userguide/welcome-examples.html) oficial da Amazon pode ser útil. 

## Desenvolvimento

### VPC

Primeiramente deve-se criar uma **VPC**(Virtual Private Cloud), um serviço que permite isolar seus recursos em um ambiente virtual privado na nuvem, além da personalização de sua rede e controle de sua infraestrutura. Nesse projeto, uma VPC foi criada e a partir dela criou-se uma **sub-rede** pública com disponiblidade para a região configurada no AWS cli, "us-east-1". Ademais, para haver conexão com a internet foi criado um **gateway**, que a partir da da definição de **rota** foi associado com a sub-rede criada anteriormente. 

```tf
#Criando rede
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# Criando gateway para a internet
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
}


# Criando subnet
resource "aws_subnet" "public_subnet_a" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.0.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true
}



resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
}
 
resource "aws_route" "public" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.main.id
}
 
resource "aws_route_table_association" "public" {
    subnet_id      = aws_subnet.public_subnet_a.id
    route_table_id = aws_route_table.public.id
}

```
A ultima configuração feita na rede foi o **security group**, um firewall virtual que define as regras de ingresso e egresso do tráfego de rede. Define-se aqui o ingresso apenas pela porta 80, que será definida como a porta de acesso do conteiner mais adiante, e o egresso sem restrições, facilitando o acesso a servidores que contenham as imagens de aplicações a serem executadas em containers. 

```tf
resource "aws_security_group" "ecs_tasks" {
  name   = "white-hart-sg-task-foo"
  vpc_id = aws_vpc.main.id
 
  ingress {
   protocol         = "tcp"
   from_port        = 80
   to_port          = 80
   cidr_blocks      = ["0.0.0.0/0"]
   ipv6_cidr_blocks = ["::/0"]
  }
 
  egress {
   protocol         = "-1"
   from_port        = 0
   to_port          = 0
   cidr_blocks      = ["0.0.0.0/0"]
   ipv6_cidr_blocks = ["::/0"]
  }
}

```



### Cluster

Com sua rede de uso pronta, agora deve-se criar um **cluster** ECS Fargate, um grupo lógico de recursos de computação em nuvem onde você pode executar e gerenciar os contêineres de suas tarefas em questão. A criação do cluster é feita apenas com as default options, caso queira definir o tipo de cluster e suas configurações como capacidade de computação por exemplo, deve-se usar **capacity providers**, que no caso abaixo está definindo que o cluster **foo** utilizará o Fargate. 

```tf
#Criando cluster ECS
resource "aws_ecs_cluster" "foo" {
  name = "white-hart"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

#Definindo uso do Fargate
resource "aws_ecs_cluster_capacity_providers" "foo" {
  cluster_name = aws_ecs_cluster.foo.name

  capacity_providers = ["FARGATE"]

  default_capacity_provider_strategy {
    base              = 1
    weight            = 100
    capacity_provider = "FARGATE"
  }
}

```

## Como usar

Para utilizar esse projeto você deve clonar este repositório, e dentro dele rodar no terminal os comandos **terraform init** e **terraform apply**. Agora ao entrar na plataforma da AWS poderá checar o funcionamento de suas criações.
