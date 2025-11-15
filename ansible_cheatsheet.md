# Ansible - Полная шпаргалка (ansible-lint v25)

## 📋 Содержание
1. [Установка и настройка](#установка-и-настройка)
2. [Структура проекта](#структура-проекта)
3. [Inventory (Инвентарь)](#inventory-инвентарь)
4. [Playbooks](#playbooks)
5. [Tasks (Задачи)](#tasks-задачи)
6. [Variables (Переменные)](#variables-переменные)
7. [Handlers](#handlers)
8. [Templates (Jinja2)](#templates-jinja2)
9. [Roles](#roles)
10. [Модули](#модули)
11. [Условия и циклы](#условия-и-циклы)
12. [Ansible Vault](#ansible-vault)
13. [Команды CLI](#команды-cli)
14. [Best Practices](#best-practices)
15. [Ansible-lint v25 правила](#ansible-lint-v25-правила)

---

## Установка и настройка

### Установка
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install ansible

# RHEL/CentOS
sudo yum install ansible

# macOS
brew install ansible

# Python pip (рекомендуется)
pip install ansible

# Установка ansible-lint v25
pip install ansible-lint
```

### Конфигурационный файл ansible.cfg
```ini
[defaults]
# Файл инвентаря
inventory = ./inventory/hosts.ini

# Отключить проверку SSH host key
host_key_checking = False

# Пользователь для подключения
remote_user = ansible

# Путь к ролям
roles_path = ./roles

# Количество параллельных процессов
forks = 5

# Timeout для подключения
timeout = 10

# Формат вывода (stdout, yaml, json)
stdout_callback = yaml

# Логирование
log_path = ./ansible.log

# Retry файлы
retry_files_enabled = True
retry_files_save_path = ./retry

[privilege_escalation]
# Использовать sudo
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
# SSH аргументы
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
# Использовать pipelining для ускорения
pipelining = True
```

---

## Структура проекта

### Рекомендуемая структура директорий
```
ansible-project/
├── ansible.cfg              # Конфигурация Ansible
├── .ansible-lint            # Конфигурация ansible-lint
├── inventory/               # Директория с инвентарем
│   ├── hosts.ini           # Статический инвентарь
│   ├── group_vars/         # Переменные для групп
│   │   ├── all.yml
│   │   ├── webservers.yml
│   │   └── databases.yml
│   └── host_vars/          # Переменные для конкретных хостов
│       ├── web01.yml
│       └── db01.yml
├── playbooks/              # Плейбуки
│   ├── site.yml           # Главный плейбук
│   ├── webservers.yml
│   └── databases.yml
├── roles/                  # Роли
│   ├── common/
│   ├── webserver/
│   └── database/
├── files/                  # Статические файлы
├── templates/              # Jinja2 шаблоны
├── vars/                   # Дополнительные переменные
├── vault/                  # Зашифрованные данные
└── requirements.yml        # Зависимости (коллекции, роли)
```

### .ansible-lint конфигурация (v25)
```yaml
---
# .ansible-lint
profile: production  # Профиль: min, basic, moderate, safety, shared, production

# Пропустить определенные правила
skip_list:
  - yaml[line-length]  # Игнорировать длину строк

# Только предупреждения для этих правил
warn_list:
  - experimental

# Исключить пути
exclude_paths:
  - .cache/
  - .github/
  - tests/
  - molecule/
  - .venv/

# Включить offline режим
# offline: true

# Использовать parseable формат для CI/CD
# parseable: true

# Строгий режим
# strict: true
```

---

## Inventory (Инвентарь)

### Формат INI
```ini
# inventory/hosts.ini

# Одиночные хосты
web01.example.com
192.168.1.10

# Группы хостов
[webservers]
web01.example.com
web02.example.com
web03.example.com

[databases]
db01.example.com ansible_host=192.168.1.20
db02.example.com ansible_host=192.168.1.21

# Группа групп
[production:children]
webservers
databases

# Переменные для группы
[webservers:vars]
ansible_user=deploy
ansible_port=22
http_port=80
max_clients=200

# Паттерны хостов
[web_cluster]
web[01:10].example.com

# С разными портами
[web_special]
web01.example.com:2222
web02.example.com:3333

# Алиасы хостов
[databases]
db_master ansible_host=192.168.1.20 ansible_user=admin
db_slave ansible_host=192.168.1.21 ansible_user=admin
```

### Формат YAML
```yaml
# inventory/hosts.yml
---
all:
  children:
    webservers:
      hosts:
        web01.example.com:
          ansible_host: 192.168.1.10
          http_port: 80
        web02.example.com:
          ansible_host: 192.168.1.11
          http_port: 8080
      vars:
        ansible_user: deploy
        ansible_become: true

    databases:
      hosts:
        db01.example.com:
          ansible_host: 192.168.1.20
        db02.example.com:
          ansible_host: 192.168.1.21
      vars:
        ansible_user: dbadmin

    production:
      children:
        webservers:
        databases:
```

### Динамический инвентарь
```python
#!/usr/bin/env python3
# inventory/dynamic_inventory.py

import json

def get_inventory():
    return {
        "webservers": {
            "hosts": ["web01.example.com", "web02.example.com"],
            "vars": {
                "ansible_user": "deploy",
                "http_port": 80
            }
        },
        "_meta": {
            "hostvars": {
                "web01.example.com": {
                    "ansible_host": "192.168.1.10"
                },
                "web02.example.com": {
                    "ansible_host": "192.168.1.11"
                }
            }
        }
    }

if __name__ == "__main__":
    print(json.dumps(get_inventory()))
```

---

## Playbooks

### Базовая структура плейбука
```yaml
---
# playbooks/site.yml
- name: Configure web servers
  hosts: webservers
  become: true
  gather_facts: true

  vars:
    http_port: 80
    max_clients: 200

  vars_files:
    - vars/common.yml
    - vars/webserver.yml

  pre_tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"

  roles:
    - common
    - role: webserver
      http_port: 8080
    - role: monitoring
      vars:
        monitoring_enabled: true

  tasks:
    - name: Ensure nginx is installed
      ansible.builtin.package:
        name: nginx
        state: present

    - name: Start and enable nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

  post_tasks:
    - name: Verify nginx is running
      ansible.builtin.uri:
        url: "http://localhost:{{ http_port }}"
        status_code: 200

  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

### Множественные плеи в одном плейбуке
```yaml
---
# Первый плей - для веб-серверов
- name: Configure web servers
  hosts: webservers
  become: true

  tasks:
    - name: Install nginx
      ansible.builtin.package:
        name: nginx
        state: present

# Второй плей - для баз данных
- name: Configure databases
  hosts: databases
  become: true

  tasks:
    - name: Install postgresql
      ansible.builtin.package:
        name: postgresql
        state: present

# Третий плей - для всех хостов
- name: Configure monitoring
  hosts: all
  become: true

  tasks:
    - name: Install monitoring agent
      ansible.builtin.package:
        name: node-exporter
        state: present
```

---

## Tasks (Задачи)

### Базовый синтаксис задачи (FQCN - обязательно в v25)
```yaml
# FQCN (Fully Qualified Collection Name) - обязательно в ansible-lint v25
- name: Install nginx package
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: true
  register: result_variable
  when: condition
  notify: Handler name
  changed_when: false
  failed_when: false
  ignore_errors: false
  delegate_to: other_host
  run_once: true
  tags:
    - configuration
    - webserver
```

### Примеры задач с FQCN
```yaml
tasks:
  # Установка пакета
  - name: Install nginx
    ansible.builtin.apt:
      name: nginx
      state: present
      update_cache: true
    tags:
      - packages
      - nginx

  # Копирование файла
  - name: Copy nginx config
    ansible.builtin.copy:
      src: files/nginx.conf
      dest: /etc/nginx/nginx.conf
      owner: root
      group: root
      mode: "0644"
      backup: true
    notify: Restart nginx

  # Использование шаблона
  - name: Generate nginx config from template
    ansible.builtin.template:
      src: templates/nginx.conf.j2
      dest: /etc/nginx/nginx.conf
      owner: root
      group: root
      mode: "0644"
      validate: nginx -t -c %s
    notify: Restart nginx

  # Выполнение команды (обязательно с changed_when)
  - name: Check nginx configuration
    ansible.builtin.command: nginx -t
    register: nginx_test
    changed_when: false
    failed_when: nginx_test.rc != 0

  # Shell команда с pipefail
  - name: Get running processes
    ansible.builtin.shell: |
      set -o pipefail
      ps aux | grep nginx | grep -v grep
    args:
      executable: /bin/bash
    register: nginx_processes
    changed_when: false
    failed_when: false

  # Создание директории
  - name: Create web directory
    ansible.builtin.file:
      path: /var/www/mysite
      state: directory
      owner: www-data
      group: www-data
      mode: "0755"

  # Создание символической ссылки
  - name: Create symlink
    ansible.builtin.file:
      src: /etc/nginx/sites-available/mysite
      dest: /etc/nginx/sites-enabled/mysite
      state: link

  # Управление сервисом
  - name: Ensure nginx is running
    ansible.builtin.service:
      name: nginx
      state: started
      enabled: true

  # Скачивание файла
  - name: Download file
    ansible.builtin.get_url:
      url: https://example.com/file.tar.gz
      dest: /tmp/file.tar.gz
      checksum: sha256:abc123...
      mode: "0644"

  # Распаковка архива
  - name: Extract archive
    ansible.builtin.unarchive:
      src: /tmp/file.tar.gz
      dest: /opt/myapp
      remote_src: true

  # Git checkout
  - name: Clone repository
    ansible.builtin.git:
      repo: https://github.com/user/repo.git
      dest: /opt/myapp
      version: main
      force: true

  # Управление пользователями
  - name: Create user
    ansible.builtin.user:
      name: deploy
      shell: /bin/bash
      groups: sudo
      append: true
      create_home: true

  # Управление правами sudo
  - name: Add sudoers entry
    ansible.builtin.lineinfile:
      path: /etc/sudoers.d/deploy
      line: deploy ALL=(ALL) NOPASSWD: ALL
      create: true
      mode: "0440"
      validate: visudo -cf %s

  # Добавление строки в файл
  - name: Add line to file
    ansible.builtin.lineinfile:
      path: /etc/hosts
      line: 192.168.1.10 myserver.local
      state: present

  # Замена в файле
  - name: Replace string in file
    ansible.builtin.replace:
      path: /etc/nginx/nginx.conf
      regexp: worker_processes.*
      replace: worker_processes auto;

  # Блок строк в файле
  - name: Add block to file
    ansible.builtin.blockinfile:
      path: /etc/ssh/sshd_config
      block: |
        PermitRootLogin no
        PasswordAuthentication no
        PubkeyAuthentication yes
      marker: "# {mark} ANSIBLE MANAGED BLOCK"

  # Проверка URL
  - name: Check website is up
    ansible.builtin.uri:
      url: http://localhost
      status_code: 200

  # Ожидание порта
  - name: Wait for port 80
    ansible.builtin.wait_for:
      port: 80
      delay: 5
      timeout: 60

  # Pause (пауза)
  - name: Wait for manual verification
    ansible.builtin.pause:
      prompt: Please verify the configuration and press enter to continue

  # Debug вывод
  - name: Debug variable
    ansible.builtin.debug:
      var: nginx_test
      verbosity: 2

  # Assert (проверка условия)
  - name: Assert condition
    ansible.builtin.assert:
      that:
        - ansible_distribution == "Ubuntu"
        - ansible_distribution_version is version('20.04', '>=')
      fail_msg: Unsupported OS version

  # Выполнение на localhost
  - name: Run locally
    ansible.builtin.command: echo "Running on control node"
    delegate_to: localhost
    run_once: true
    changed_when: false
```

### Блоки задач
```yaml
- name: Block example with error handling
  block:
    - name: Install package
      ansible.builtin.apt:
        name: nginx
        state: present

    - name: Start service
      ansible.builtin.service:
        name: nginx
        state: started

  rescue:
    - name: Handle error
      ansible.builtin.debug:
        msg: Installation failed, rolling back

    - name: Remove package
      ansible.builtin.apt:
        name: nginx
        state: absent

  always:
    - name: Cleanup
      ansible.builtin.file:
        path: /tmp/nginx_install.lock
        state: absent
```

---

## Variables (Переменные)

### Именование переменных (snake_case)
```yaml
# ✅ Правильно - snake_case
my_variable: value
user_name: john
http_port: 80
database_host: localhost
enable_monitoring: true

# ❌ Неправильно
MyVariable: value      # CamelCase
my-variable: value     # kebab-case
123variable: value     # начинается с цифры
my.variable: value     # содержит точку
```

### Типы переменных
```yaml
# Строки
server_name: web01.example.com
description: This is a string
path: /var/www/html

# Числа
http_port: 80
max_connections: 1000
timeout: 30.5

# Булевы значения (только true/false)
debug_mode: true
enabled: false
use_ssl: true

# Списки (массивы)
packages:
  - nginx
  - postgresql
  - redis

# Альтернативный синтаксис списков
packages: [nginx, postgresql, redis]

# Словари (объекты)
database:
  host: localhost
  port: 5432
  name: mydb
  user: dbuser
  password: secret

# Альтернативный синтаксис словарей
database: {host: localhost, port: 5432, name: mydb}

# Многострочные значения
ssl_certificate: |
  -----BEGIN CERTIFICATE-----
  MIIDXTCCAkWgAwIBAgIJAKZ...
  -----END CERTIFICATE-----

# Null значения
optional_value: null
empty_value: ~
```

### Приоритет переменных (от низкого к высокому)
```
1. role defaults (defaults/main.yml в роли)
2. inventory vars (group_vars/all.yml)
3. inventory group_vars (group_vars/webservers.yml)
4. inventory host_vars (host_vars/web01.yml)
5. playbook group_vars
6. playbook host_vars
7. host facts
8. registered vars
9. set_facts
10. play vars
11. play vars_prompt
12. play vars_files
13. role and include vars
14. block vars
15. task vars
16. extra vars (ansible-playbook -e)
```

### Файлы переменных

#### group_vars/all.yml
```yaml
---
# Переменные для всех хостов
ansible_user: ansible
ansible_become: true
ntp_servers:
  - 0.pool.ntp.org
  - 1.pool.ntp.org
```

#### group_vars/webservers.yml
```yaml
---
# Переменные для группы webservers
http_port: 80
https_port: 443
document_root: /var/www/html
max_clients: 200
```

#### host_vars/web01.yml
```yaml
---
# Переменные для конкретного хоста
ansible_host: 192.168.1.10
http_port: 8080
is_primary: true
```

### Регистрация переменных
```yaml
- name: Check if file exists
  ansible.builtin.stat:
    path: /etc/myapp/config.yml
  register: config_file

- name: Debug registered variable
  ansible.builtin.debug:
    msg: "File exists: {{ config_file.stat.exists }}"

- name: Use registered variable in condition
  ansible.builtin.template:
    src: config.yml.j2
    dest: /etc/myapp/config.yml
    mode: "0644"
  when: not config_file.stat.exists

# Получение вывода команды
- name: Get service status
  ansible.builtin.command: systemctl status nginx
  register: service_status
  changed_when: false
  failed_when: false

- name: Show command output
  ansible.builtin.debug:
    msg: "{{ service_status.stdout_lines }}"
```

### Set_fact
```yaml
- name: Calculate derived values
  ansible.builtin.set_fact:
    full_name: "{{ first_name }} {{ last_name }}"
    is_production: "{{ env == 'prod' }}"
    cache_size_mb: "{{ (total_memory_mb * 0.25) | int }}"

- name: Use calculated fact
  ansible.builtin.debug:
    msg: "Full name is {{ full_name }}"
```

### Переменные окружения
```yaml
- name: Set environment variables for task
  ansible.builtin.command: /opt/myapp/script.sh
  environment:
    PATH: "/usr/local/bin:{{ ansible_env.PATH }}"
    DB_HOST: "{{ database_host }}"
    DB_USER: "{{ database_user }}"
    DB_PASSWORD: "{{ database_password }}"
  changed_when: false
```

### Магические переменные (встроенные)
```yaml
# Информация о хосте
{{ inventory_hostname }}           # Имя хоста из инвентаря
{{ ansible_hostname }}             # Фактическое имя хоста
{{ ansible_fqdn }}                 # FQDN хоста
{{ ansible_default_ipv4.address }} # IP адрес

# Информация о системе
{{ ansible_distribution }}         # Ubuntu, CentOS, etc.
{{ ansible_distribution_version }} # 20.04, 7, etc.
{{ ansible_os_family }}           # Debian, RedHat, etc.
{{ ansible_architecture }}        # x86_64, armv7l, etc.

# Информация о плейбуке
{{ playbook_dir }}                # Директория плейбука
{{ role_path }}                   # Путь к текущей роли
{{ ansible_play_hosts }}          # Список хостов в плее
{{ ansible_play_batch }}          # Текущий batch хостов

# Группы и хосты
{{ groups['webservers'] }}        # Список хостов в группе
{{ groups.keys() }}               # Список всех групп
{{ hostvars[inventory_hostname] }}# Переменные другого хоста
```

---

## Handlers

### Базовый синтаксис (с FQCN)
```yaml
# В playbook или role/handlers/main.yml
---
handlers:
  - name: Restart nginx
    ansible.builtin.service:
      name: nginx
      state: restarted

  - name: Reload nginx
    ansible.builtin.service:
      name: nginx
      state: reloaded

  - name: Restart apache
    ansible.builtin.service:
      name: apache2
      state: restarted
    when: ansible_os_family == "Debian"
```

### Вызов handlers
```yaml
tasks:
  - name: Update nginx config
    ansible.builtin.template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
      mode: "0644"
    notify:
      - Restart nginx

  - name: Update multiple configs
    ansible.builtin.template:
      src: "{{ item }}.j2"
      dest: "/etc/nginx/{{ item }}"
      mode: "0644"
    loop:
      - nginx.conf
      - sites-available/default
    notify:
      - Reload nginx
      - Restart php-fpm
```

### Listening handlers (группировка)
```yaml
handlers:
  - name: Restart web services
    ansible.builtin.service:
      name: "{{ item }}"
      state: restarted
    loop:
      - nginx
      - php-fpm
    listen: restart web stack

tasks:
  - name: Update config
    ansible.builtin.template:
      src: config.j2
      dest: /etc/config
      mode: "0644"
    notify: restart web stack
```

### Принудительный вызов handlers
```yaml
- name: Ensure handlers run now
  ansible.builtin.meta: flush_handlers

- name: Continue with other tasks
  ansible.builtin.debug:
    msg: Handlers have been executed
```

---

## Templates (Jinja2)

### Базовый шаблон
```jinja2
{# templates/nginx.conf.j2 #}
{# Комментарий в Jinja2 #}

# Nginx Configuration
user {{ nginx_user }};
worker_processes {{ ansible_processor_vcpus }};
pid /run/nginx.pid;

events {
    worker_connections {{ max_connections | default(1024) }};
}

http {
    server_tokens {{ 'on' if debug_mode else 'off' }};

    # Upstream servers
    {% for server in backend_servers %}
    upstream backend_{{ loop.index }} {
        server {{ server.host }}:{{ server.port }};
    }
    {% endfor %}

    server {
        listen {{ http_port }};
        server_name {{ server_name }};
        root {{ document_root }};

        {% if enable_ssl %}
        listen {{ https_port }} ssl;
        ssl_certificate {{ ssl_cert_path }};
        ssl_certificate_key {{ ssl_key_path }};
        {% endif %}
    }
}
```

### Условия
```jinja2
{% if ansible_distribution == "Ubuntu" %}
    # Ubuntu specific configuration
    user www-data;
{% elif ansible_distribution == "CentOS" %}
    # CentOS specific configuration
    user nginx;
{% else %}
    # Default configuration
    user nobody;
{% endif %}

{# Тернарный оператор #}
{{ 'enabled' if feature_enabled else 'disabled' }}

{# Проверка существования #}
{% if database_password is defined %}
    password {{ database_password }}
{% endif %}

{# Проверка на None/пустоту #}
{% if variable is not none %}
    value {{ variable }}
{% endif %}
```

### Циклы
```jinja2
{# Простой цикл #}
{% for package in packages %}
    - {{ package }}
{% endfor %}

{# Цикл с индексом #}
{% for server in servers %}
    server {{ loop.index }}: {{ server.name }}
    {# loop.index начинается с 1 #}
    {# loop.index0 начинается с 0 #}
{% endfor %}

{# Цикл по словарю #}
{% for key, value in database_config.items() %}
    {{ key }} = {{ value }}
{% endfor %}

{# Специальные переменные цикла #}
{% for item in items %}
    {% if loop.first %}
        # First item
    {% endif %}

    {{ item }}

    {% if loop.last %}
        # Last item
    {% endif %}
{% endfor %}

{# Цикл с else (если список пустой) #}
{% for user in users %}
    - {{ user }}
{% else %}
    No users defined
{% endfor %}
```

### Фильтры
```jinja2
{# Строковые фильтры #}
{{ server_name | upper }}           # UPPERCASE
{{ server_name | lower }}           # lowercase
{{ server_name | capitalize }}      # Capitalize
{{ server_name | title }}           # Title Case

{# Значение по умолчанию #}
{{ variable | default('default_value') }}
{{ variable | default(omit) }}      # Пропустить параметр если не определен

{# Числовые операции #}
{{ memory_size | int }}             # Преобразовать в int
{{ price | float }}                 # Преобразовать в float
{{ number | abs }}                  # Абсолютное значение
{{ number | round }}                # Округлить

{# Списки #}
{{ list_var | length }}             # Длина списка
{{ list_var | first }}              # Первый элемент
{{ list_var | last }}               # Последний элемент
{{ list_var | join(', ') }}         # Объединить с разделителем
{{ list_var | unique }}             # Уникальные значения
{{ list_var | sort }}               # Сортировать

{# JSON #}
{{ dict_var | to_json }}            # В JSON
{{ dict_var | to_nice_json }}       # В красивый JSON
{{ json_string | from_json }}       # Из JSON

{# YAML #}
{{ dict_var | to_yaml }}            # В YAML
{{ dict_var | to_nice_yaml }}       # В красивый YAML

{# Работа с путями #}
{{ filepath | basename }}           # Имя файла
{{ filepath | dirname }}            # Директория

{# Хеши и кодирование #}
{{ password | password_hash('sha512') }}
{{ string | b64encode }}            # Base64 encode
{{ encoded | b64decode }}           # Base64 decode

{# Regex #}
{{ string | regex_replace('^www\.', '') }}
{{ string | regex_search('pattern') }}
```

### Использование шаблона в задаче
```yaml
- name: Generate config from template
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: "0644"
    backup: true
    validate: nginx -t -c %s
  vars:
    http_port: 80
    server_name: example.com
  notify: Restart nginx
```

---

## Roles

### Структура роли
```
roles/webserver/
├── defaults/           # Переменные по умолчанию (самый низкий приоритет)
│   └── main.yml
├── files/             # Статические файлы
│   ├── index.html
│   └── logo.png
├── handlers/          # Обработчики
│   └── main.yml
├── meta/              # Метаданные и зависимости
│   └── main.yml
├── tasks/             # Задачи
│   ├── main.yml
│   ├── install.yml
│   └── configure.yml
├── templates/         # Jinja2 шаблоны
│   ├── nginx.conf.j2
│   └── site.conf.j2
├── tests/             # Тесты роли
│   ├── inventory
│   └── test.yml
└── vars/              # Переменные (высокий приоритет)
    └── main.yml
```

### Создание роли
```bash
# Создать структуру роли
ansible-galaxy init webserver

# Создать роль в определенной директории
ansible-galaxy init --init-path ./roles webserver
```

### defaults/main.yml
```yaml
---
# Переменные по умолчанию (могут быть переопределены)
webserver_http_port: 80
webserver_https_port: 443
webserver_user: www-data
webserver_worker_processes: auto
webserver_worker_connections: 1024
webserver_enable_ssl: false
webserver_server_name: localhost
webserver_document_root: /var/www/html
```

### vars/main.yml
```yaml
---
# Переменные с высоким приоритетом (сложно переопределить)
webserver_package_name: nginx
webserver_service_name: nginx
webserver_config_dir: /etc/nginx
webserver_log_dir: /var/log/nginx
```

### tasks/main.yml
```yaml
---
# Главный файл задач - точка входа
- name: Include OS-specific variables
  ansible.builtin.include_vars: "{{ ansible_os_family }}.yml"

- name: Include installation tasks
  ansible.builtin.include_tasks: install.yml
  tags:
    - install
    - webserver

- name: Include configuration tasks
  ansible.builtin.include_tasks: configure.yml
  tags:
    - configure
    - webserver

- name: Include SSL setup tasks
  ansible.builtin.include_tasks: ssl.yml
  when: webserver_enable_ssl
  tags:
    - ssl
    - webserver
```

### tasks/install.yml
```yaml
---
- name: Install nginx on Debian/Ubuntu
  ansible.builtin.apt:
    name: "{{ webserver_package_name }}"
    state: present
    update_cache: true
  when: ansible_os_family == "Debian"

- name: Install nginx on RedHat/CentOS
  ansible.builtin.yum:
    name: "{{ webserver_package_name }}"
    state: present
  when: ansible_os_family == "RedHat"

- name: Ensure nginx is started and enabled
  ansible.builtin.service:
    name: "{{ webserver_service_name }}"
    state: started
    enabled: true
```

### tasks/configure.yml
```yaml
---
- name: Create document root directory
  ansible.builtin.file:
    path: "{{ webserver_document_root }}"
    state: directory
    owner: "{{ webserver_user }}"
    group: "{{ webserver_user }}"
    mode: "0755"

- name: Deploy nginx configuration
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: "{{ webserver_config_dir }}/nginx.conf"
    owner: root
    group: root
    mode: "0644"
    validate: nginx -t -c %s
  notify: Restart nginx

- name: Deploy site configuration
  ansible.builtin.template:
    src: site.conf.j2
    dest: "{{ webserver_config_dir }}/sites-available/default"
    owner: root
    group: root
    mode: "0644"
  notify: Reload nginx
```

### handlers/main.yml
```yaml
---
- name: Restart nginx
  ansible.builtin.service:
    name: "{{ webserver_service_name }}"
    state: restarted

- name: Reload nginx
  ansible.builtin.service:
    name: "{{ webserver_service_name }}"
    state: reloaded

- name: Validate nginx config
  ansible.builtin.command: nginx -t
  changed_when: false
```

### meta/main.yml
```yaml
---
galaxy_info:
  author: Your Name
  description: Nginx webserver role
  company: Your Company

  license: MIT

  min_ansible_version: "2.9"

  platforms:
    - name: Ubuntu
      versions:
        - focal
        - jammy
    - name: Debian
      versions:
        - bullseye
        - bookworm
    - name: EL
      versions:
        - "7"
        - "8"
        - "9"

  galaxy_tags:
    - web
    - nginx
    - webserver

# Зависимости роли
dependencies:
  - role: common
    vars:
      common_ntp_enabled: true

  - role: firewall
    vars:
      firewall_allowed_ports:
        - 80
        - 443
```

### Использование ролей в плейбуке
```yaml
---
- name: Configure web servers
  hosts: webservers
  become: true

  roles:
    # Простое использование
    - common
    - webserver

    # С переменными
    - role: webserver
      vars:
        webserver_http_port: 8080
        webserver_enable_ssl: true

    # С тегами
    - role: monitoring
      tags: [monitoring]

    # Условное применение
    - role: backup
      when: environment == "production"
```

### Включение ролей динамически
```yaml
tasks:
  - name: Include role dynamically
    ansible.builtin.include_role:
      name: webserver
    vars:
      webserver_http_port: 8080

  - name: Import role (статически)
    ansible.builtin.import_role:
      name: database
    when: install_database
```

### requirements.yml
```yaml
---
# Роли из Ansible Galaxy
roles:
  - name: geerlingguy.nginx
    version: 3.1.4

  - name: geerlingguy.postgresql
    version: 3.3.0

  # Роль из Git
  - name: custom_role
    src: https://github.com/user/custom-role.git
    version: main

  - name: another_role
    src: git+https://github.com/user/another-role.git
    version: v1.0.0

# Коллекции
collections:
  - name: community.general
    version: 5.8.0

  - name: ansible.posix
    version: 1.5.1
```

---

## Модули

### Управление пакетами

#### apt (Debian/Ubuntu)
```yaml
- name: Install package
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: true
    cache_valid_time: 3600

- name: Install multiple packages
  ansible.builtin.apt:
    name:
      - nginx
      - postgresql
      - redis-server
    state: present

- name: Install specific version
  ansible.builtin.apt:
    name: nginx=1.18.0-0ubuntu1
    state: present

- name: Remove package
  ansible.builtin.apt:
    name: nginx
    state: absent
    purge: true
    autoremove: true
```

#### yum/dnf (RHEL/CentOS/Fedora)
```yaml
- name: Install package
  ansible.builtin.yum:
    name: nginx
    state: present

- name: Update all packages
  ansible.builtin.yum:
    name: "*"
    state: latest

- name: Install from URL
  ansible.builtin.yum:
    name: https://example.com/package.rpm
    state: present
```

#### package (универсальный)
```yaml
- name: Install package (any OS)
  ansible.builtin.package:
    name: nginx
    state: present
```

### Управление файлами

#### copy
```yaml
- name: Copy file
  ansible.builtin.copy:
    src: files/index.html
    dest: /var/www/html/index.html
    owner: www-data
    group: www-data
    mode: "0644"
    backup: true

- name: Copy with inline content
  ansible.builtin.copy:
    content: |
      Hello World
      This is a test
    dest: /tmp/test.txt
    mode: "0644"
```

#### template
```yaml
- name: Template file
  ansible.builtin.template:
    src: templates/nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: "0644"
    backup: true
    validate: nginx -t -c %s
```

#### file
```yaml
- name: Create directory
  ansible.builtin.file:
    path: /var/www/mysite
    state: directory
    owner: www-data
    group: www-data
    mode: "0755"

- name: Create file
  ansible.builtin.file:
    path: /tmp/myfile
    state: touch
    mode: "0644"

- name: Remove file/directory
  ansible.builtin.file:
    path: /tmp/myfile
    state: absent

- name: Create symlink
  ansible.builtin.file:
    src: /etc/nginx/sites-available/mysite
    dest: /etc/nginx/sites-enabled/mysite
    state: link

- name: Change ownership recursively
  ansible.builtin.file:
    path: /var/www
    owner: www-data
    group: www-data
    recurse: true
```

#### lineinfile
```yaml
- name: Add line to file
  ansible.builtin.lineinfile:
    path: /etc/hosts
    line: 192.168.1.10 server.local
    state: present

- name: Replace line matching regex
  ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    regexp: ^#?PermitRootLogin
    line: PermitRootLogin no

- name: Remove line
  ansible.builtin.lineinfile:
    path: /etc/hosts
    regexp: old-server
    state: absent

- name: Add line after pattern
  ansible.builtin.lineinfile:
    path: /etc/fstab
    line: /dev/sdb1 /mnt/data ext4 defaults 0 0
    insertafter: ^/dev/sda
```

#### blockinfile
```yaml
- name: Add block of lines
  ansible.builtin.blockinfile:
    path: /etc/ssh/sshd_config
    block: |
      PermitRootLogin no
      PasswordAuthentication no
      PubkeyAuthentication yes
    marker: "# {mark} ANSIBLE MANAGED BLOCK"

- name: Remove block
  ansible.builtin.blockinfile:
    path: /etc/ssh/sshd_config
    state: absent
    marker: "# {mark} ANSIBLE MANAGED BLOCK"
```

#### replace
```yaml
- name: Replace text in file
  ansible.builtin.replace:
    path: /etc/nginx/nginx.conf
    regexp: 'worker_processes\s+\d+;'
    replace: worker_processes auto;
```

### Управление сервисами

#### service / systemd
```yaml
- name: Start and enable service
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true

- name: Restart service
  ansible.builtin.service:
    name: nginx
    state: restarted

- name: Reload service
  ansible.builtin.service:
    name: nginx
    state: reloaded

- name: Stop and disable service
  ansible.builtin.service:
    name: nginx
    state: stopped
    enabled: false

# Systemd specific
- name: Reload systemd daemon
  ansible.builtin.systemd:
    daemon_reload: true

- name: Mask service
  ansible.builtin.systemd:
    name: nginx
    masked: true
```

### Выполнение команд

#### command
```yaml
- name: Run command
  ansible.builtin.command: /usr/bin/mycommand arg1 arg2
  args:
    chdir: /opt/myapp
    creates: /opt/myapp/output.txt
  register: command_result
  changed_when: false

- name: Command with specific user
  ansible.builtin.command: whoami
  become: true
  become_user: www-data
  changed_when: false
```

#### shell
```yaml
- name: Run shell command with pipes
  ansible.builtin.shell: |
    set -o pipefail
    ps aux | grep nginx | grep -v grep
  args:
    executable: /bin/bash
  register: nginx_processes
  changed_when: false
  failed_when: false

- name: Run multiline shell script
  ansible.builtin.shell: |
    cd /opt/myapp
    source venv/bin/activate
    python manage.py migrate
  args:
    executable: /bin/bash
  changed_when: false
```

#### script
```yaml
- name: Run local script on remote host
  ansible.builtin.script: scripts/setup.sh
  args:
    creates: /opt/myapp/.installed
```

### Работа с архивами

#### unarchive
```yaml
- name: Extract archive from local
  ansible.builtin.unarchive:
    src: files/app.tar.gz
    dest: /opt/myapp
    owner: myapp
    group: myapp

- name: Extract archive from remote
  ansible.builtin.unarchive:
    src: /tmp/app.tar.gz
    dest: /opt/myapp
    remote_src: true

- name: Extract specific files
  ansible.builtin.unarchive:
    src: files/app.tar.gz
    dest: /opt/myapp
    include:
      - config/*
      - bin/*
```

#### archive
```yaml
- name: Create archive
  community.general.archive:
    path:
      - /etc/nginx
      - /var/www
    dest: /backup/web.tar.gz
    format: gz
```

### Сеть и HTTP

#### get_url
```yaml
- name: Download file
  ansible.builtin.get_url:
    url: https://example.com/file.tar.gz
    dest: /tmp/file.tar.gz
    checksum: sha256:abc123...
    mode: "0644"

- name: Download with authentication
  ansible.builtin.get_url:
    url: https://example.com/file
    dest: /tmp/file
    url_username: user
    url_password: pass
    mode: "0644"
```

#### uri
```yaml
- name: Check URL
  ansible.builtin.uri:
    url: http://localhost
    status_code: 200

- name: POST request
  ansible.builtin.uri:
    url: https://api.example.com/users
    method: POST
    body_format: json
    body:
      name: John
      email: john@example.com
    headers:
      Authorization: "Bearer {{ api_token }}"
    status_code: 201
  register: api_response

- name: GET with return content
  ansible.builtin.uri:
    url: https://api.example.com/status
    return_content: true
  register: api_status
```

#### wait_for
```yaml
- name: Wait for port
  ansible.builtin.wait_for:
    port: 80
    delay: 5
    timeout: 300

- name: Wait for file
  ansible.builtin.wait_for:
    path: /tmp/install.lock
    state: absent

- name: Wait for string in file
  ansible.builtin.wait_for:
    path: /var/log/app.log
    search_regex: Application started
```

### Управление пользователями

#### user
```yaml
- name: Create user
  ansible.builtin.user:
    name: deploy
    comment: Deployment User
    uid: 1040
    group: deploy
    groups: sudo,docker
    append: true
    shell: /bin/bash
    create_home: true
    home: /home/deploy

- name: Remove user
  ansible.builtin.user:
    name: olduser
    state: absent
    remove: true
```

#### authorized_key
```yaml
- name: Add SSH public key
  ansible.posix.authorized_key:
    user: deploy
    state: present
    key: "{{ lookup('file', '/path/to/public_key.pub') }}"

- name: Add multiple keys
  ansible.posix.authorized_key:
    user: deploy
    state: present
    key: "{{ item }}"
  loop:
    - "{{ lookup('file', 'keys/user1.pub') }}"
    - "{{ lookup('file', 'keys/user2.pub') }}"
```

#### group
```yaml
- name: Create group
  ansible.builtin.group:
    name: developers
    gid: 1050
    state: present
```

### Git

#### git
```yaml
- name: Clone repository
  ansible.builtin.git:
    repo: https://github.com/user/repo.git
    dest: /opt/myapp
    version: main

- name: Clone with specific branch and tag
  ansible.builtin.git:
    repo: git@github.com:user/repo.git
    dest: /opt/myapp
    version: v1.0.0
    force: true

- name: Clone with authentication
  ansible.builtin.git:
    repo: https://github.com/user/private-repo.git
    dest: /opt/myapp
    version: develop
    key_file: /home/user/.ssh/id_rsa
    accept_hostkey: true
```

### Базы данных

#### mysql_db
```yaml
- name: Create database
  community.mysql.mysql_db:
    name: myapp_db
    state: present
    encoding: utf8mb4
    collation: utf8mb4_unicode_ci
    login_user: root
    login_password: "{{ mysql_root_password }}"
```

#### mysql_user
```yaml
- name: Create MySQL user
  community.mysql.mysql_user:
    name: myapp_user
    password: "{{ mysql_password }}"
    priv: 'myapp_db.*:ALL'
    host: "%"
    state: present
```

#### postgresql_db
```yaml
- name: Create PostgreSQL database
  community.postgresql.postgresql_db:
    name: myapp_db
    encoding: UTF-8
    lc_collate: en_US.UTF-8
    lc_ctype: en_US.UTF-8
    template: template0
```

#### postgresql_user
```yaml
- name: Create PostgreSQL user
  community.postgresql.postgresql_user:
    name: myapp_user
    password: "{{ postgres_password }}"
    role_attr_flags: CREATEDB,NOSUPERUSER
```

### Cron

#### cron
```yaml
- name: Add cron job
  ansible.builtin.cron:
    name: Daily backup
    minute: "0"
    hour: "2"
    job: /usr/local/bin/backup.sh
    user: root

- name: Run every 15 minutes
  ansible.builtin.cron:
    name: Check service
    minute: "*/15"
    job: /usr/local/bin/check.sh

- name: Remove cron job
  ansible.builtin.cron:
    name: Daily backup
    state: absent
```

### Другие полезные модули

#### stat
```yaml
- name: Check if file exists
  ansible.builtin.stat:
    path: /etc/myapp/config.yml
  register: config_file

- name: Use stat result
  ansible.builtin.debug:
    msg: "File exists: {{ config_file.stat.exists }}"
```

#### find
```yaml
- name: Find files
  ansible.builtin.find:
    paths: /var/log
    patterns: "*.log"
    age: 7d
    size: 100m
  register: old_logs

- name: Delete old files
  ansible.builtin.file:
    path: "{{ item.path }}"
    state: absent
  loop: "{{ old_logs.files }}"
```

#### fetch
```yaml
- name: Fetch file from remote
  ansible.builtin.fetch:
    src: /etc/nginx/nginx.conf
    dest: backups/{{ inventory_hostname }}/nginx.conf
    flat: true
```

#### synchronize (rsync)
```yaml
- name: Sync directories
  ansible.posix.synchronize:
    src: /local/path/
    dest: /remote/path/
    delete: true
    recursive: true
```

---

## Условия и циклы

### Условия (when)

#### Базовые условия
```yaml
- name: Install nginx on Ubuntu
  ansible.builtin.apt:
    name: nginx
    state: present
  when: ansible_distribution == "Ubuntu"

- name: Task for production only
  ansible.builtin.command: /opt/production-setup.sh
  when: environment == "production"
  changed_when: false

- name: Multiple conditions (AND)
  ansible.builtin.service:
    name: nginx
    state: started
  when:
    - ansible_distribution == "Ubuntu"
    - ansible_distribution_version is version('20.04', '>=')

- name: Multiple conditions (OR)
  ansible.builtin.package:
    name: httpd
    state: present
  when: ansible_distribution == "CentOS" or ansible_distribution == "RedHat"

- name: Complex conditions
  ansible.builtin.command: /usr/bin/special-command
  when: >
    (ansible_distribution == "Ubuntu" and ansible_distribution_major_version == "20")
    or
    (ansible_distribution == "Debian" and ansible_distribution_major_version == "11")
  changed_when: false
```

#### Проверка переменных
```yaml
- name: Check if variable is defined
  ansible.builtin.debug:
    msg: Variable is defined
  when: my_variable is defined

- name: Check if variable is undefined
  ansible.builtin.debug:
    msg: Variable is not defined
  when: my_variable is undefined

- name: Check if variable is true
  ansible.builtin.command: /usr/bin/enable-feature
  when: feature_enabled is true
  changed_when: false

- name: Check if variable is false
  ansible.builtin.command: /usr/bin/disable-feature
  when: feature_enabled is false
  changed_when: false

- name: Check if variable is empty
  ansible.builtin.debug:
    msg: Variable is empty
  when: my_list | length == 0

- name: Check if variable exists in list
  ansible.builtin.debug:
    msg: Found
  when: "'nginx' in packages"
```

### Циклы (loop)

#### Простые циклы
```yaml
- name: Install multiple packages
  ansible.builtin.package:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - postgresql
    - redis-server

- name: Create multiple users
  ansible.builtin.user:
    name: "{{ item }}"
    state: present
  loop:
    - alice
    - bob
    - charlie
```

#### Цикл с индексом
```yaml
- name: Create numbered directories
  ansible.builtin.file:
    path: /opt/dir{{ item }}
    state: directory
    mode: "0755"
  loop: "{{ range(1, 11) | list }}"  # 1-10

- name: Use loop variable names
  ansible.builtin.debug:
    msg: "Item {{ ansible_loop.index }}: {{ item }}"
  loop:
    - one
    - two
    - three
  loop_control:
    extended: true
```

#### Цикл по словарю
```yaml
- name: Create users with details
  ansible.builtin.user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
    shell: "{{ item.shell }}"
  loop:
    - {name: alice, groups: sudo, shell: /bin/bash}
    - {name: bob, groups: developers, shell: /bin/zsh}
    - {name: charlie, groups: operators, shell: /bin/bash}

# Альтернативный синтаксис
- name: Create users with dict2items
  ansible.builtin.user:
    name: "{{ item.key }}"
    uid: "{{ item.value.uid }}"
    groups: "{{ item.value.groups }}"
  loop: "{{ users | dict2items }}"
  vars:
    users:
      alice:
        uid: 1001
        groups: sudo
      bob:
        uid: 1002
        groups: developers
```

#### Loop control
```yaml
- name: Custom loop variable name
  ansible.builtin.debug:
    msg: "User: {{ user.name }}"
  loop:
    - {name: alice, age: 30}
    - {name: bob, age: 25}
  loop_control:
    loop_var: user  # Вместо 'item'

- name: Add pause between iterations
  ansible.builtin.command: /usr/bin/heavy-operation
  loop: "{{ servers }}"
  loop_control:
    pause: 5  # 5 секунд между итерациями
  changed_when: false

- name: Label for cleaner output
  ansible.builtin.package:
    name: "{{ item.name }}"
    state: present
  loop:
    - {name: nginx, version: "1.18"}
    - {name: postgresql, version: "13"}
  loop_control:
    label: "{{ item.name }}"  # Показывать только имя
```

#### Until loops (retry)
```yaml
- name: Wait for service to be ready
  ansible.builtin.uri:
    url: http://localhost:8080/health
    status_code: 200
  register: result
  until: result.status == 200
  retries: 10
  delay: 5  # секунд между попытками

- name: Wait for file to appear
  ansible.builtin.stat:
    path: /tmp/install.complete
  register: install_complete
  until: install_complete.stat.exists
  retries: 30
  delay: 2
```

---

## Ansible Vault

### Создание и управление
```bash
# Создать зашифрованный файл
ansible-vault create secrets.yml

# Редактировать зашифрованный файл
ansible-vault edit secrets.yml

# Зашифровать существующий файл
ansible-vault encrypt vars/passwords.yml

# Расшифровать файл
ansible-vault decrypt vars/passwords.yml

# Просмотреть содержимое
ansible-vault view secrets.yml

# Изменить пароль
ansible-vault rekey secrets.yml

# Зашифровать строку
ansible-vault encrypt_string 'secret_password' --name 'db_password'
```

### Зашифрованный файл
```yaml
# vault/secrets.yml (зашифрован)
---
db_password: supersecret
api_key: abc123xyz
ssl_key_password: strongpass
```

### Использование зашифрованных переменных
```yaml
---
- name: Deploy application
  hosts: webservers
  become: true

  vars_files:
    - vars/common.yml
    - vault/secrets.yml  # Зашифрованный файл

  tasks:
    - name: Configure database
      ansible.builtin.template:
        src: db_config.j2
        dest: /etc/myapp/db.conf
        mode: "0600"
      vars:
        db_pass: "{{ db_password }}"  # Из vault/secrets.yml
      no_log: true
```

### Inline зашифрованные переменные
```yaml
---
# vars/database.yml
db_host: localhost
db_port: 5432
db_name: myapp
db_user: admin
db_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          66386439653865663837623232626663643265373532363331653833323234643034303866663933
          3937373965363039383564623265616265313664383861660a626433306137343261376164326464
```

### Файл паролей vault
```bash
# Создать файл с паролем
echo 'my_vault_password' > .vault_pass

# Защитить файл
chmod 600 .vault_pass

# Добавить в .gitignore
echo '.vault_pass' >> .gitignore
```

### ansible.cfg для vault
```ini
[defaults]
vault_password_file = ./.vault_pass
```

### Запуск с vault
```bash
# С паролем из файла
ansible-playbook playbook.yml --vault-password-file .vault_pass

# С интерактивным вводом пароля
ansible-playbook playbook.yml --ask-vault-pass

# С несколькими vault ID
ansible-playbook playbook.yml \
  --vault-id dev@.vault_pass_dev \
  --vault-id prod@.vault_pass_prod
```

---

## Команды CLI

### ansible (ad-hoc команды)

```bash
# Базовый синтаксис
ansible <host-pattern> -m <module> -a <arguments>

# Ping всех хостов
ansible all -m ping

# Выполнить команду
ansible all -m command -a "uptime"

# Установить пакет
ansible webservers -m apt -a "name=nginx state=present" --become

# Копировать файл
ansible all -m copy -a "src=/tmp/file dest=/tmp/file"

# Управление сервисом
ansible webservers -m service -a "name=nginx state=restarted" --become

# Собрать факты
ansible all -m setup

# С дополнительными переменными
ansible webservers -m debug -a "msg='{{ http_port }}'" -e "http_port=8080"
```

### ansible-playbook

```bash
# Базовый запуск
ansible-playbook playbook.yml

# С конкретным inventory
ansible-playbook -i inventory/production.ini playbook.yml

# С переменными из командной строки
ansible-playbook playbook.yml -e "env=production http_port=8080"

# Dry-run (проверка без изменений)
ansible-playbook playbook.yml --check

# Diff mode (показать изменения)
ansible-playbook playbook.yml --check --diff

# С тегами
ansible-playbook playbook.yml --tags "configuration,ssl"
ansible-playbook playbook.yml --skip-tags "monitoring"

# Лимит хостов
ansible-playbook playbook.yml --limit webservers

# Синтаксис-проверка
ansible-playbook playbook.yml --syntax-check

# Verbose mode
ansible-playbook playbook.yml -v     # verbose
ansible-playbook playbook.yml -vvv   # even more verbose

# С vault
ansible-playbook playbook.yml --ask-vault-pass
ansible-playbook playbook.yml --vault-password-file .vault_pass
```

### ansible-lint (v25)

```bash
# Проверить плейбук
ansible-lint playbook.yml

# Проверить всю директорию
ansible-lint .

# Проверить роль
ansible-lint roles/webserver/

# С конкретным профилем
ansible-lint --profile production playbook.yml
ansible-lint --profile safety playbook.yml

# Игнорировать конкретные правила
ansible-lint -x yaml[line-length] playbook.yml

# Вывести список правил
ansible-lint -L

# Вывести список профилей