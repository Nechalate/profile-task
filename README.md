# Плейбук для развёртывания Chrony, Nginx, PostgreSQL на RHEL системах.

## Запуск

1. Клонирование репозитория.

git clone https://github.com/Nechalate/profile-task.git
cd ansible-ntp-app-db

2. Роли установлены через Ansible Galaxy.

**Роли включены в проект, в них внесены изменения для предотвращения падения при запуске чек мода (ещё не установленные/созданный пакеты/пользователь БД), повторная загрузка вызовет падения в чек моде**

Ansible Galaxy - galaxy.ansible.com

Установка ролей через файл requirements:

ansible-galaxy install -r requirements.yml -p ./roles

3. Настройка инвентаря.

Отредактируйте inventory/hosts.yml, указав IP адреса и имена пользователей ваших серверов.

```yaml
ntp:
  hosts:
    ntp-server:
      ansible_host: <IP-адрес>
      ansible_user: <Имя пользователя>
      ansible_ssh_private_key_file: ~/.ssh/id_rsa
```

4. Проверьте соединения до ваших серверов.

ansible all -i inventory/hosts.yml -m ping --become

5. Запустите плейбук в режиме проверки. 

ansible-playbook -i inventory/hosts.yml site.yml --check --diff

6. Установка пакетов на сервера.

ansible-playbook -i inventory/hosts.yml site.yml

Если вам нужно развернуть отдельную роль укажите её командой:

ansible-playbook -i inventory/hosts.yml site.yml --tags <группа к которой принадлежит сервер>

Пример:

ansible-playbook -i inventory/hosts.yml site.yml --tags ntp


## Проверка работоспособности

1. Chrony

Подключитесь по ssh и выполните команду

ssh user@<IP>
chronyc tracking

Ожидаемый вывод:

Leap status : Normal

2. Nginx

Выволните команду curl

curl http://<IP>

Или зайдите в ваш браузер, и войдите по ссылке:
http://<IP>

Ожидаемый результат:

В терминале:

Html код страницы с указанием имени хоста и айпи адресом.

В браузере:

Приветственная страница Nginx с указанием имени хоста и айпи адресом.

3. PostgreSQL

Подключитесь по ssh и напишите несколько команд:

sudo -u postgres psql -c "SELECT version();"

Ожидаемый вывод:

PostgreSQL 13.23

sudo cat /var/lib/pgsql/data/pg_hba.conf | grep -E "127.0.0.1|::1"

Ожидаемый вывод:

host all all 127.0.0.1/32   md5 
host all all ::1/128   md5 

## Внесённые изменения

Падения при вызове check

Были внесены небольшие правки в скачанные роли, а именно убраны проверки ещё не установленных и не созданных пакетов/пользователей.

Изменения были внесены в следующие файлы:

- roles/nginx/tasks/main.yml
- roles/postgresql/tasks/initialize.yml
- roles/postgresql/tasks/configure.yml
- roles/postgresql/tasks/main.yml
- roles/postgresql/handlers/main.yml

Правки не влияют на реальный запуск а лишь предотвращают падения плейбука в check моде.

## Переменные и хендлеры

Переменные

- group_vars/ntp.yml - список ntp серверов.
- group_vars/app.yml - удаление дефолтного хоста, отключение управления nginx ролью для предотвращения падения с ошибкой, поведение в check-mode
- group_vars/db.yml - указание версии, разрешение соеденения с localhost.

Handlers

- restart chrony - вызов при изменении конфигурации
- restart nginx - вызов после обновления index.html


## Запуск линтера

ansible-lint site.yml

Ожидаемый результат:

Passing

## Очистка установленных пакетов

sudo systemctl stop chronyd nginx postgresql-13 2>/dev/null || true
sudo systemctl disable chronyd nginx postgresql-13 2>/dev/null || true

sudo dnf remove -y chrony nginx postgresql13\* postgresql-private-libs\* postgresql-contrib\* postgresql-server\*

sudo rm -f /etc/yum.repos.d/pgdg-redhat-all.repo

sudo rm -rf /etc/chrony.conf.rpmsave /etc/nginx /var/lib/pgsql /var/log/nginx /usr/share/nginx/html

sudo userdel -r nginx 2>/dev/null || true
sudo userdel -r postgres 2>/dev/null || true

sudo dnf clean all

rpm -qa | grep -E "chrony|nginx|postgresql"
