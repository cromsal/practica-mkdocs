# Infraestructura en AWS con Terraform

## Descripción
Este proyecto configura la infraestructura en AWS utilizando Terraform. Incluye la creación de grupos de seguridad, reglas de seguridad y múltiples instancias EC2 para un entorno NFS, frontend, backend y un balanceador de carga.

## Configuración de Terraform

### Configuración del proveedor AWS
```hcl
provider "aws" {
  region = var.region
}
```

### Creación de Grupos de Seguridad
```hcl
resource "aws_security_group" "sg_NFS" {
  name        = var.sg_name_nfs
  description = var.sg_description
}

resource "aws_security_group" "sg_backend" {
  name        = var.sg_name_backend
  description = var.sg_description
}

resource "aws_security_group" "sg_frontend" {
  name        = var.sg_name_frontend
  description = var.sg_description
}

resource "aws_security_group" "sg_loadbalancer" {
  name        = var.sg_name_loadbalancer
  description = var.sg_description
}
```

### Reglas de Seguridad
#### Backend
```hcl
resource "aws_security_group_rule" "ingress_backend" {
  security_group_id = aws_security_group.sg_backend.id
  type              = "ingress"

  count       = length(var.allowed_ingress_ports_backend)
  from_port   = var.allowed_ingress_ports_backend[count.index]
  to_port     = var.allowed_ingress_ports_backend[count.index]
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}

resource "aws_security_group_rule" "egress_backend" {
  security_group_id = aws_security_group.sg_backend.id
  type              = "egress"
  from_port   = 0
  to_port     = 0
  protocol    = "-1"
  cidr_blocks = ["0.0.0.0/0"]
}
```
#### Frontend
```hcl
resource "aws_security_group_rule" "ingress_frontend" {
  security_group_id = aws_security_group.sg_frontend.id
  type              = "ingress"
  count       = length(var.allowed_ingress_ports)
  from_port   = var.allowed_ingress_ports[count.index]
  to_port     = var.allowed_ingress_ports[count.index]
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}

resource "aws_security_group_rule" "egress_frontend" {
  security_group_id = aws_security_group.sg_frontend.id
  type              = "egress"
  from_port   = 0
  to_port     = 0
  protocol    = "-1"
  cidr_blocks = ["0.0.0.0/0"]
}
```
#### LoadBalancer
```hcl
resource "aws_security_group_rule" "ingress_loadbalancer" {
  security_group_id = aws_security_group.sg_loadbalancer.id
  type              = "ingress"
  from_port   = 80
  to_port     = 80
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}

resource "aws_security_group_rule" "egress_loadbalancer" {
  security_group_id = aws_security_group.sg_loadbalancer.id
  type              = "egress"
  from_port   = 0
  to_port     = 0
  protocol    = "-1"
  cidr_blocks = ["0.0.0.0/0"]
}
```
#### NFS
```hcl
resource "aws_security_group_rule" "ingress_nfs" {
  security_group_id = aws_security_group.sg_NFS.id
  type              = "ingress"
  count       = length(var.allowed_ingress_ports_nfs)
  from_port   = var.allowed_ingress_ports_nfs[count.index]
  to_port     = var.allowed_ingress_ports_nfs[count.index]
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}

resource "aws_security_group_rule" "egress_nfs" {
  security_group_id = aws_security_group.sg_NFS.id
  type              = "egress"
  from_port   = 0
  to_port     = 0
  protocol    = "-1"
  cidr_blocks = ["0.0.0.0/0"]
}
```

### Creación de Instancias EC2
```hcl
resource "aws_instance" "instancia_nfs" {
  ami             = var.ami_id
  instance_type   = var.instance_type
  key_name        = var.key_name
  security_groups = [aws_security_group.sg_NFS.name]
  tags = { Name = var.instance_name_nfs }
}

resource "aws_instance" "instancia_frontend" {
  ami             = var.ami_id
  instance_type   = var.instance_type
  key_name        = var.key_name
  security_groups = [aws_security_group.sg_frontend.name]
  tags = { Name = var.instance_name_frontend }
}

resource "aws_instance" "instancia_frontend1" {
  ami             = var.ami_id
  instance_type   = var.instance_type
  key_name        = var.key_name
  security_groups = [aws_security_group.sg_frontend.name]
  tags = { Name = var.instance_name_frontend1 }
}

resource "aws_instance" "instancia_backend" {
  ami             = var.ami_id
  instance_type   = var.instance_type
  key_name        = var.key_name
  security_groups = [aws_security_group.sg_backend.name]
  tags = { Name = var.instance_name_backend }
}

resource "aws_instance" "instancia_loadbalancer" {
  ami             = var.ami_id
  instance_type   = var.instance_type
  key_name        = var.key_name
  security_groups = [aws_security_group.sg_loadbalancer.name]
  tags = { Name = var.instance_name_loadbalancer }
}
```

### Configuración de IP Elástica
```hcl
resource "aws_eip" "ip_elastica" {
  instance = aws_instance.instancia_loadbalancer.id
}

output "elastic_ip" {
  value = aws_eip.ip_elastica.public_ip
}
```