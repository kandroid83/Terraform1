## 1. Установка Terraform

**Версия Terraform:**

```bash
terraform --version
text
Terraform v1.14.7
on linux_amd64

![Terraform-1](Terraform-1.png)


2. Клонирование репозитория и переход в 01/src
Репозиторий скачан с GitHub (ZIP-архив). Рабочая директория:

text
~/Загрузки/ter-homeworks-main/01/src
Содержимое каталога:

text
.gitignore  main.tf  .terraformrc
3. Файл для хранения секретов (из .gitignore)
Ответ: в файле .gitignore присутствует строка:

text
# own secret vars store.
personal.auto.tfvars
Следовательно, личную, секретную информацию (логины, пароли, ключи, токены) допустимо хранить в файле personal.auto.tfvars. Terraform автоматически подхватывает переменные из файлов с суффиксом .auto.tfvars, а Git их игнорирует.

![Terraform-2](Terraform-2.png)

4. Секретное содержимое ресурса random_password в state-файле
После выполнения первого terraform apply (с закомментированным блоком Docker) в файле terraform.tfstate был создан ресурс random_password.random_string. В state-файле найден атрибут:

Ключ: result

Значение (сгенерированный пароль): 3Vsmz48nyryt0kAA

Фрагмент terraform.tfstate:

json
"instances": [
    {
        "attributes": {
            ...
            "result": "3Vsmz48nyryt0kAA",
            ...
        }
    }
]![Terraform-3](Terraform-3.png)

5. Исправление ошибок в раскомментированном блоке (строки 29–42)
При раскомментировании блока с Docker и выполнении terraform validate были обнаружены намеренные ошибки. Ниже приведены все ошибки и их исправления.

№	Ошибка	Исправление
1	У ресурса docker_image отсутствует локальное имя	Добавлено "nginx": resource "docker_image" "nginx" {
2	Имя ресурса docker_container начинается с цифры ("1nginx")	Переименовано в "nginx": resource "docker_container" "nginx" {
3	Неправильная интерполяция в name контейнера – использованы фигурные скобки {} вместо ${} и указан несуществующий ресурс random_string_FAKE	Исправлено на: name = "example_${random_password.random_string.result}"
4	Блок ports находился вне ресурса docker_container	Перенесён внутрь ресурса (перед закрывающей })
5	В required_version присутствовала лишняя кавычка и неправильный синтаксис	Исправлено на: required_version = ">= 1.12.0"
Исправленный фрагмент main.tf:

hcl
terraform {
  required_providers {
    docker = {
      source = "kreuzwerker/docker"
    }
  }
  required_version = ">= 1.12.0"
}

provider "docker" {}

resource "random_password" "random_string" {
  length      = 16
  special     = false
  min_upper   = 1
  min_lower   = 1
  min_numeric = 1
}

resource "docker_image" "nginx" {
  name         = "nginx:latest"
  keep_locally = true
}

resource "docker_container" "nginx" {
  image = docker_image.nginx.image_id
  name  = "example_${random_password.random_string.result}"
  ports {
    internal = 80
    external = 9090
  }
}
После исправлений команда terraform validate вернула:

text
Success! The configuration is valid.
6. Выполнение кода и вывод docker ps (после исправления)
После применения конфигурации:

bash
terraform apply -auto-approve
Вывод docker ps:

text
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                  NAMES
c867805c5104   b34848eff6db   "/docker-entrypoint.…"   15 seconds ago   Up 14 seconds   0.0.0.0:9090->80/tcp   example_3Vsmz48nyryt0kAA
![Terraform-5](Terraform-5.png)

7. Замена имени контейнера на hello_world и опасность -auto-approve
В ресурсе docker_container изменено поле name на "hello_world" (имя образа осталось "nginx:latest"). После этого выполнен apply:

bash
terraform apply -auto-approve
Опасность ключа -auto-approve:

Terraform применяет изменения автоматически, без запроса подтверждения (yes).

В производственной среде это может привести к непреднамеренному удалению или изменению критических ресурсов, недоступности сервисов или потере данных.

В то же время ключ полезен для CI/CD и автоматических развертываний, а также для локальной разработки, когда вы уверены в изменениях и не хотите тратить время на подтверждение.

Вывод docker ps после замены:

text
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                  NAMES
9c7f4637c09f   b34848eff6db   "/docker-entrypoint.…"   14 seconds ago   Up 13 seconds   0.0.0.0:9090->80/tcp   hello_world
![Terraform-6](Terraform-6.png)

8. Уничтожение ресурсов и содержимое terraform.tfstate
Выполнено уничтожение:

bash
terraform destroy -auto-approve
Результат:

text
Destroy complete! Resources: 0 destroyed.
Содержимое terraform.tfstate после уничтожения:

json
{
    "version": 4,
    "terraform_version": "1.14.7",
    "serial": 1,
    "lineage": "f6c793f1-c843-c0ba-566b-559949b89dd3",
    "outputs": {},
    "resources": [],
    "check_results": null
}
![Terraform-7](Terraform-7.png)

9. Почему образ nginx:latest не был удалён
В ресурсе docker_image явно указан параметр:

hcl
keep_locally = true
Согласно документации провайдера Docker:

keep_locally – If true, then the Docker image won't be deleted on destroy operation.

Именно поэтому после terraform destroy образ nginx:latest остался в локальном хранилище Docker. Проверка:

bash
docker images | grep nginx
text
nginx:latest    b34848eff6db    241MB    66.2MB
![Terraform-8](Terraform-8.png)

