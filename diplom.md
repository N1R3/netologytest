```bash
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
}

provider "yandex" {
token     = "сcloud key"
cloud_id  = "b1gdqrsimkog95ftf7a6"
folder_id = "b1g6v0h3tgtvb9189u9c"
zone      = "ru-central1-c"
}



### Nginx1 vm zone "c"

resource "yandex_compute_instance" "Nginx1" {
  name        = "Nginx1"
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
    nat       = true
    #ip_address = "192.168.0.10/24"
    ipv4      = true
  }

  metadata = {
    user-data = "${file("./meta.txt")}"
  }
 }

resource "yandex_vpc_network" "network1" {
  name = "network1"
}

resource "yandex_vpc_subnet" "subnet2" {
  name           = "subnet2"
  zone           = "ru-central1-c"
  network_id     = yandex_vpc_network.network1.id
  v4_cidr_blocks = ["10.2.0.0/24"]
}
#output "internal_ip_address_vm_1" {
#  value = yandex_compute_instance.vm-1.network_interface.0.ip_address
#  value = yandex_compute_instance.vm-1.network_interface.0.nat_ip_address
#}

### Nginx2 vm zone "b"

resource "yandex_compute_instance" "Nginx2"{
  name        = "Nginx2-vm"
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
    subnet_id = yandex_vpc_subnet.subnet2.id
    nat       = true
    #ip_address = "192.168.0.20/24"
    ipv4      = true
  }

  metadata = {
  user-data = "${file("./meta.txt")}"
  }
}

#output "internal_ip_address_vm_2" {
#  value = yandex_compute_instance.vm-2.network_interface.0.ip_address
#}
#output "external_ip_address_vm_2" {
#  value = yandex_compute_instance.vm-2.network_interface.0.nat_ip_address
#}

### Elastick

resource "yandex_compute_instance" "Elastick" {
  name        = "Elastick-vm"
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
    subnet_id = yandex_vpc_subnet.subnet2.id
     nat       = true
     #ip_address = "192.168.0.30/24"
     ipv4      = true
  }

  metadata = {
    user-data = "${file("./meta.txt")}"
   }
 }

### HTTP router

resource "yandex_alb_http_router" "tf-router" {
  name          = "my-http-router"
  folder_id     = "b1g6v0h3tgtvb9189u9c"

 labels = {
    tf-label    = "tf-label-value"
    empty-label = ""
  }
}

resource "yandex_alb_virtual_host" "my-virtual-host" {
  name           = "my-virtual-host"
  http_router_id = "${yandex_alb_http_router.tf-router.id}"

  route {
    name = "my-route"
    http_route {
      http_route_action {
        backend_group_id = "${yandex_alb_backend_group.backend-group.id}"
        timeout          = "3s"
      }
    }
  }
}

### Target group

resource "yandex_alb_target_group" "target-group" {
  name           = "my-target-group"
  target {
    subnet_id    = "${yandex_vpc_subnet.subnet2.id}"
    ip_address   = "${yandex_compute_instance.Nginx1.network_interface.0.ip_address}"
  }

  target {
    subnet_id    = "${yandex_vpc_subnet.subnet2.id}"
    ip_address   = "${yandex_compute_instance.Nginx2.network_interface.0.ip_address}"
  }
}

### Backend Group

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
      timeout = "3s"
      interval = "3s"
      http_healthcheck {
       path  = "/"
      }
    }
    http2 = "true"
  }
}

### Load balancer

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

  log_options {
    log_group_id = ""
    discard_rule {
      http_codes          = ["200"]
      #http_code_intervals = ["2XX"]
      #grpc_codes          = ["GRPC_OK"]
      discard_percent     = 50
    }
  }
}

### Zabbix

resource "yandex_compute_instance" "Zabbix" {
  name        = "Zabbix-vm"
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
    nat       = true
    ip_address = "192.168.0.50/24"
    ipv4      = true
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

### Kibana

resource "yandex_compute_instance" "Kibana" {
  name        = "Kibana-vm"
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
    nat       = true
    ip_address = "192.168.0.51/24"
    ipv4      = true
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









