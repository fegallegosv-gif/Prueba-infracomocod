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


Variables

variable "aws_region" {
  default = "us-east-1"
}

variable "vpc_cidr" {
  default = "10.0.0.0/16"
}

variable "subnet_cidrs" {
  type    = list(string)
  default = ["10.0.1.0/24", "10.0.2.0/24"]
}



(main)

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
  }
}

provider "aws" {
  region = var.aws_region
}


module "vpc" {
  source       = "./modulos/vpc"
  vpc_cidr     = var.vpc_cidr
  subnets_cidr = var.subnet_cidrs
}


module "ec2" {
  source     = "./modulos/ec2"
  subnet_ids = module.vpc.subnet_ids
}


module "nlb" {
  source     = "./modulos/nlb"
  subnet_ids = module.vpc.subnet_ids
}

module "s3" {
  source        = "./modulos/s3"
  random_suffix = local.rand_id
}


locals


resource "random_string" "sufijo" {
  length  = 6
  special = false
  upper   = false
}

locals {
  rand_id     = random_string.sufijo.result
  environment = "eval"
}




outputs


output "resultado_vpc_id" {
  value = module.vpc.vpc_id
}

output "resultado_subredes" {
  value = module.vpc.subnet_ids
}

output "resultado_ec2" {
  value = module.ec2.instance_ids
}

output "resultado_s3_buckets" {
  value = module.s3.bucket_names
}

output "sufijo_generado" {
  value = local.rand_id
}



# Inicializa el proyecto y descarga providers
terraform init

# Formatea el código para que quede ordenado y alineado
terraform fmt -recursive

# Valida que la sintaxis no tenga errores
terraform validate

# Genera el plan de ejecución
terraform plan


$env:TF_LOG="DEBUG"
terraform plan
