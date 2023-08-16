```bash
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
}
provider "yandex" {
token     = ""
cloud_id  = "b1gdqrsimkog95ftf7a6"
folder_id = "b1g6v0h3tgtvb9189u9c"
}

#Создаем 1й веб сервер в зоне С и виртаульную сеть (network1)  с подсетью (subnet2)
resource "yandex_compute_instance" "Nginx1" {
  name        = "nginx1"
  platform_id = "standard-v3"
  zone        = "ru-central1-c"

 resources {
    cores  = "2"
    memory = "6"
  }
  boot_disk {
    initialize_params {
      image_id = "fd861l8ckd35g2rrhsfd"
      size =  20
    }
}
  network_interface {
    subnet_id = yandex_vpc_subnet.subnet2.id
    nat        = true
    ipv4       = true
  }

  metadata = {
    user-data = "${file("./meta.txt")}"
  }
 }

resource "yandex_vpc_network" "network1" {
  name = "my-network"
}

resource "yandex_vpc_subnet" "subnet2" {
  name           = "subnet2"
  zone           = "ru-central1-c"
  network_id     = yandex_vpc_network.network1.id
  v4_cidr_blocks = ["10.2.0.0/24"]
}

# Создаем 2ой вебсервер в зоне b и той же сети, создайем подсеть (sudnet3)
resource "yandex_compute_instance" "Nginx2"{
  name        = "nginx2-vm"
  platform_id = "standard-v3"
  zone        = "ru-central1-b"

  resources {
    cores  = "2"
    memory = "6"
  }

  boot_disk {
    initialize_params {
      image_id = "fd861l8ckd35g2rrhsfd"
      size =  20
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.subnet3.id
    nat        = true
    ipv4       = true
  }

  metadata = {
  user-data = "${file("./meta.txt")}"
  }
}

resource "yandex_vpc_subnet" "subnet3" {
  name           = "subnet3"
  zone           = "ru-central1-b"
  network_id     = yandex_vpc_network.network1.id
  v4_cidr_blocks = ["172.16.0.0/24"]
}

# Создаем Elastick в зоне b и той же сети, в подсети (subnet3)
resource "yandex_compute_instance" "Elastick" {
  name        = "elastick-vm"
  platform_id = "standard-v3"
  zone        = "ru-central1-b"

 resources {
    cores  = "4"
    memory = "12"
  }

boot_disk {
    initialize_params {
      image_id = "fd861l8ckd35g2rrhsfd"
      size =  60
    }
  }

network_interface {
    subnet_id = yandex_vpc_subnet.subnet3.id
     nat        = true
     ipv4       = true
  }

  metadata = {
    user-data = "${file("./meta.txt")}"
   }
 }

#Создаем HTTP router и виртуальный хост для управления 
resource "yandex_alb_http_router" "tf-router" {
  name          = "my-http-router"
  folder_id     = "b1g6v0h3tgtvb9189u9c"

 labels = {
    tf-label    = "tf-label-value"
    empty-label = ""
  }
}

#В виртуальном хосте указываем идентификатор роутера, к которому он принадлежит.Указываем серверную группу для маршрути>
resource "yandex_alb_virtual_host" "my-virtual-host" {
  name           = "my-virtual-host"
  http_router_id = "${yandex_alb_http_router.tf-router.id}"
  route {
    name = "my-route"
    http_route {
      http_route_action {
        backend_group_id = "${yandex_alb_backend_group.backend-group.id}"
        timeout          = "15s"
      }
    }
  }
}

#Создаем группу и помещаем в нее 2 веб сервера 
resource "yandex_alb_target_group" "target-group" {
  name           = "my-target-group"
  target {
    subnet_id    = "${yandex_vpc_subnet.subnet2.id}"
    ip_address   = "${yandex_compute_instance.Nginx1.network_interface.0.ip_address}"
  }

  target {
    subnet_id    = "${yandex_vpc_subnet.subnet3.id}"
    ip_address   = "${yandex_compute_instance.Nginx2.network_interface.0.ip_address}"
  }
}

#Создаем серверную группу и настраиваем ее на группу с веб серверами 
resource "yandex_alb_backend_group" "backend-group" {
  name      = "my-backend-group"
  http_backend {
    name = "http-backend"
    weight = 1
    port = 80
    target_group_ids = ["${yandex_alb_target_group.target-group.id}"]

    load_balancing_config {
      panic_threshold = 50
    }

    healthcheck {
      timeout = "50s"
      interval = "3s"
      http_healthcheck {
       path  = "/"
      }
    }
    http2 = "true"
  }
}

#Создаем балансировшик нагрузки и помешаем его в подсеть (subnet1 ее сделаем ниже).Указываем маршрут через роутер созда>
resource "yandex_alb_load_balancer" "balancer" {
  name        = "balancer"
  network_id  = yandex_vpc_network.network1.id
  allocation_policy {
    location {
      zone_id   = "ru-central1-a"
      subnet_id = yandex_vpc_subnet.subnet1.id
    }
  }

  listener {
    name = "my-listener"
    endpoint {
    address {
        external_ipv4_address {
        }
      }
      ports = [ 80 ]
    }
    http {
      handler {
        http_router_id = "${yandex_alb_http_router.tf-router.id}"
      }
    }
  }

#Создаем заббик и подсеть (subnet1), в которую уже поместили балансировщик и туда определяем заббикс
resource "yandex_compute_instance" "Zabbix" {
  name        = "zabbix-vm"
  platform_id = "standard-v3"
  zone        = "ru-central1-a"

  resources {
    cores  = "2"
    memory = "6"
  }

  boot_disk {
    initialize_params {
      image_id = "fd861l8ckd35g2rrhsfd"
      size = 20
    }
 }

  network_interface {
    subnet_id = yandex_vpc_subnet.subnet1.id
    nat        = true
    ipv4       = true
  }

  metadata = {
user-data = "${file("./meta.txt")}"
  }
}

resource "yandex_vpc_subnet" "subnet1" {
  name           = "subnet1"
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.network1.id
  v4_cidr_blocks = ["192.168.0.0/24"]
}

output "internal_ip_address_Zabbix" {
  value = yandex_compute_instance.Zabbix.network_interface.0.ip_address
}
output "external_ip_address_Zabbix" {
  value = yandex_compute_instance.Zabbix.network_interface.0.nat_ip_address
}

# Создаем Kibana и определяем ее в подсеть (subnet1)
resource "yandex_compute_instance" "Kibana" {
  name        = "kibana-vm"
  platform_id = "standard-v3"
  zone        = "ru-central1-a"

  resources {
    cores  = "2"
    memory = "8"
}

  boot_disk {
    initialize_params {
      image_id = "fd861l8ckd35g2rrhsfd"
      size = 20
    }
 }

  network_interface {
    subnet_id = yandex_vpc_subnet.subnet1.id
    nat        = true
    ipv4       = true
}

  metadata = {
    user-data = "${file("./meta.txt")}"
  }
}

output "internal_ip_address_Kibana" {
  value = yandex_compute_instance.Kibana.network_interface.0.ip_address
}
output "external_ip_address_Kibana" {
  value = yandex_compute_instance.Kibana.network_interface.0.nat_ip_address
}

# Создаем группы безопасности 
# Группа для веб серверов 
resource "yandex_vpc_security_group" "sg1" {
  name        = "security group"
  description = "Description for security group"
  network_id  = "${yandex_vpc_network.network1.id}"
# Правила для балансировщика нагрузки
  ingress {
    protocol       = "HTTP"
    description    = "Rule description 1"
    v4_cidr_blocks = ["192.168.0.0/24"]
    port           = "80"
  }
  egress {
    protocol       = "HTTP"
    description    = "Rule description 2"
    v4_cidr_blocks = ["192.168.0.0/24"]
    port           = "80"
  }
#Правила для забикса
  ingress {
    protocol       = "TCP"
    description    = "Rule description 3"
    v4_cidr_blocks = ["192.168.0.0/24"]
    port           = "10050"
 }
  egress {
    protocol       = "TCP"
    description    = "Rule description 4"
    v4_cidr_blocks = ["192.168.0.0/24"]
    port           = "10050"
 }
# Правило для Filebit и Elactick
  egress {
    protocol       = "TCP"
    description    = "Rule description 5"
    v4_cidr_blocks = ["172.16.0.0/24"]
    port           = "9200"
 }
# Правила для SSH (потом укажу подсеть из которой буду рулить ансеблом)
  ingress {
    protocol       = "TCP"
    description    = "Rule description 6"
    v4_cidr_blocks = ["0.0.0.0/0""], ["192.168.0.0/0"]
    port           = "22"
  }
}
# Создаем группу для Elastick
resource "yandex_vpc_security_group" "sg2" {
  name        = "security group"
  description = "Description for security group"
  network_id  = "${yandex_vpc_network.network1.id}"
# Правила для Kibana
  ingress {
    protocol       = "HTTP"
    description    = "Rule description 1"
    v4_cidr_blocks = ["192.168.0.0/24"]
    port           = "5601"
  }
  egress {
    protocol       = "TCP"
    description    = "Rule description 2"
    v4_cidr_blocks = ["192.168.0.0/24"]
    port           = "5601"
 }
#Правила для забикса
  ingress {
    protocol       = "TCP"
    description    = "Rule description 3"
    v4_cidr_blocks = ["192.168.0.0/24"]
    port           = 10050
 }
  egress {
    protocol       = "TCP"
    description    = "Rule description 4"
    v4_cidr_blocks = ["192.168.0.0/24"]
    port           = "10050"
 }
# Правила для Filebit
  ingress {
    protocol       = "TCP"
    description    = "Rule description 5"
    v4_cidr_blocks = ["172.16.0.0/24"], ["10.0.2.0/24"]
    from_port      = "9200"
    to_port        = "9400"
 }
  egress {
    protocol       = "TCP"
    description    = "Rule description 6"
    v4_cidr_blocks = ["172.16.0.0/24"], ["10.0.2.0/24"]
    from_ port     = "9200"
    to_port        = "9400"
 }
# Правило SSH (потом укажу подсеть из которой буду рулить ансеблом)
  ingress {
    protocol       = "TCP"
    description    = "Rule description 7"
    v4_cidr_blocks = ["0.0.0.0/0"], ["192.168.0.0/0"]
    port           = "22"
  }
}
# Создаем группу для BastionVM (потом укажу подсеть из которой буду рулить ансеблом)
resource "yandex_vpc_security_group" "sg3" {
  name        = "security group"
  description = "Description for security group"
  network_id  = "${yandex_vpc_network.network1.id}"

  ingress {
    protocol       = "TCP"
    description    = "Rule description 1"
    v4_cidr_blocks = ["0.0.0.0/0"], ["172.16.0.0/24"], ["10.0.2.0/24"], ["192.168.0.0/0"]
    port           = "22"
  }
  egress {
    protocol       = "TCP"
    description    = "Rule description 2"
    v4_cidr_blocks = ["0.0.0.0/0""], ["172.16.0.0/24"], ["10.0.2.0/24"], ["192.168.0.0/0"]
    port           = "22"
  }
}

















