terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
}
provider "yandex" {
token     = "y0_AgAAAABQCI1oAATuwQAAAADVEaOUnXse39KKR-GsyFwp6C6JQsbWfYs"
cloud_id  = "b1gdqrsimkog95ftf7a6"
folder_id = "b1g6v0h3tgtvb9189u9c"
zone      = "ru-central1-c"
}

# Nginx1 vm zona "c"

resource "yandex_compute_instance" "vm-1" {
  name        = "Nginx1-vm"
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
    subnet_id = yandex_vpc_subnet.subnet1.id
    nat       = true
 # ip_address = 192.168.0.10/24
 # ipv4      = true

  }

  metadata = {
    user-data = "${file("./meta.txt")}"
  }
}

resource "yandex_vpc_network" "network1" {
  name = "network1"
}
resource "yandex_vpc_subnet" "subnet1" {
  name           = "subnet1"
  zone           = "ru-central1-c"
  network_id     = yandex_vpc_network.network1.id
  v4_cidr_blocks = ["192.168.0.0/24"]
}
output "internal_ip_address_vm_1" {
  value = yandex_compute_instance.vm-1.network_interface.0.ip_address
}
output "external_ip_address_vm_1" {
  value = yandex_compute_instance.vm-1.network_interface.0.nat_ip_address
}

# Nginx2 vm zona "a"

resource "yandex_compute_instance" "vm-2" {
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
   # ip_address = 192.168.0.20/24
   # ipv4      = true
  }
  
metadata = {
    user-data = "${file("./meta.txt")}"
  }
}

resource "yandex_vpc_network" "network2" {
name = "network2"
}

resource "yandex_vpc_subnet" "subnet2" {
  name           = "subnet2"
  zone           = "ru-central1-b"
  network_id     = yandex_vpc_network.network2.id
  v4_cidr_blocks = ["192.168.0.0/24"]
}

output "internal_ip_address_vm_2" {
  value = yandex_compute_instance.vm-2.network_interface.0.ip_address
}
output "external_ip_address_vm_2" {
  value = yandex_compute_instance.vm-2.network_interface.0.nat_ip_address
}

# HTTP router

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

# Target group

resource "yandex_alb_target_group" "target-group" {
  name           = "my-target-group"
  
  target {
    subnet_id    = "${yandex_vpc_subnet.subnet1.id}"
    ip_address   = "${yandex_compute_instance.vm-1.network_interface.0.ip_address}"
  }
  
  target {
    subnet_id    = "${yandex_vpc_subnet.subnet2.id}"
    ip_address   = "${yandex_compute_instance.vm-2.network_interface.0.ip_address}"
  }
}

# Backend Group

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

# Load balancer









