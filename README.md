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
  token = ""
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

## Задание 2
Столнулся с тем что во время команды terraform destroy сначала удаляются права у сервисного аккаунта, по этому он не может удалить группу машин, не нашел как с этим бороться. Приходилось после команды terraform destroy через web интерфейс вручную снова добавлять права аккаунту. Также не нашел как создать балансировщик для группы машин и делал это вручную через web интерфейс

playbook terraform
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
  token = ""
#  cloud_id  = var.cloud_id
  folder_id = "b1g817nmob937losobc1"
  zone      = "ru-central1-a"
}


resource "yandex_iam_service_account" "test" {
  name        = "test"
  folder_id = "b1g817nmob937losobc1"
  description = "service account to manage IG"
}

resource "yandex_resourcemanager_folder_iam_binding" "editor" {
  folder_id = "b1g817nmob937losobc1"
  role      = "editor"
  members   = [
    "serviceAccount:${yandex_iam_service_account.test.id}",
  ]
}

resource "yandex_compute_instance_group" "ig-1" {
  name               = "fixed-ig-with-balancer"
  folder_id          = "b1g817nmob937losobc1"
  service_account_id = "${yandex_iam_service_account.test.id}"
  instance_template {
    platform_id = "standard-v3"
    resources {
      memory = 2
      cores  = 2
    }

    boot_disk {
      mode = "READ_WRITE"
      initialize_params {
        image_id = "fd8m8s42796gm6v7sf8e"
      }
    }

    network_interface {
      network_id = "${yandex_vpc_network.network-1.id}"
      subnet_ids = ["${yandex_vpc_subnet.subnet-1.id}"]
      nat = true
    }

    metadata = {
      ssh-keys = "test:ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDuMPfujkHgJ4IKwI580dgIO5dCO+60S952RQ/rPt3+ZAMutQcmuqxAt1+5ot5LNUecmZIIHs1yPGaO8UKs87n4Ww9VAgs4T5iFzF5w0dPj1ldozX9Yv0rEwcy35GKfQpV74omNMQRaGn2VuisnhuYKaoXFErlXnR7cxt0ebKnZWz02rCQtVhKNXnNbSfBq2gihtoXZe8/DBplagTBCF0yMSoM3UfboGqIrLXNuEbmVfPGAEViW95S+AjSpfSHWoJ2Kbsg+elqGx9LCziRMP6nM0MR2vz+aqBBPsRYNZmEkN6FanxhzHerU942vxQpR+N1+Wa5Zm4vyvD5GmTL4hW9GNC16HTvceRdUNNHcOgWvMDEVWWOEAjgckMm2kgQIJi16vz5XMNQf2SHvI7VF7xhAfSUW1Qg+tMZQ+iw/xp28y95Er5XijDKmzKRXGGD7K7smv2BtYJyRWeRJFcProY/CgP0UWKU+46oEBcOTny60dhHtYgYoyAGc72ak+wy1F9U= paromov@debian11"
      user-data = "${file("/home/paromov/terraform_nginx2/metadata.yaml")}"
    }
  }

  scale_policy {
    fixed_scale {
      size = 2
    }
  }

  allocation_policy {
    zones = ["ru-central1-a"]
  }

  deploy_policy {
    max_unavailable = 1
    max_expansion   = 0
  }

  load_balancer {
    target_group_name        = "target-group"
    target_group_description = "load balancer target group"
  }
}

resource "yandex_vpc_network" "network-1" {
  name = "network1"
}

resource "yandex_vpc_subnet" "subnet-1" {
  name           = "subnet1"
  zone           = "ru-central1-a"
  network_id     = "${yandex_vpc_network.network-1.id}"
  v4_cidr_blocks = ["192.168.10.0/24"]
}
resource "yandex_lb_network_load_balancer" "load_balancer" {
  name = "load-balancer"
  listener {
    name = "my-listener"
    port = 80
  }
  attached_target_group {
    target_group_id = ??????
    healthcheck {
      name = "http"
        http_options {
          port = 80
          path = "/"
        }
    }
  }
}
```
![](https://github.com/Romera14/homework_terraform_balancer/blob/main/Снимок%20экрана%202022-12-28%20в%2021.17.08.png)
![](https://github.com/Romera14/homework_terraform_balancer/blob/main/Снимок%20экрана%202022-12-28%20в%2021.17.55.png)
![](https://github.com/Romera14/homework_terraform_balancer/blob/main/Снимок%20экрана%202022-12-28%20в%2021.21.04.png)
