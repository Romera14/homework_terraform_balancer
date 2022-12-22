# Домашнее задание к занятию "`10.7 Отказоустойчивость в облаке`" - `Паромов Роман`
## Задание 1
1) playbook terraform
```
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
  required_version = ">= 0.13"
}

provider "yandex" {
  token = "y0_AgAAAAAWRmE_AATuwQAAAADRep88EndWCV6zS5yJ5-q_9_Sc4_DFR7I"
#  cloud_id  = var.cloud_id
  folder_id = "b1g817nmob937losobc1"
  zone      = "ru-central1-a"
}

resource "yandex_compute_instance" "server" {
  count = 2
  name = "server${count.index}"

  resources {
    cores  = 2
    memory = 2
    core_fraction = 20
  }

  boot_disk {
    initialize_params {
      image_id = "fd8m8s42796gm6v7sf8e"
      size = 3
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    nat       = true
  }
  
  metadata = {
    user-data = "${file("/home/paromov/terraform_nginx/meta.txt")}"
  }

}

resource "yandex_vpc_network" "network-1" {
  name = "network1"
}

resource "yandex_vpc_subnet" "subnet-1" {
  name           = "subnet1"
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.network-1.id
  v4_cidr_blocks = ["192.168.10.0/24"]
}

resource "yandex_lb_target_group" "test" {
  name           = "my-target-group"

  target {
    subnet_id    = "${yandex_vpc_subnet.subnet-1.id}"
    address   = "${yandex_compute_instance.server[0].network_interface.0.ip_address}"
  }
  target {
    subnet_id    = "${yandex_vpc_subnet.subnet-1.id}"
    address   = "${yandex_compute_instance.server[1].network_interface.0.ip_address}"
  }
}

resource "yandex_lb_network_load_balancer" "load_balancer" {
  name = "load-balancer"
  listener {
    name = "my-listener"
    port = 80
  }
  attached_target_group {
    target_group_id = "${yandex_lb_target_group.test.id}"
    healthcheck {
      name = "http"
        http_options {
          port = 80
          path = "/"
        }
    }
  }
}

output "server_name" {
  value = yandex_compute_instance.server[*].name
}

output "internal_ip_address_server" {
  value = yandex_compute_instance.server[*].network_interface.0.ip_address
}
output "external_ip_address_server" {
  value = yandex_compute_instance.server[*].network_interface.0.nat_ip_address
}
```
playbook ansible
```
- name: nginx
  become: true
  hosts: servers
  tasks:
  - name: install nginx
    apt:
      name:
        - nginx
      state: present
      update_cache: yes
  - name: start nginx
    ansible.builtin.service:
      name: nginx
      state: started
      enabled: yes
```
![2](https://github.com/Romera14/homework_terraform_balancer/blob/main/Снимок%20экрана%202022-12-22%20в%2021.14.45.png)
![3](https://github.com/Romera14/homework_terraform_balancer/blob/main/Снимок%20экрана%202022-12-22%20в%2021.13.44.png)

