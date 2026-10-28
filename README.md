# Prueba-infracomocod

(modulo de vpc variables) 

variable "vpc_cidr" {
  description = "CIDR de la VPC"
  type        = string
}

variable "subnets_cidr" {
  description = "Lista con los CIDR para las subredes"
  type        = list(string)
}


(main de la vpc)

locals {
  nombre_vpc = "vpc-evaluacion"
}

resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true

  tags = {
    Name = local.nombre_vpc
  }
}

resource "aws_subnet" "this" {
  count      = length(var.subnets_cidr)
  vpc_id     = aws_vpc.this.id
  cidr_block = var.subnets_cidr[count.index]

  tags = {
    Name = "subred-${count.index + 1}"
  }
}


(outputs de la vpc)


output "vpc_id" {
  value = aws_vpc.this.id
}

output "subnet_ids" {
  value = aws_subnet.this[*].id
}



(modulo de la ec2 variable)

variable "subnet_ids" {
  description = "IDs de las subredes donde se desplegarán las instancias"
  type        = list(string)
}




(ec2 main)

locals {
  tipo_instancia = "t2.micro"
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }
}

resource "aws_instance" "this" {
  count         = 2
  ami           = data.aws_ami.amazon_linux.id
  instance_type = local.tipo_instancia
  subnet_id     = var.subnet_ids[count.index % length(var.subnet_ids)]

  tags = {
    Name = "servidor-${count.index + 1}"
  }
}


(outputs de la ec2)

output "instance_ids" {
  value = aws_instance.this[*].id
}



(variables de la nbl)


variable "subnet_ids" {
  description = "Subredes para asociar al NLB"
  type        = list(string)
}


(main de la nbl)

locals {
  nombres_nlb = toset(["nlb-principal", "nlb-secundario"])
}

resource "aws_lb" "this" {
  for_each           = local.nombres_nlb
  name               = each.key
  internal           = true
  load_balancer_type = "network"
  subnets            = var.subnet_ids

  tags = {
    Name = each.key
  }
}



(outputs)


output "nlb_arns" {
  value = { for k, v in aws_lb.this : k => v.arn }
}


(variables de la s3)

variable "random_suffix" {
  description = "Sufijo aleatorio de 6 caracteres generado fuera"
  type        = string
}

(main)

locals {
  prefijo = "bucket-privado"
}


resource "aws_s3_bucket" "this" {
  count         = 2
  bucket        = "${local.prefijo}-${count.index + 1}-${var.random_suffix}"
  force_destroy = true
}


resource "aws_s3_bucket_public_access_block" "this" {
  count  = length(aws_s3_bucket.this)
  bucket = aws_s3_bucket.this[count.index].id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}


outputs

output "bucket_names" {
  value = aws_s3_bucket.this[*].id
}



