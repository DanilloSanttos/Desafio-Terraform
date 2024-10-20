                	Análise Técnica do Código Terraform

•	Provider AWS: Neste trecho temos a definição do provedor da nuvem, e sua região para a criação das instâncias.

•	Variáveis: Neste trecho temos a definição das variáveis que serão utilizadas no projeto, elas servirão de apoio para modificações do código e sua reutilização.

•	Chave SSH: Neste trecho tls_private_key temos a criação de uma chave privada utilizando o algoritmo RSA com 2048 bits. O algoritmo RSA gera pares de chaves que podem encriptar uma mensagem. Nesse caso ele gerara pares necessário para acessar a instancia EC2. No Trecho aws_key_pair ,  temos a geração da chave publica que foi gerada a partir da chave privada utilizando as variáveis criadas anteriormente.

•	VPC e Sub-Rede: Neste trecho aws_vpc  temos a criação de uma rede virtual pública incluindo acesso ao suporte de DNS através das chaves enable_dns_support e enable_dns_hostnames. No trecho aws_subnet temos a criação de uma sub-rede dentro de uma rede virtual CIDR as zonas de disponibilidades.

•	Gateway de Internet e Roteamento: No trecho aws_internet_gateway  é necessário para que as instancias tenham acesso a internet, no aws_route_table é criada uma tabela com uma rota padrão, que direciona todo o trafego externo para o gateway. No trecho aws_route_table_association associa a sub-rede a tebela de rotas garantindo que o trafego siga as rotas definidas.

•	Grupo de Segurança: No aws_security_group é criado um grupo de segurança que permite a entrada de trafego SSH e todo trafego de saída.

•	Imagem e Instância EC2: No trecho aws_ami é utilizado o recurso Amazon Machine Image mais recente (Debian 12). No trecho aws_instance é criada uma instancia EC2 que utiliza a AMI, o tipo de instancia t2.micro esta associada a sub-rede e ao grupo de segurança criados.  Utilizamos o user_data para realizar uma atualização do sistema, e também temos 20GB associado a instancia através do volume EBS.

•	Outputs: private_key exibe a chave privada necessária para o acesso, é marcado como sensitive para evitar exposição. ec2_public_ip mostra o endereço IP público da instancia.



                              main.tf modificado

provider "aws" {
  region = "us-east-1"
}

variable "projeto" {
  description = "Nome do projeto"
  type        = string
  default     = "VExpenses"
}

variable "candidato" {
  description = "Nome do candidato"
  type        = string
  default     = "SeuNome"
}

resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "${var.projeto}-${var.candidato}-key"
  public_key = tls_private_key.ec2_key.public_key_openssh
}

resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.projeto}-${var.candidato}-vpc"
  }
}

resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "${var.projeto}-${var.candidato}-subnet"
  }
}

resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-igw"
  }
}

resource "aws_route_table" "main_route_table" {
  vpc_id = aws_vpc.main_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main_igw.id
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table"
  }
}

resource "aws_route_table_association" "main_association" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_route_table.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table_association"
  }
}

resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH apenas de IP específico e tráfego de saída controlado"
  vpc_id      = aws_vpc.main_vpc.id

  # Regras de entrada
  ingress {
    description      = "Allow SSH from a specific IP range"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["203.0.113.0/24"]
  }

  # Regras de saída
  egress {
    description      = "Allow HTTP and HTTPS outbound traffic"
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }

   egress {
    description = "Allow HTTPS outbound traffic"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-sg"
  }
}

data "aws_ami" "debian12" {
  most_recent = true

  filter {
    name   = "name"
    values = ["debian-12-amd64-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["679593333241"]
}

resource "aws_instance" "debian_ec2" {
  ami             = data.aws_ami.debian12.id
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.main_subnet.id
  key_name        = aws_key_pair.ec2_key_pair.key_name
  security_groups = [aws_security_group.main_sg.name]

  associate_public_ip_address = true

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    delete_on_termination = true
  }

  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
               apt-get install nginx -y
              systemctl start nginx
              systemctl enable nginx
              EOF

  tags = {
    Name = "${var.projeto}-${var.candidato}-ec2"
  }
}

output "private_key" {
  description = "Chave privada para acessar a instância EC2"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}

output "ec2_public_ip" {
  description = "Endereço IP público da instância EC2"
  value       = aws_instance.debian_ec2.public_ip
}




                 Modificação e Melhoria do Código Terraform

•	Restrição do tráfego SSH: Foi alterado o grupo de segurança para permitir conexões SSH apenas de um intervalo especifico de IPs ao invés de permitir de qualquer lugar.

•	Bloqueio de IPv6: Foi feito bloqueio de trafego IPv6 pois não foi uma especificação necessária.

•	Desativação de tráfego desnecessário: Especificamos portas e protocolos para não permitir todos os tipos de saída.




                                Instruções de Uso

	1 Pré-requisitos
Antes de executar este projeto, certifique-se de ter os seguintes componentes instalados:

Terraform:

Baixe e instale a versão mais recente do Terraform a partir do site oficial: Terraform Download.
Verifique a instalação do Terraform rodando o comando: terraform --version	
terraform --version	

Chave de acesso AWS:

Tenha suas credenciais da AWS prontas (Access Key ID e Secret Access Key). Se preferir, você pode definir estas credenciais como variáveis de ambiente:

export AWS_ACCESS_KEY_ID=your_access_key
export AWS_SECRET_ACCESS_KEY=your_secret_key
export AWS_DEFAULT_REGION=us-east-1

   2 Estrutura de Arquivos
O projeto está dividido em três arquivos principais:

main.tf: Define os recursos da infraestrutura, como VPC, sub-rede, gateway de internet, grupo de segurança, instância EC2, e os comandos necessários para a instalação do Nginx.
variables.tf: Contém as variáveis reutilizáveis que facilitam a customização do projeto (como nome do projeto e nome do candidato).
outputs.tf: Define os outputs da infraestrutura, como o IP público da instância EC2 e a chave privada para acesso via SSH.

   3 Passos para Execução
3.1. Inicializar o Terraform
No diretório onde seus arquivos .tf estão localizados, inicialize o Terraform para baixar os provedores e preparar o ambiente de execução: terraform init

3.2. Aplicar o Plano
Se tudo estiver correto, aplique o plano para criar a infraestrutura na AWS: terraform apply
Será solicitado que você confirme a execução digitando yes.

3.3. Verificar Outputs
Ao final da execução do terraform apply, você verá os outputs definidos no arquivo outputs.tf. Estes outputs incluem:

A chave privada (private_key) para acessar a instância EC2 via SSH.
O endereço IP público (ec2_public_ip) da instância EC2 provisionada.

3.4. Acessar a Instância EC2
Salve a chave privada exibida no output como um arquivo .pem, por exemplo: my-key.pem.
Ajuste as permissões do arquivo para garantir que ele seja seguro: chmod 400 my-key.pem

Conecte-se à instância EC2 utilizando o endereço IP público e a chave privada: ssh -i my-key.pem admin@<ec2_public_ip>

Ao se conectar à instância EC2, o Nginx já estará instalado e rodando, pois o script de inicialização (user_data) configurado no Terraform faz a instalação automática.

4. Destruir a Infraestrutura
Quando a infraestrutura não for mais necessária, você pode destruir todos os recursos provisionados para evitar custos adicionais: terraform destroy
O comando pedirá confirmação. Digite yes para confirmar a destruição.

