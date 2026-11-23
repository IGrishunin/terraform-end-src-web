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

# Запуск проекта
Для запуска проекта необходимо следующее:
1) Заполнить personal.auto.tfvars(в репозитории лежит example)
2) Выполнить ```terraform init``` -> ```terraform plan``` -> ```terraform apply```
3) После того, как выполнится ```terraform apply``` на экран выведется output, в котором будет указано куда нужно запушить собранный docker образ
4) Клонируем репозиторий
