# terraform-end-src-web
За основу было взято моё решение для ДЗ **ter-homeworks-05**
Был добавлен модуль **container_registry**, в котором создаётся хранилище образов контейнеров и даются необходимые права:
```
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
  required_version = ">1.12.0"
а}

resource "yandex_container_registry" "registry" {
  name      = var.name
  folder_id = var.folder_id

  labels = var.labels
}

resource "yandex_container_registry_iam_binding" "puller" {
  count = length(var.role) > 0 ? 1 : 0

  registry_id = yandex_container_registry.registry.id
  role        = var.role

  members = var.members
}
```
Через iam_binding даём права на pull из репозитория для всех пользователей, чтобы в ВМ можно было скачать образ без авторизации
В рут модуле был изменён data с формированием cloud-init:
```
data "template_file" "cloudinit" {
  template = file("./cloud-init.yml")

  vars     = {
    ssh-pub-key = local.ssh-pub-key,
    packages    = jsonencode(var.vm_cloud_init_packages)
    user        = var.vm_user
    env_content = indent(6, local.env_content)
  }
}
```
В него была добавлена переменная env_content, в которую помещаются переменные для запуска **compose** приложения. Переменная local.env_content выглядит следующим образом:
```
env_content = templatefile("${path.module}/env.template", {
    image       = "cr.yandex/${module.container_registry.id}/${var.docker_image_name}"
    db_host     = module.mysql_cluster.host[0].fqdn
    db_password = local.db_password
    db_user     = var.mysql_cluster_db_username
    db_name     = var.mysql_cluster_db_name
  })
```
Здесь подробнее:
1) После создания container registry заполняется переменная image, к которой обращается композ для того, чтобы скачать образ, залитый в container_registry
2) db_host - DNS имя хоста СУБД для обращения к нему приложения. Формируется после создания кластера
3) db_password - Пароль, который будет передан в env для подключения к БД. Я решил сформировать его из трёх переменных, а именно:
```
lockbox_password = try(data.yandex_lockbox_secret_version.db_password[0].entries[0].text_value, null)
generic_password = length(var.manual_db_password) > 0 ? var.manual_db_password : random_password.db_password.result
db_password = length(local.lockbox_password) > 0 ? local.lockbox_password : local.generic_password
```
Если в lockbox создан секрет(его код будет далее), то в переменную db_password попадает он, иначе проверяется наличие переменной var.manual_db_password(заполняется вручную в personal.auto.tfvars), если такой переменной нету то используется значение, сгенерированное через **random_password**. Таким образом гарантируется, что БД будет присвоен пароль LockBox -> var.manual_db_password -> random_password.db_password.
4) db_user - берётся из переменной, которая в т.ч. передаётся для создания пользователя для БД
5) db_name - берётся из переменной, которая в т.ч. передаётся для создания БД
Далее LockBox. Я решил, что будет некорректно создавать секрет и версию из терраформа, а потом использовать его же в терраформе, поэтому секрет заранее создаётся в веб-интерфейсе, а **terraform** просто обращается к нему, чтобы забрать креды
```
data "yandex_lockbox_secret_version" "db_password" {
  count      = length(var.lockbox_db_secret_id) > 0 ? 1 : 0

  secret_id  = var.lockbox_db_secret_id
  version_id = var.lockbox_db_secret_version
}
```
Причём значения из него берутся только в том случае, если мы в переменных указали значение lockbox_db_secret_id, иначе используем random_password или manual_db_password(если указано)
env.template, в который подставляются указанные выше переменные выглядит так:
```
WEB_IMAGE=${image}
DB_HOST=${db_host}
MYSQL_USER=${db_user}
MYSQL_PASSWORD=${db_password}
MYSQL_DATABASE=${db_name}
```
cloud-init выглядит следующим образом
```
package_update: true
package_upgrade: false
packages: ${packages}
users:
  - name: ${user}
    groups: sudo, docker
    shell: /bin/bash
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    ssh_authorized_keys:
      - ${ssh-pub-key}
write_files:
  - path: /opt/web/.env
    permissions: '0600'
    owner: ${user}:${user}
    defer: true
    content: |
      ${env_content}
runcmd:
  - git clone https://github.com/IGrishunin/terraform-end-src-web.git /tmp/temp
  - cp -ra /tmp/temp/* /opt/web/
  - rm -rf /tmp/temp
  - chown -R ${user}:${user} /opt/web
```
В нём указана установка пакетов, прописанных в переменной packages(docker, docker-compose-v2, git)
Создаётся пользователь ubuntu(указывается через переменную user), а также этот пользователь сразу добавляется в группу docker для управления контейнерами
Создаётся файл .env в рабочем каталоге, в котором уже есть необходимые переменные для запуска проекта
Копируется репозиторий с композ файлами в рабочий каталог для запуска приложения.

# Инструкция по запуску проекта:
Для запуска проекта необходимо следующее:
1) Склонировать репозиторий https://github.com/IGrishunin/ter-homeworks-05.git и зайти в ветку **terraform-end**
2) Создать в домашнем каталоге пользователя, от имени которого будет запускаться проект ~/.aws/credentials и заполнить его(aws.credentials.example лежит в проекте)
3) Заполнить personal.auto.tfvars(в репозитории лежит personal.auto.tfvars.example)
4) Выполнить ```terraform init``` -> ```terraform plan``` -> ```terraform apply```
5) После того, как выполнится ```terraform apply``` на экран выведется output, в котором будет указано куда нужно запушить собранный docker образ
6) Клонируем репозиторий https://github.com/IGrishunin/terraform-end-src-web.git
7) Собираем образ командой ```docker build -t <output> .``` находясь в каталоге склонированного репозитория
8) Выполняем ```docker push <output>```
9) Подключаемся к ВМ по ssh, заходим в **/opt/web/** и выполняем ```docker compose up -d```
10) После этого приложение будет доступно по http://<public vm ip>

# Выполнение
```
mfc-admin@nb-grishunin:~/NETOLOGY/ter-homeworks-05$ terraform plan
data.yandex_lockbox_secret_version.db_password[0]: Reading...
module.web-vm.data.yandex_compute_image.my_image: Reading...
data.yandex_lockbox_secret_version.db_password[0]: Read complete after 0s [id=e6qent78vn29vcqskk67]
module.web-vm.data.yandex_compute_image.my_image: Read complete after 0s [id=fd8jag83fdegioetshlr]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create
 <= read (data resources)

Terraform will perform the following actions:

  # data.template_file.cloudinit will be read during apply
  # (config refers to values not yet known)
 <= data "template_file" "cloudinit" {
      + id       = (known after apply)
      + rendered = (known after apply)
      + template = <<-EOT
            #cloud-config
            package_update: true
            package_upgrade: false
            packages: ${packages}
            users:
              - name: ${user}
                groups: sudo, docker
                shell: /bin/bash
                sudo: ["ALL=(ALL) NOPASSWD:ALL"]
                ssh_authorized_keys:
                  - ${ssh-pub-key}
            write_files:
              - path: /opt/web/.env
                permissions: '0600'
                owner: ${user}:${user}
                defer: true
                content: |
                  ${env_content}
            runcmd:
              - git clone https://github.com/IGrishunin/terraform-end-src-web.git /tmp/temp
              - cp -ra /tmp/temp/* /opt/web/
              - rm -rf /tmp/temp
              - chown -R ${user}:${user} /opt/web
        EOT
      + vars     = {
          + "env_content" = (sensitive value)
          + "packages"    = jsonencode(
                [
                  + "docker",
                  + "docker-compose-v2",
                  + "git",
                ]
            )
          + "ssh-pub-key" = <<-EOT
                ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOmRn0dsEASZhy3tV6ZMLvffGNmZGXgdpT9+DMIAhPJb mfc-admin@nb-grishunin.mfcl.mfclnr.ru
            EOT
          + "user"        = "ubuntu"
        }
    }

  # random_password.db_password will be created
  + resource "random_password" "db_password" {
      + bcrypt_hash      = (sensitive value)
      + id               = (known after apply)
      + length           = 16
      + lower            = true
      + min_lower        = 0
      + min_numeric      = 0
      + min_special      = 0
      + min_upper        = 0
      + number           = true
      + numeric          = true
      + result           = (sensitive value)
      + special          = true
      + upper            = true
        # (1 unchanged attribute hidden)
    }

  # module.container_registry.yandex_container_registry.registry will be created
  + resource "yandex_container_registry" "registry" {
      + created_at  = (known after apply)
      + folder_id   = "b1gjeu5fo05f0jh5mfug"
      + id          = (known after apply)
      + labels      = {
          + "environment" = "dev"
        }
      + name        = "registry-develop"
      + registry_id = (known after apply)
      + status      = (known after apply)
    }

  # module.container_registry.yandex_container_registry_iam_binding.puller[0] will be created
  + resource "yandex_container_registry_iam_binding" "puller" {
      + id          = (known after apply)
      + members     = [
          + "system:allUsers",
        ]
      + registry_id = (known after apply)
      + role        = "container-registry.images.puller"
    }

  # module.mysql_cluster.yandex_mdb_mysql_cluster.cluster will be created
  + resource "yandex_mdb_mysql_cluster" "cluster" {
      + allow_regeneration_host   = false
      + backup_retain_period_days = (known after apply)
      + created_at                = (known after apply)
      + deletion_protection       = (known after apply)
      + disk_encryption_key_id    = (known after apply)
      + environment               = "PRESTABLE"
      + folder_id                 = (known after apply)
      + health                    = (known after apply)
      + host_group_ids            = (known after apply)
      + id                        = (known after apply)
      + mysql_config              = (known after apply)
      + name                      = "cluster_web"
      + network_id                = (known after apply)
      + security_group_ids        = (known after apply)
      + status                    = (known after apply)
      + version                   = "8.0"

      + access (known after apply)

      + backup_window_start (known after apply)

      + connection_manager (known after apply)

      + disk_size_autoscaling (known after apply)

      + host {
          + assign_public_ip   = true
          + fqdn               = (known after apply)
          + replication_source = (known after apply)
          + subnet_id          = (known after apply)
          + zone               = "ru-central1-d"
        }

      + maintenance_window (known after apply)

      + performance_diagnostics (known after apply)

      + resources {
          + disk_size          = 10
          + disk_type_id       = "network-hdd"
          + resource_preset_id = "b2.medium"
        }
    }

  # module.mysql_cluster_content.yandex_mdb_mysql_database.db will be created
  + resource "yandex_mdb_mysql_database" "db" {
      + cluster_id = (known after apply)
      + id         = (known after apply)
      + name       = "db_web"
    }

  # module.mysql_cluster_content.yandex_mdb_mysql_user.user will be created
  + resource "yandex_mdb_mysql_user" "user" {
      + authentication_plugin = (known after apply)
      + cluster_id            = (known after apply)
      + connection_manager    = (known after apply)
      + generate_password     = false
      + id                    = (known after apply)
      + name                  = "appweb"
      + password              = (sensitive value)
      + password_wo           = (write-only attribute)

      + connection_limits (known after apply)

      + permission {
          + database_name = "db_web"
          + roles         = [
              + "ALL",
            ]
        }

      + user_connection_manager (known after apply)
    }

  # module.vpc.yandex_vpc_network.network will be created
  + resource "yandex_vpc_network" "network" {
      + created_at                = (known after apply)
      + default_security_group_id = (known after apply)
      + folder_id                 = (known after apply)
      + id                        = (known after apply)
      + labels                    = (known after apply)
      + name                      = "develop"
      + subnet_ids                = (known after apply)
    }

  # module.vpc.yandex_vpc_security_group.security["sec-database"] will be created
  + resource "yandex_vpc_security_group" "security" {
      + created_at = (known after apply)
      + folder_id  = "b1gjeu5fo05f0jh5mfug"
      + id         = (known after apply)
      + labels     = (known after apply)
      + name       = "sec-database"
      + network_id = (known after apply)
      + status     = (known after apply)

      + egress {
          + description       = "разрешить весь исходящий трафик"
          + from_port         = 0
          + id                = (known after apply)
          + labels            = (known after apply)
          + port              = -1
          + protocol          = "TCP"
          + to_port           = 65365
          + v4_cidr_blocks    = [
              + "0.0.0.0/0",
            ]
          + v6_cidr_blocks    = []
            # (2 unchanged attributes hidden)
        }

      + ingress {
          + description       = "разрешить входящий ssh"
          + from_port         = -1
          + id                = (known after apply)
          + labels            = (known after apply)
          + port              = 22
          + protocol          = "TCP"
          + to_port           = -1
          + v4_cidr_blocks    = [
              + "0.0.0.0/0",
            ]
          + v6_cidr_blocks    = []
            # (2 unchanged attributes hidden)
        }
      + ingress {
          + description       = "разрешить входящий трафик на подключение к БД 3306"
          + from_port         = -1
          + id                = (known after apply)
          + labels            = (known after apply)
          + port              = 3306
          + protocol          = "TCP"
          + to_port           = -1
          + v4_cidr_blocks    = [
              + "0.0.0.0/0",
            ]
          + v6_cidr_blocks    = []
            # (2 unchanged attributes hidden)
        }
    }

  # module.vpc.yandex_vpc_security_group.security["sec-web"] will be created
  + resource "yandex_vpc_security_group" "security" {
      + created_at = (known after apply)
      + folder_id  = "b1gjeu5fo05f0jh5mfug"
      + id         = (known after apply)
      + labels     = (known after apply)
      + name       = "sec-web"
      + network_id = (known after apply)
      + status     = (known after apply)

      + egress {
          + description       = "разрешить весь исходящий трафик"
          + from_port         = 0
          + id                = (known after apply)
          + labels            = (known after apply)
          + port              = -1
          + protocol          = "TCP"
          + to_port           = 65365
          + v4_cidr_blocks    = [
              + "0.0.0.0/0",
            ]
          + v6_cidr_blocks    = []
            # (2 unchanged attributes hidden)
        }

      + ingress {
          + description       = "разрешить входящий ssh"
          + from_port         = -1
          + id                = (known after apply)
          + labels            = (known after apply)
          + port              = 22
          + protocol          = "TCP"
          + to_port           = -1
          + v4_cidr_blocks    = [
              + "0.0.0.0/0",
            ]
          + v6_cidr_blocks    = []
            # (2 unchanged attributes hidden)
        }
      + ingress {
          + description       = "разрешить входящий трафик на 443"
          + from_port         = -1
          + id                = (known after apply)
          + labels            = (known after apply)
          + port              = 443
          + protocol          = "TCP"
          + to_port           = -1
          + v4_cidr_blocks    = [
              + "0.0.0.0/0",
            ]
          + v6_cidr_blocks    = []
            # (2 unchanged attributes hidden)
        }
      + ingress {
          + description       = "разрешить входящий трафик на 80"
          + from_port         = -1
          + id                = (known after apply)
          + labels            = (known after apply)
          + port              = 80
          + protocol          = "TCP"
          + to_port           = -1
          + v4_cidr_blocks    = [
              + "0.0.0.0/0",
            ]
          + v6_cidr_blocks    = []
            # (2 unchanged attributes hidden)
        }
    }

  # module.vpc.yandex_vpc_subnet.subnet["0"] will be created
  + resource "yandex_vpc_subnet" "subnet" {
      + created_at     = (known after apply)
      + folder_id      = (known after apply)
      + id             = (known after apply)
      + labels         = (known after apply)
      + name           = "develop-0"
      + network_id     = (known after apply)
      + v4_cidr_blocks = [
          + "10.0.0.0/24",
        ]
      + v6_cidr_blocks = (known after apply)
      + zone           = "ru-central1-d"
    }

  # module.web-vm.yandex_compute_instance.vm[0] will be created
  + resource "yandex_compute_instance" "vm" {
      + allow_stopping_for_update = true
      + created_at                = (known after apply)
      + description               = "TODO: description; {{terraform yyy managed}}"
      + folder_id                 = (known after apply)
      + fqdn                      = (known after apply)
      + gpu_cluster_id            = (known after apply)
      + hardware_generation       = (known after apply)
      + hostname                  = "web-0"
      + id                        = (known after apply)
      + labels                    = {
          + "project" = "web"
        }
      + maintenance_grace_period  = (known after apply)
      + maintenance_policy        = (known after apply)
      + metadata                  = (known after apply)
      + name                      = "web-0"
      + network_acceleration_type = "standard"
      + platform_id               = "standard-v3"
      + reserved_instance_pool_id = (known after apply)
      + status                    = (known after apply)
      + zone                      = "ru-central1-d"

      + boot_disk {
          + auto_delete = true
          + device_name = (known after apply)
          + disk_id     = (known after apply)
          + mode        = (known after apply)

          + initialize_params {
              + block_size  = (known after apply)
              + description = (known after apply)
              + image_id    = "fd8jag83fdegioetshlr"
              + name        = (known after apply)
              + size        = 10
              + snapshot_id = (known after apply)
              + type        = "network-hdd"
            }
        }

      + metadata_options (known after apply)

      + network_interface {
          + index              = (known after apply)
          + ip_address         = (known after apply)
          + ipv4               = true
          + ipv6               = (known after apply)
          + ipv6_address       = (known after apply)
          + mac_address        = (known after apply)
          + nat                = true
          + nat_ip_address     = (known after apply)
          + nat_ip_version     = (known after apply)
          + security_group_ids = (known after apply)
          + subnet_id          = (known after apply)
        }

      + placement_policy (known after apply)

      + resources {
          + core_fraction = 20
          + cores         = 2
          + memory        = 1
        }

      + scheduling_policy {
          + preemptible = true
        }
    }

Plan: 11 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + container_registry_image_tag = (known after apply)

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.
mfc-admin@nb-grishunin:~/NETOLOGY/ter-homeworks-05$ terraform apply
module.web-vm.data.yandex_compute_image.my_image: Reading...
data.yandex_lockbox_secret_version.db_password[0]: Reading...
module.web-vm.data.yandex_compute_image.my_image: Read complete after 0s [id=fd8jag83fdegioetshlr]
data.yandex_lockbox_secret_version.db_password[0]: Read complete after 0s [id=e6qent78vn29vcqskk67]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create
 <= read (data resources)

Terraform will perform the following actions:

  # data.template_file.cloudinit will be read during apply
  # (config refers to values not yet known)
 <= data "template_file" "cloudinit" {
      + id       = (known after apply)
      + rendered = (known after apply)
      + template = <<-EOT
            #cloud-config
            package_update: true
            package_upgrade: false
            packages: ${packages}
            users:
              - name: ${user}
                groups: sudo, docker
                shell: /bin/bash
                sudo: ["ALL=(ALL) NOPASSWD:ALL"]
                ssh_authorized_keys:
                  - ${ssh-pub-key}
            write_files:
              - path: /opt/web/.env
                permissions: '0600'
                owner: ${user}:${user}
                defer: true
                content: |
                  ${env_content}
            runcmd:
              - git clone https://github.com/IGrishunin/terraform-end-src-web.git /tmp/temp
              - cp -ra /tmp/temp/* /opt/web/
              - rm -rf /tmp/temp
              - chown -R ${user}:${user} /opt/web
        EOT
      + vars     = {
          + "env_content" = (sensitive value)
          + "packages"    = jsonencode(
                [
                  + "docker",
                  + "docker-compose-v2",
                  + "git",
                ]
            )
          + "ssh-pub-key" = <<-EOT
                ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOmRn0dsEASZhy3tV6ZMLvffGNmZGXgdpT9+DMIAhPJb mfc-admin@nb-grishunin.mfcl.mfclnr.ru
            EOT
          + "user"        = "ubuntu"
        }
    }

  # random_password.db_password will be created
  + resource "random_password" "db_password" {
      + bcrypt_hash      = (sensitive value)
      + id               = (known after apply)
      + length           = 16
      + lower            = true
      + min_lower        = 0
      + min_numeric      = 0
      + min_special      = 0
      + min_upper        = 0
      + number           = true
      + numeric          = true
      + result           = (sensitive value)
      + special          = true
      + upper            = true
        # (1 unchanged attribute hidden)
    }

  # module.container_registry.yandex_container_registry.registry will be created
  + resource "yandex_container_registry" "registry" {
      + created_at  = (known after apply)
      + folder_id   = "b1gjeu5fo05f0jh5mfug"
      + id          = (known after apply)
      + labels      = {
          + "environment" = "dev"
        }
      + name        = "registry-develop"
      + registry_id = (known after apply)
      + status      = (known after apply)
    }

  # module.container_registry.yandex_container_registry_iam_binding.puller[0] will be created
  + resource "yandex_container_registry_iam_binding" "puller" {
      + id          = (known after apply)
      + members     = [
          + "system:allUsers",
        ]
      + registry_id = (known after apply)
      + role        = "container-registry.images.puller"
    }

  # module.mysql_cluster.yandex_mdb_mysql_cluster.cluster will be created
  + resource "yandex_mdb_mysql_cluster" "cluster" {
      + allow_regeneration_host   = false
      + backup_retain_period_days = (known after apply)
      + created_at                = (known after apply)
      + deletion_protection       = (known after apply)
      + disk_encryption_key_id    = (known after apply)
      + environment               = "PRESTABLE"
      + folder_id                 = (known after apply)
      + health                    = (known after apply)
      + host_group_ids            = (known after apply)
      + id                        = (known after apply)
      + mysql_config              = (known after apply)
      + name                      = "cluster_web"
      + network_id                = (known after apply)
      + security_group_ids        = (known after apply)
      + status                    = (known after apply)
      + version                   = "8.0"

      + access (known after apply)

      + backup_window_start (known after apply)

      + connection_manager (known after apply)

      + disk_size_autoscaling (known after apply)

      + host {
          + assign_public_ip   = true
          + fqdn               = (known after apply)
          + replication_source = (known after apply)
          + subnet_id          = (known after apply)
          + zone               = "ru-central1-d"
        }

      + maintenance_window (known after apply)

      + performance_diagnostics (known after apply)

      + resources {
          + disk_size          = 10
          + disk_type_id       = "network-hdd"
          + resource_preset_id = "b2.medium"
        }
    }

  # module.mysql_cluster_content.yandex_mdb_mysql_database.db will be created
  + resource "yandex_mdb_mysql_database" "db" {
      + cluster_id = (known after apply)
      + id         = (known after apply)
      + name       = "db_web"
    }

  # module.mysql_cluster_content.yandex_mdb_mysql_user.user will be created
  + resource "yandex_mdb_mysql_user" "user" {
      + authentication_plugin = (known after apply)
      + cluster_id            = (known after apply)
      + connection_manager    = (known after apply)
      + generate_password     = false
      + id                    = (known after apply)
      + name                  = "appweb"
      + password              = (sensitive value)
      + password_wo           = (write-only attribute)

      + connection_limits (known after apply)

      + permission {
          + database_name = "db_web"
          + roles         = [
              + "ALL",
            ]
        }

      + user_connection_manager (known after apply)
    }

  # module.vpc.yandex_vpc_network.network will be created
  + resource "yandex_vpc_network" "network" {
      + created_at                = (known after apply)
      + default_security_group_id = (known after apply)
      + folder_id                 = (known after apply)
      + id                        = (known after apply)
      + labels                    = (known after apply)
      + name                      = "develop"
      + subnet_ids                = (known after apply)
    }

  # module.vpc.yandex_vpc_security_group.security["sec-database"] will be created
  + resource "yandex_vpc_security_group" "security" {
      + created_at = (known after apply)
      + folder_id  = "b1gjeu5fo05f0jh5mfug"
      + id         = (known after apply)
      + labels     = (known after apply)
      + name       = "sec-database"
      + network_id = (known after apply)
      + status     = (known after apply)

      + egress {
          + description       = "разрешить весь исходящий трафик"
          + from_port         = 0
          + id                = (known after apply)
          + labels            = (known after apply)
          + port              = -1
          + protocol          = "TCP"
          + to_port           = 65365
          + v4_cidr_blocks    = [
              + "0.0.0.0/0",
            ]
          + v6_cidr_blocks    = []
            # (2 unchanged attributes hidden)
        }

      + ingress {
          + description       = "разрешить входящий ssh"
          + from_port         = -1
          + id                = (known after apply)
          + labels            = (known after apply)
          + port              = 22
          + protocol          = "TCP"
          + to_port           = -1
          + v4_cidr_blocks    = [
              + "0.0.0.0/0",
            ]
          + v6_cidr_blocks    = []
            # (2 unchanged attributes hidden)
        }
      + ingress {
          + description       = "разрешить входящий трафик на подключение к БД 3306"
          + from_port         = -1
          + id                = (known after apply)
          + labels            = (known after apply)
          + port              = 3306
          + protocol          = "TCP"
          + to_port           = -1
          + v4_cidr_blocks    = [
              + "0.0.0.0/0",
            ]
          + v6_cidr_blocks    = []
            # (2 unchanged attributes hidden)
        }
    }

  # module.vpc.yandex_vpc_security_group.security["sec-web"] will be created
  + resource "yandex_vpc_security_group" "security" {
      + created_at = (known after apply)
      + folder_id  = "b1gjeu5fo05f0jh5mfug"
      + id         = (known after apply)
      + labels     = (known after apply)
      + name       = "sec-web"
      + network_id = (known after apply)
      + status     = (known after apply)

      + egress {
          + description       = "разрешить весь исходящий трафик"
          + from_port         = 0
          + id                = (known after apply)
          + labels            = (known after apply)
          + port              = -1
          + protocol          = "TCP"
          + to_port           = 65365
          + v4_cidr_blocks    = [
              + "0.0.0.0/0",
            ]
          + v6_cidr_blocks    = []
            # (2 unchanged attributes hidden)
        }

      + ingress {
          + description       = "разрешить входящий ssh"
          + from_port         = -1
          + id                = (known after apply)
          + labels            = (known after apply)
          + port              = 22
          + protocol          = "TCP"
          + to_port           = -1
          + v4_cidr_blocks    = [
              + "0.0.0.0/0",
            ]
          + v6_cidr_blocks    = []
            # (2 unchanged attributes hidden)
        }
      + ingress {
          + description       = "разрешить входящий трафик на 443"
          + from_port         = -1
          + id                = (known after apply)
          + labels            = (known after apply)
          + port              = 443
          + protocol          = "TCP"
          + to_port           = -1
          + v4_cidr_blocks    = [
              + "0.0.0.0/0",
            ]
          + v6_cidr_blocks    = []
            # (2 unchanged attributes hidden)
        }
      + ingress {
          + description       = "разрешить входящий трафик на 80"
          + from_port         = -1
          + id                = (known after apply)
          + labels            = (known after apply)
          + port              = 80
          + protocol          = "TCP"
          + to_port           = -1
          + v4_cidr_blocks    = [
              + "0.0.0.0/0",
            ]
          + v6_cidr_blocks    = []
            # (2 unchanged attributes hidden)
        }
    }

  # module.vpc.yandex_vpc_subnet.subnet["0"] will be created
  + resource "yandex_vpc_subnet" "subnet" {
      + created_at     = (known after apply)
      + folder_id      = (known after apply)
      + id             = (known after apply)
      + labels         = (known after apply)
      + name           = "develop-0"
      + network_id     = (known after apply)
      + v4_cidr_blocks = [
          + "10.0.0.0/24",
        ]
      + v6_cidr_blocks = (known after apply)
      + zone           = "ru-central1-d"
    }

  # module.web-vm.yandex_compute_instance.vm[0] will be created
  + resource "yandex_compute_instance" "vm" {
      + allow_stopping_for_update = true
      + created_at                = (known after apply)
      + description               = "TODO: description; {{terraform yyy managed}}"
      + folder_id                 = (known after apply)
      + fqdn                      = (known after apply)
      + gpu_cluster_id            = (known after apply)
      + hardware_generation       = (known after apply)
      + hostname                  = "web-0"
      + id                        = (known after apply)
      + labels                    = {
          + "project" = "web"
        }
      + maintenance_grace_period  = (known after apply)
      + maintenance_policy        = (known after apply)
      + metadata                  = (known after apply)
      + name                      = "web-0"
      + network_acceleration_type = "standard"
      + platform_id               = "standard-v3"
      + reserved_instance_pool_id = (known after apply)
      + status                    = (known after apply)
      + zone                      = "ru-central1-d"

      + boot_disk {
          + auto_delete = true
          + device_name = (known after apply)
          + disk_id     = (known after apply)
          + mode        = (known after apply)

          + initialize_params {
              + block_size  = (known after apply)
              + description = (known after apply)
              + image_id    = "fd8jag83fdegioetshlr"
              + name        = (known after apply)
              + size        = 10
              + snapshot_id = (known after apply)
              + type        = "network-hdd"
            }
        }

      + metadata_options (known after apply)

      + network_interface {
          + index              = (known after apply)
          + ip_address         = (known after apply)
          + ipv4               = true
          + ipv6               = (known after apply)
          + ipv6_address       = (known after apply)
          + mac_address        = (known after apply)
          + nat                = true
          + nat_ip_address     = (known after apply)
          + nat_ip_version     = (known after apply)
          + security_group_ids = (known after apply)
          + subnet_id          = (known after apply)
        }

      + placement_policy (known after apply)

      + resources {
          + core_fraction = 20
          + cores         = 2
          + memory        = 1
        }

      + scheduling_policy {
          + preemptible = true
        }
    }

Plan: 11 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + container_registry_image_tag = (known after apply)

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

random_password.db_password: Creating...
random_password.db_password: Creation complete after 0s [id=none]
module.vpc.yandex_vpc_network.network: Creating...
module.container_registry.yandex_container_registry.registry: Creating...
module.vpc.yandex_vpc_network.network: Creation complete after 2s [id=enptuba06rcdkeqnqc1h]
module.vpc.yandex_vpc_subnet.subnet["0"]: Creating...
module.vpc.yandex_vpc_security_group.security["sec-database"]: Creating...
module.vpc.yandex_vpc_security_group.security["sec-web"]: Creating...
module.vpc.yandex_vpc_subnet.subnet["0"]: Creation complete after 1s [id=fl8oefs7jr5bio8fa2pc]
module.vpc.yandex_vpc_security_group.security["sec-web"]: Creation complete after 2s [id=enp7hg17kp5mn7888qqm]
module.vpc.yandex_vpc_security_group.security["sec-database"]: Creation complete after 4s [id=enp0kdifusmftdpdf433]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Creating...
module.container_registry.yandex_container_registry.registry: Still creating... [00m10s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [00m10s elapsed]
module.container_registry.yandex_container_registry.registry: Still creating... [00m20s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [00m20s elapsed]
module.container_registry.yandex_container_registry.registry: Still creating... [00m30s elapsed]
module.container_registry.yandex_container_registry.registry: Creation complete after 32s [id=crp5mckd1v62viorqvla]
module.container_registry.yandex_container_registry_iam_binding.puller[0]: Creating...
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [00m30s elapsed]
module.container_registry.yandex_container_registry_iam_binding.puller[0]: Creation complete after 5s [id=crp5mckd1v62viorqvla/container-registry.images.puller]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [00m40s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [00m50s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [01m00s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [01m10s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [01m20s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [01m30s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [01m40s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [01m50s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [02m00s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [02m10s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [02m20s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [02m30s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [02m40s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [02m50s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [03m00s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [03m10s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [03m20s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [03m30s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [03m40s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [03m50s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [04m00s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [04m10s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [04m20s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [04m30s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [04m40s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [04m50s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [05m00s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [05m10s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [05m20s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [05m30s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [05m40s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [05m50s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [06m00s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [06m10s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [06m20s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [06m30s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [06m40s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [06m50s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [07m00s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [07m10s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [07m20s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [07m30s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [07m40s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [07m50s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [08m00s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [08m10s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [08m20s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [08m30s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [08m40s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [08m50s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [09m00s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [09m10s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [09m20s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [09m30s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [09m40s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [09m50s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [10m00s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [10m10s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [10m20s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [10m30s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [10m40s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [10m50s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [11m00s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [11m10s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Still creating... [11m20s elapsed]
module.mysql_cluster.yandex_mdb_mysql_cluster.cluster: Creation complete after 11m21s [id=c9qq9ls894vab85h1ts2]
data.template_file.cloudinit: Reading...
module.mysql_cluster_content.yandex_mdb_mysql_database.db: Creating...
data.template_file.cloudinit: Read complete after 0s [id=e87c43320c1e24dd0563e6f266fd9136da5e914b0d5e6edf1bb1853a357eb272]
module.web-vm.yandex_compute_instance.vm[0]: Creating...
module.mysql_cluster_content.yandex_mdb_mysql_database.db: Still creating... [00m10s elapsed]
module.web-vm.yandex_compute_instance.vm[0]: Still creating... [00m10s elapsed]
module.mysql_cluster_content.yandex_mdb_mysql_database.db: Still creating... [00m20s elapsed]
module.web-vm.yandex_compute_instance.vm[0]: Still creating... [00m20s elapsed]
module.mysql_cluster_content.yandex_mdb_mysql_database.db: Still creating... [00m30s elapsed]
module.web-vm.yandex_compute_instance.vm[0]: Still creating... [00m30s elapsed]
module.mysql_cluster_content.yandex_mdb_mysql_database.db: Still creating... [00m40s elapsed]
module.web-vm.yandex_compute_instance.vm[0]: Still creating... [00m40s elapsed]
module.web-vm.yandex_compute_instance.vm[0]: Creation complete after 41s [id=fv4de7vrv0o1vfg8gdo3]
module.mysql_cluster_content.yandex_mdb_mysql_database.db: Still creating... [00m50s elapsed]
module.mysql_cluster_content.yandex_mdb_mysql_database.db: Creation complete after 53s [id=c9qq9ls894vab85h1ts2:db_web]
module.mysql_cluster_content.yandex_mdb_mysql_user.user: Creating...
module.mysql_cluster_content.yandex_mdb_mysql_user.user: Still creating... [00m10s elapsed]
module.mysql_cluster_content.yandex_mdb_mysql_user.user: Still creating... [00m20s elapsed]
module.mysql_cluster_content.yandex_mdb_mysql_user.user: Still creating... [00m30s elapsed]
module.mysql_cluster_content.yandex_mdb_mysql_user.user: Still creating... [00m40s elapsed]
module.mysql_cluster_content.yandex_mdb_mysql_user.user: Still creating... [00m50s elapsed]
module.mysql_cluster_content.yandex_mdb_mysql_user.user: Still creating... [01m00s elapsed]
module.mysql_cluster_content.yandex_mdb_mysql_user.user: Creation complete after 1m7s [id=c9qq9ls894vab85h1ts2:appweb]

Apply complete! Resources: 11 added, 0 changed, 0 destroyed.

Outputs:

container_registry_image_tag = "cr.yandex/crp5mckd1v62viorqvla/application-web:latest"
```
Видимо в конце необходимый output
<img width="807" height="136" alt="image" src="https://github.com/user-attachments/assets/f8d833ca-63c4-4317-b28a-d221775dde62" />

Заходим в папку с репой приложения и собираем образ
<img width="907" height="758" alt="image" src="https://github.com/user-attachments/assets/eb6768b6-4b23-44b1-937b-6d1544809795" />

Далее подключаемся к ВМ, заходим в рабочую папку /opt/web и выполняем ```docker compose up -d```
<img width="636" height="594" alt="image" src="https://github.com/user-attachments/assets/05d528aa-0db1-4577-94e6-3551971741a1" />

Проверяем работоспособность приложения через браузер
<img width="452" height="187" alt="image" src="https://github.com/user-attachments/assets/319ba517-fb85-4f4d-b63f-dc7e1013e8c8" />

Всё работает. Таким образом, ручной труд минимизирован, нужно только скачать исходники, запустить terraform, собрать контейнер и запушить. Вручную тягать проект на ВМ и заполнять переменные не нужно.

P.S. также хотел вместо iam_bindind для реестра контейнеров создать **yandex_container_registry_ip_permission**. Создал для этого в модуле ресурс
```
#resource "yandex_container_registry_ip_permission" "my_ip_permission" {
#  count = length(var.push_ip) > 0 || length(var.pull_ip) > 0 ? 1 : 0
#
#  registry_id = yandex_container_registry.registry.id
#  push        = var.push_ip
#  pull        = var.pull_ip
#}
```
Добавил переменные
```
#variable "push_ip" {
#  type = list(string)
#
#  default = []
#
#  validation {
#    condition = alltrue([
#      for ip in var.push_ip: can(cidrhost(ip, 0))
#    ])
#    error_message = "Проверьте корректность указанной подсети в push_ip"
#  }
#}

#variable "pull_ip" {
#  type = list(string)
#с 
#  default = []
#
#  validation {
#    condition = alltrue([
#      for ip in var.pull_ip: can(cidrhost(ip, 0))
#    ])
#    error_message = "Проверьте корректность указанной подсети в pull_ip"
#  }
#}
```
В переменную pull_ip подставлял белый адрес созданной ВМ, через обращение к готовому ресурсу
Политика доступа по IP создавалась, но ВМ всё равно не могла выполнить pull необходимого контейнера, пытался подставлять и серый адрес(понимаю, что это бессмысленно, но попробовать же стоит), но это не помогло. Информацию найти так и не смог как это сделать корректно
Буду очень признателен, если подскажите, как разрешить доступ с ВМ в реестр без открытия пула для всех(как сейчас) и без передачи кредов сервисного аккаунта на ВМ. Тогда доработаю проект, чтобы можно было использовать 2 варианта(iam bindind и ip bindind)


