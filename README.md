# Домашнее задание выполнено для курса [Infrastructure as a code](https://otus.ru/lessons/infrastructure-as-a-code/)

00. Перед началом выполнения заданий, требуется доступ в Yandex Cloud. Создать новый каталог (заповнив его id + id облака)

### Цель: Подготовка отказоустойчивой облачной инфраструктуры для установки Wordpress

01. Создать в корне проекта директорию `terraform` и перейти в нее:
```bash
mkdir terraform && cd terraform
```

02. Создать файл `provider.tf` и объявить наш облачный провайдер Yandex.Cloud
03. Создать файл с переменными `variables.tf` и указать все переменные, которые будут использоваться в манифестах (их значения указать в файде `wp.auto.tfvars`, который необходимо добавить в `.gitignore` данного проекта!). В данный файл внести id каталога и облака + указать свой токен (можно узнать через команду `yc config list`)
04. Выполнить инициализацию терраформа:
```bash
terraform init
```
05. Создать виртуальную сеть, описав ее в файле `network.tf`
06. Создание хостов выполнить через `wp-app.tf`
07. Использование балансировщика описывает файл `lb.tf`
08. Создание кластера БД PostgreSQL: `db.tf` 
09. Получить и настроить информативный вывод в консоль можно через `output.tf`
10. После описания инфраструктуры выполнить комаду:
```bash
terraform apply --auto-approve
```
11. По завершению работы рекомендуется освободить ресурсы облака выполнив команду:
```bash
terraform destroy --auto-approve
```

---

### Использование Ansible ролей для установки Wordpress на группу хостов

00. Наличие установленного Ansible не ниже версии 2.9. Выполнить редактирование файла db.tf, а именно,  
   теперь нужно использовать ресурс типа `yandex_mdb_mysql_cluster` для создания базы + теперь  
   используем механизм проверки пароля пользователя `authentication_plugin = "MYSQL_NATIVE_PASSWORD"`

1.  `healthcheck` в файле `lb.tf` изменить на `tcp`
2.  Создать структуру в новой директории `ansible` следующего вида:
```bash
├── ansible.cfg
├── environments
│   └── prod
│       ├── group_vars
│       │   └── wp_app
│       └── inventory
├── playbooks
│   └── install.yml
└── roles
    └── wordpress
        ├── defaults
        │   └── main.yml
        ├── files
        │   └── root.crt
        ├── meta
        │   └── main.yml
        ├── tasks
        │   └── main.yml
        └── templates
            ├── wordpress.conf.j2
            └── wp-config.php.j2 
```

- `ansible.cfg` - указать следующие параметры:  
  `private_key_file` - путь до вашего приватного ключа  
  `inventory` - путь до файла inventory по-умолчанию
- `wp_app` - заполняется в соответсвии с ранее указанными в терраформе параметрами  
  \+ FQDN БД можно узнать по [инструкции](https://cloud.yandex.com/en-ru/docs/managed-mysql/operations/connect#fqdn-master).
- Файл `inventory` заполнить в соответсвии с публичными (!) IP ваших app-серверов.
- Файл `wordpress/defaults/main.yml`  - содержит абстрактные переменные, которые по факту перезапишутся  
  реальными из `environments/prod/group_vars/wp_app`  
- `wordpress/files/root.crt` - корневой сертификат облака полученный путем выполнения команды:
  ```bash
  wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" -O .wordpress/files/root.crt
  ```
- `wordpress/meta/main.yml` - общие сведения о роли
- `wordpress/tasks/main.yml` - основные задачи, который выполняет роль
- `wordpress/templates/` - содержит шаблоны конфигураций для wordpress, которые заливаем на app-хосты

3.  Запуск можно выполнять из директории проекта командой:  
    ```bash
    ansible-playbook playbooks/install.yml
    ```


### Тестирование инфраструктуры с помощью Go

0. Тесты проводить в отедбном каталоге в YC. При его создании (например каталог `stage`) запомнить его id,  
   т.к. он потребуется в дальнейшем при запуске тестов.

1. [Установить](https://golang.org/doc/install) самую последнюю версию Go локально  
2. Тесты в проекте включают в себя следующие пункты:
   - проверка наличия IP Load balancer-а в state-е терраформа
   - возможность подключиться по ssh к одной из виртуальных машин

3. Тесты созданы в отдельной ветке `ansible`
4. Тесты располагаются в каталоге `terrafotm/test/end2end_test.go`
5. Перед запуском выполнить инициализацию модуля Go:
  ```bash
  go mod init test  
  ```
6. Все новые зависимости в блоке `import` необходимо подгружать командой:
  ```bash
  go mod vendor
  ```
7. Запуск тестов:
  ```bash
  go test -v ./ -timeout 30m -folder 'yc_folder_id'
  ```
8. Управление стадиями `SKIP_setup`, `SKIP_validate`, `SKIP_teardown` происходит через определение  
  переменных окружений, например:
  ```bash
  export SKIP_validate=true
  export SKIP_teardown=true
  export SKIP_setup=true
  ```
  Чтобы вернуть необходимую стадию, выполнить, например:
  ```bash
  unset SKIP_validate
  ```

9. Пример команды запуска теста с проверкой возможности зайти по ssh:
  ```bash
  go test -v ./ -timeout 30m -folder 'yc_folder_id' -ssh-key-pass '/home/user/.ssh/id_rsa'
  
  ```