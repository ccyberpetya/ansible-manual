# Ansible: справочник по лабораторным работам


## Содержание

- [Как пользоваться справочником](#start)
- [Основные понятия и структура YAML](#basics)
- [1. Подключение и ad-hoc команды](#lab01)
- [2. Первый playbook и идемпотентность](#lab02)
- [3. Несколько серверов и шаблоны Jinja](#lab03)
- [4. Роли, defaults, handlers и include_tasks](#lab04)
- [5. Facts, register, циклы, условия и ошибки](#lab05)
- [6. Tags, check mode и Vault](#lab06)
- [7. Группы, group_vars, serial и делегирование](#lab07)
- [8. Итоговый проект: web, app и db](#lab08)
- [Команды Ansible с назначением](#commands)
- [Таблица модулей и ключевых слов](#lookup)
- [Частые ошибки и диагностика](#troubleshooting)
- [Дополнения: host_vars, Galaxy, Molecule, AWX](#extra)
- [Проверка документа и источники](#sources)

<a id="start"></a>


<a id="basics"></a>
## Основные понятия и структура YAML

| Понятие | Что это |
| --- | --- |
| Control node | Машина, с которой запускается Ansible |
| Managed node / target | Сервер, на котором выполняются действия |
| Inventory | Хосты, группы и параметры подключения |
| Module | Готовое действие: файл, пользователь, сервис, HTTP-запрос |
| Task | Вызов модуля с параметрами и условиями |
| Play | Выбор хостов и набор задач/ролей для них |
| Playbook | YAML-файл со списком plays |
| Role | Переиспользуемые tasks, defaults, templates, handlers и другие файлы |
| Handler | Задача, запускаемая после уведомления об изменении |
| Facts | Собранные сведения об ОС и хосте |
| Idempotence | Повторное применение того же желаемого состояния не создаёт лишних изменений |

Полный playbook начинается со списка plays; файл `tasks/main.yml` роли — сразу со списка tasks. Файл переменных — словарь, без `hosts` и `tasks`.

```yaml
---
- name: Example play                 # Название play для вывода
  hosts: targets                     # Группа из inventory
  gather_facts: false                 # Не запускать автоматический сбор facts
  become: true                       # Выполнять задачи с повышением привилегий
  vars:                              # Переменные этого play
    config_path: /tmp/example.conf
  tasks:                             # Список задач
    - name: Write configuration
      ansible.builtin.copy:          # Модуль; полное имя называется FQCN
        dest: "{{ config_path }}"    # Параметры модуля вложены на один уровень
        content: "enabled=true\n"    # Строка, которую нужно записать
        mode: "0644"                 # Права задаём строкой в кавычках
      tags:                          # Свойство задачи, на уровне модуля
        - config
```

`tags`, `when`, `register`, `notify`, `loop`, `changed_when` и `no_log` — свойства **задачи**, а не параметры `copy`/`file`. У них тот же отступ, что у `ansible.builtin.copy`.

<a id="lab01"></a>
## 1. Подключение и ad-hoc команды

**Цель:** разобраться в inventory, подключиться к target и управлять файлами без playbook.

### Файл `inventory.ini`

```ini
[targets]
target1 ansible_host=192.0.2.11
target2 ansible_host=192.0.2.12

[all:vars]
ansible_user=root
ansible_ssh_private_key_file=/root/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3
```

`target1` — имя Ansible, а `ansible_host` — реальный адрес подключения. `[all:vars]` задаёт параметры всем хостам. Для стенда с одной VM убери строку `target2`.

В исходнике было отключение проверки SSH host key. Здесь проверка остаётся включённой: ключи хостов должны быть заранее проверены и добавлены в `known_hosts`. Только для одноразового изолированного стенда допустимо осознанно добавить `ansible_host_key_checking=False`.

### Команды и последовательность

```bash
# Показать хосты группы без подключения к ним
ansible -i inventory.ini targets --list-hosts

# Проверить SSH, аутентификацию и возможность запустить Python
ansible -i inventory.ini targets -m ansible.builtin.ping

# Узнать системное имя конкретного target
ansible -i inventory.ini target1 -m ansible.builtin.command -a 'hostname'

# Удалить файл, созданный при ручной SSH-части исходной лабораторной
ansible -i inventory.ini target1 -m ansible.builtin.file -a 'path=/tmp/manual-ssh-note.txt state=absent'

# Создать управляемый файл с заданным содержимым
ansible -i inventory.ini target1 -m ansible.builtin.copy -a 'dest=/tmp/ansible-managed.txt content=managed-by-ansible mode=0644'

# Прочитать его через Ansible
ansible -i inventory.ini target1 -m ansible.builtin.command -a 'cat /tmp/ansible-managed.txt'

# Удалить файл; повтор команды не должен вносить изменения
ansible -i inventory.ini target1 -m ansible.builtin.file -a 'path=/tmp/ansible-managed.txt state=absent'
```

В ручной части курса сначала подключаются по SSH к IP из inventory, проверяют, что это target, и создают `/tmp/manual-ssh-note.txt` со строкой `manual-ssh`. Это упражнение показывает разницу между control и target; отдельные команды SSH и оболочки здесь не дублируются.

**Самопроверка:** `ping` возвращает `pong`; `copy` создаёт файл именно на target; повторное удаление показывает `changed: false`. Модуль `ping` **не ICMP ping**. `command` по умолчанию может показать `changed: true` даже при чтении файла — это не доказательство изменения.

<a id="lab02"></a>
## 2. Первый playbook и идемпотентность

**Цель:** описать состояние каталога и файлов, проверить синтаксис, применить и повторить запуск.

Начальный каркас из задания можно записать как один play с `tasks: []`. Ниже сразу полная итоговая версия, включая добавленный в конце marker.

### Файл `site.yml`

```yaml
---
- name: Configure target
  hosts: targets
  gather_facts: false
  tasks:
    - name: Create application directory
      ansible.builtin.file:
        path: /opt/ansible-demo
        state: directory             # Создать каталог, если его ещё нет
        owner: root
        group: root
        mode: "0755"

    - name: Write application environment
      ansible.builtin.copy:
        dest: /opt/ansible-demo/app.env
        content: |                   # Многострочный текст с сохранением переносов
          APP_NAME=ansible-demo
          APP_PORT=8080
        owner: root
        group: root
        mode: "0644"

    - name: Mark Ansible ownership
      ansible.builtin.copy:
        dest: /opt/ansible-demo/managed-by.txt
        content: "ansible-playbook\n"
        mode: "0644"
```

### Запуск и проверка

```bash
# Проверить синтаксис; задачи не выполняются
ansible-playbook -i inventory.ini site.yml --syntax-check

# Оценить изменения и увидеть разницу содержимого
ansible-playbook -i inventory.ini site.yml --check --diff

# Применить состояние
ansible-playbook -i inventory.ini site.yml

# Проверить содержимое на target
ansible -i inventory.ini target1 -m ansible.builtin.command -a 'cat /opt/ansible-demo/app.env'

# Проверить свойства каталога
ansible -i inventory.ini target1 -m ansible.builtin.stat -a 'path=/opt/ansible-demo'

# Повторить применение для проверки идемпотентности
ansible-playbook -i inventory.ini site.yml
```

**Ожидается:** каталог с правами `0755`, два файла с правами `0644`; при неизменных входных данных повторный запуск — `changed=0`, `failed=0`, `unreachable=0`.

**Ограничение check mode:** на совершенно чистой VM задача `file` только прогнозирует создание каталога. Следующая `copy` может обнаружить, что каталога ещё нет, и завершиться ошибкой. Это ограничение симуляции связанных действий. Полный реальный запуск на тестовом хосте всё равно нужен. `--syntax-check` также не проверяет наличие файлов и доступность серверов.

<a id="lab03"></a>
## 3. Несколько серверов и шаблоны Jinja

**Цель:** один playbook создаёт разные конфиги на `target1` и `target2`.

### Файл `node.conf.j2`

```jinja2
node={{ inventory_hostname }}
port={{ app_port }}
```

`inventory_hostname` — имя текущего хоста из inventory. `app_port` — переменная play. Шаблон содержит ровно две непустые строки, как требует задание.

### Файл `multi.yml`

```yaml
---
- name: Configure multiple targets
  hosts: targets
  gather_facts: false
  vars:
    app_port: 9090
  tasks:
    - name: Render node configuration
      ansible.builtin.template:
        src: node.conf.j2            # Исходный шаблон на control node
        dest: /etc/devops404-node.conf # Готовый файл на текущем target
        owner: root
        group: root
        mode: "0644"
```

### Запуск и проверка

```bash
ansible -i inventory.ini targets --list-hosts
ansible -i inventory.ini targets -m ansible.builtin.command -a 'hostname -I'
ansible-playbook -i inventory.ini multi.yml --syntax-check
ansible-playbook -i inventory.ini multi.yml
ansible -i inventory.ini targets -m ansible.builtin.command -a 'cat /etc/devops404-node.conf'

# Создать одинаковый marker сразу на всех targets
ansible -i inventory.ini targets -m ansible.builtin.copy -a 'dest=/tmp/all-targets-marker content=multi-target mode=0644'

# Повторить: на обоих хостах должно быть changed=false
ansible -i inventory.ini targets -m ansible.builtin.copy -a 'dest=/tmp/all-targets-marker content=multi-target mode=0644'
```

В исходной ручной части нужно сверить адрес второй VM через SSH с результатом `hostname -I`. При нескольких интерфейсах вывод может содержать несколько адресов.

**Ожидаемые конфиги:**

| Хост | Первая строка | Вторая строка |
| --- | --- | --- |
| target1 | `node=target1` | `port=9090` |
| target2 | `node=target2` | `port=9090` |

`copy: content:` тоже подставляет переменные Jinja. `template` удобен, когда содержимое вынесено в отдельный файл, содержит условия, циклы или много строк.

<a id="lab04"></a>
## 4. Роли, defaults, handlers и include_tasks

**Цель:** собрать самостоятельную роль `demo_app` и переопределять её настройки из playbook.

### Состав роли

| Файл | Назначение |
| --- | --- |
| `roles/demo_app/defaults/main.yml` | Значения по умолчанию |
| `roles/demo_app/tasks/main.yml` | Точка входа задач |
| `roles/demo_app/tasks/marker.yml` | Дополнительный список задач |
| `roles/demo_app/templates/app.conf.j2` | Исходный шаблон |
| `roles/demo_app/handlers/main.yml` | Обработчик изменения |

### Файл `roles/demo_app/defaults/main.yml`

```yaml
---
demo_app_name: demo-app
demo_app_port: 8080
demo_app_dir: /opt/demo-app
```

Это словарь переменных. Здесь не нужны `- name:`, `hosts:` и `tasks:`. Префикс `demo_app_` снижает вероятность конфликта имён с другими ролями.

### Файл `roles/demo_app/templates/app.conf.j2`

```jinja2
name={{ demo_app_name }}
port={{ demo_app_port }}
```

### Файл `roles/demo_app/tasks/main.yml`

```yaml
---
- name: Create demo directory
  ansible.builtin.file:
    path: "{{ demo_app_dir }}"       # Значение целиком из Jinja заключаем в кавычки
    state: directory
    mode: "0755"

- name: Render demo configuration
  ansible.builtin.template:
    src: app.conf.j2                 # Ansible ищет в templates этой роли
    dest: "{{ demo_app_dir }}/app.conf"
    mode: "0644"
  notify: Restart demo service       # Уведомление только при changed

- name: Include marker tasks
  ansible.builtin.include_tasks: marker.yml # Динамически подключить список задач
```

### Файл `roles/demo_app/tasks/marker.yml`

```yaml
---
- name: Create included marker
  ansible.builtin.copy:
    dest: "{{ demo_app_dir }}/included.txt"
    content: included-task
    mode: "0644"
```

### Файл `roles/demo_app/handlers/main.yml`

```yaml
---
- name: Restart demo service         # Имя должно совпадать с notify
  ansible.builtin.debug:
    msg: demo service restarted
```

Здесь handler только печатает сообщение: реальной службы в этой лабораторной нет. Если несколько задач уведомляют один handler, до ближайшей точки выполнения handlers он ставится в очередь один раз. Обычно обработчики выполняются после секции tasks/roles. Для немедленного исполнения накопленных уведомлений используется `ansible.builtin.meta: flush_handlers`.

### Файл `role-site.yml`

```yaml
---
- name: Apply demo role
  hosts: targets
  gather_facts: false
  roles:
    - demo_app
```

### Файл `role-site-override.yml`

```yaml
---
- name: Apply demo role with overrides
  hosts: targets
  gather_facts: false
  vars:                             # Эти значения приоритетнее role defaults
    demo_app_name: override-demo
    demo_app_port: 9091
  roles:
    - demo_app
```

### Запуск и проверка

```bash
ansible-playbook -i inventory.ini role-site.yml --syntax-check
ansible-playbook -i inventory.ini role-site.yml
ansible -i inventory.ini target1 -m ansible.builtin.command -a 'cat /opt/demo-app/app.conf'

ansible-playbook -i inventory.ini role-site-override.yml --syntax-check
ansible-playbook -i inventory.ini role-site-override.yml
ansible -i inventory.ini target1 -m ansible.builtin.command -a 'cat /opt/demo-app/app.conf'
ansible -i inventory.ini target1 -m ansible.builtin.command -a 'cat /opt/demo-app/included.txt'

# Повторять нужно ту же версию playbook, чтобы оценивать идемпотентность
ansible-playbook -i inventory.ini role-site-override.yml
```

**Самопроверка:** базовый запуск записывает `demo-app`/`8080`; override — `override-demo`/`9091`. При смене конфигурации появляется сообщение handler. Повтор override без изменений не должен его вызывать. Чередование двух playbooks меняет желаемое состояние и закономерно даёт `changed`.

Чтобы воспроизвести исходную работу строго по шагам, сначала выполни роль без последней задачи `include_tasks`, затем добавь `marker.yml` и подключение. Итоговая версия выше уже содержит все изменения.

<a id="lab05"></a>
## 5. Facts, register, циклы, условия и ошибки

**Цель:** получать данные от хоста, использовать результаты задач, повторять действия и обрабатывать предусмотренные ошибки.

### Файл `facts.yml`

```yaml
---
- name: Explore facts and task control
  hosts: targets
  gather_facts: true                 # Автоматически вызывает setup
  tasks:
    - name: Save distribution name
      ansible.builtin.copy:
        dest: /tmp/ansible-distribution.txt
        content: "{{ ansible_facts.distribution }}\n"
        mode: "0644"

    - name: Read kernel release
      ansible.builtin.command: uname -r
      register: kernel_result       # Сохранить словарь результата, не только stdout
      changed_when: false           # Чтение не меняет хост
      check_mode: false             # Безопасное чтение выполнить даже с --check

    - name: Save kernel release
      ansible.builtin.copy:
        dest: /tmp/kernel-release.txt
        content: "{{ kernel_result.stdout }}\n"
        mode: "0644"

    - name: Create application directories
      ansible.builtin.file:
        path: "/opt/loop-demo/{{ item }}"
        state: directory
        mode: "0755"
      loop:                         # Одна задача, три значения item
        - app
        - logs
        - data

    - name: Write Ubuntu marker
      ansible.builtin.copy:
        dest: /tmp/ubuntu-only.txt
        content: ubuntu-target
        mode: "0644"
      when: ansible_facts.distribution == "Ubuntu" # Условие без {{ }}

    - name: Create keep files
      ansible.builtin.copy:
        dest: "/opt/loop-demo/{{ item }}/.keep"
        content: created-by-loop
        mode: "0644"
      loop:
        - app
        - logs
        - data

    - name: Handle optional configuration
      block:                        # Основная группа задач
        - name: Read optional configuration
          ansible.builtin.command: cat /tmp/optional-app.conf
          changed_when: false
      rescue:                       # Выполнить после failed в block
        - name: Record recovery
          ansible.builtin.copy:
            dest: /tmp/ansible-rescued
            content: recovered
            mode: "0644"
      always:                       # Выполнить после block/rescue при обычном ходе
        - name: Record completion
          ansible.builtin.copy:
            dest: /tmp/ansible-always
            content: finished
            mode: "0644"

    - name: Read operating system release
      ansible.builtin.command: cat /etc/os-release
      register: os_release_check
      changed_when: false
      failed_when: os_release_check.rc != 0 # Ненулевой код считать ошибкой

    - name: Wait until the third attempt
      ansible.builtin.shell: |
        attempt=$(cat /tmp/ansible-attempts 2>/dev/null || echo 0)
        attempt=$((attempt + 1))
        echo "$attempt" > /tmp/ansible-attempts
        test "$attempt" -ge 3
      register: retry_result
      until: retry_result.rc == 0    # Повторять до успешной проверки
      retries: 3                    # Лимит повторов после первоначальной попытки
      delay: 1                      # Пауза между попытками, секунды
```

### Как читать результат задачи

| Поле | Значение |
| --- | --- |
| `kernel_result.stdout` | Стандартный вывод команды одной строкой |
| `kernel_result.stdout_lines` | Вывод как список строк |
| `os_release_check.rc` | Код завершения: обычно 0 — успех |
| `os_release_check.stderr` | Стандартный поток ошибок |
| `retry_result.attempts` | Число выполненных попыток в задаче с until |
| `result.changed` | Сообщила ли задача об изменении |

`register` относится к конкретному хосту. При `register` на задаче с `loop` отдельные результаты находятся в `result.results`.

### Запуск и проверка

```bash
# Гарантировать демонстрационную ошибку в block на учебном стенде
ansible -i inventory.ini targets -m ansible.builtin.file -a 'path=/tmp/optional-app.conf state=absent'

# Сбросить счётчик, чтобы вновь увидеть две неудачные попытки
ansible -i inventory.ini targets -m ansible.builtin.file -a 'path=/tmp/ansible-attempts state=absent'

ansible-playbook -i inventory.ini facts.yml --syntax-check
ansible-playbook -i inventory.ini facts.yml

# Посмотреть исходные facts дистрибутива
ansible -i inventory.ini target1 -m ansible.builtin.setup -a 'filter=ansible_distribution*'

# Проверить записанные данные
ansible -i inventory.ini targets -m ansible.builtin.command -a 'cat /tmp/ansible-distribution.txt /tmp/kernel-release.txt'

# Shell нужен здесь для конвейера | sort
ansible -i inventory.ini target1 -m ansible.builtin.shell -a 'find /opt/loop-demo -maxdepth 2 -type f | sort'

ansible -i inventory.ini targets -m ansible.builtin.command -a 'cat /tmp/ansible-rescued /tmp/ansible-always /tmp/ansible-attempts'
```

**Ожидается:** название установленного дистрибутива (на стенде курса — `Ubuntu`), текущая версия ядра, три `.keep`, marker для Ubuntu, файлы `recovered` и `finished`. С чистым счётчиком shell успешно завершится на третьем запуске.

**Важные нюансы:**

- При успешно выполненном `rescue` в выводе остаётся исходная ошибка задачи, но play может завершиться успешно; в recap увеличивается `rescued`. `rescue` не исправляет синтаксические ошибки и `UNREACHABLE`.
- `always` не означает «выполнится при любой аварии»: недоступность хоста или некорректная структура могут помешать выполнению блока.
- `failed_when: rc != 0` для обычной `command` повторяет стандартное поведение; здесь оно показано для обучения. Пользовательские условия нужны, когда код возврата имеет особый смысл.
- Временно удали `changed_when: false` у чтения `/etc/os-release`, сравни статус и верни параметр. `changed_when` управляет отчётом, а не реальным поведением команды.
- Счётчик попыток намеренно **не идемпотентен**: он записывает новое значение при каждом запуске. Не требуй `changed=0` от всей этой лабораторной. Для реального ожидания обычно проверяют `uri`, `wait_for` или результат команды чтения.
- `retries` — бюджет повторов, а не гарантия ровно указанного числа запусков. С `until` и `retries: 3` следует учитывать первоначальную попытку и до трёх повторов; успешное условие останавливает цикл раньше. В исходном тексте это было упрощено до «до трёх раз».

<a id="lab06"></a>
## 6. Tags, check mode и Vault

**Цель:** запускать выбранные части playbook и передавать зашифрованные переменные.

### Этап 1. Файл `tagged.yml` без Vault

```yaml
---
- name: Configure selected parts
  hosts: targets
  gather_facts: false
  tasks:
    - name: Write tagged configuration
      ansible.builtin.copy:
        dest: /etc/devops404-tagged.conf
        content: tag=config
        mode: "0644"
      tags:                         # На уровне задачи, не внутри copy
        - config

    - name: Write tagged marker
      ansible.builtin.copy:
        dest: /tmp/devops404-marker
        content: tag=marker
        mode: "0644"
      tags:
        - marker
```

```bash
# Проверить синтаксис и сохранить вывод для исходной проверки курса
ansible-playbook -i inventory.ini tagged.yml --syntax-check > tagged.syntax.txt

ansible-playbook -i inventory.ini tagged.yml --check --diff
ansible-playbook -i inventory.ini tagged.yml --list-tags
ansible-playbook -i inventory.ini tagged.yml --list-tasks

# По очереди применить независимые части
ansible-playbook -i inventory.ini tagged.yml --tags config
ansible-playbook -i inventory.ini tagged.yml --tags marker

# Очистить оба учебных файла на target1, как в задании
ansible -i inventory.ini target1 -m ansible.builtin.file -a 'path=/etc/devops404-tagged.conf state=absent'
ansible -i inventory.ini target1 -m ansible.builtin.file -a 'path=/tmp/devops404-marker state=absent'
```

Чтобы проверить, что `--tags config` не создаёт marker, начни с хоста без этих файлов. Теги не удаляют результат прошлых запусков. Для очистки всей группы замени `target1` на `targets`.

### Этап 2. Зашифровать переменную

Исходное открытое содержимое **`secret-vars.yml`**:

```yaml
---
demo_secret: vault-protected
```

В исходном задании `.vault-pass` содержит учебный пароль `devops404-lab`. Этот пароль публично известен из задания и подходит только для упражнения. Файл пароля создаётся на control node с правами `0600`; для собственного проекта используй иной пароль и не добавляй `.vault-pass` в Git.

```bash
# Зашифровать файл на месте
ansible-vault encrypt secret-vars.yml --vault-password-file .vault-pass

# Просмотреть расшифрованное содержимое без записи открытой копии на диск
ansible-vault view secret-vars.yml --vault-password-file .vault-pass

# Альтернатива: шифрование с интерактивным вводом пароля вместо .vault-pass
# Этот вариант выполняется вместо первой команды, пока файл ещё открыт
ansible-vault encrypt secret-vars.yml
```

После шифрования содержимое начинается с `$ANSIBLE_VAULT;...`. Команда `view` в исходнике была обрезана — выше приведён полный вариант. Не выполняй оба варианта `encrypt` подряд над уже зашифрованным файлом.

### Этап 3. Итоговый файл `tagged.yml`

Это **замена** версии этапа 1, а не второй playbook для добавления в её конец.

```yaml
---
- name: Configure selected parts and secrets
  hosts: targets
  gather_facts: false
  vars_files:                       # Уровень play, рядом с hosts и tasks
    - secret-vars.yml
  tasks:
    - name: Write tagged configuration
      ansible.builtin.copy:
        dest: /etc/devops404-tagged.conf
        content: tag=config
        mode: "0644"
      tags: [config]

    - name: Write tagged marker
      ansible.builtin.copy:
        dest: /tmp/devops404-marker
        content: tag=marker
        mode: "0644"
      tags: [marker]

    - name: Create secret directory
      ansible.builtin.file:
        path: /etc/devops404
        state: directory
        owner: root
        group: root
        mode: "0755"
      tags: [secret]                # Каталог и файл выбираются одним тегом

    - name: Write protected secret
      ansible.builtin.copy:
        dest: /etc/devops404/secret.conf
        content: "{{ demo_secret }}\n"
        owner: root
        group: root
        mode: "0600"
      no_log: true                  # Не показывать параметры/результат задачи
      diff: false                   # Не выводить разницу секретного файла
      tags: [secret]
```

```bash
ansible-playbook -i inventory.ini tagged.yml --syntax-check --vault-password-file .vault-pass
ansible-playbook -i inventory.ini tagged.yml --tags secret --vault-password-file .vault-pass

# Проверить права и существование, не печатая секрет
ansible -i inventory.ini target1 -m ansible.builtin.stat -a 'path=/etc/devops404/secret.conf'

# Вариант запуска без файла пароля
ansible-playbook -i inventory.ini tagged.yml --tags secret --ask-vault-pass
```

**Самопроверка:** у secret-файла владелец `root`, права `0600`; повторное применение secret-части без изменения значения не должно менять файл. После добавления зашифрованного `vars_files` передавай Vault-пароль и для запусков с другими тегами: файл переменных может загружаться до отбора задач.

**Границы защиты:** Vault шифрует сохранённый исходник; на target `copy` записывает обычный расшифрованный файл. Его защищают права ОС. `no_log` скрывает вывод этой задачи, но не защитит от отдельного `debug` или `cat` секрета.

<a id="lab07"></a>
## 7. Группы, group_vars, serial и делегирование

**Цель:** назначить двум хостам разные функции, применить общую логику с разными переменными и управлять порядком выполнения.

### Файл `grouped.ini`

```ini
[web]
target1 ansible_host=192.0.2.11 ansible_user=root

[database]
target2 ansible_host=192.0.2.12 ansible_user=root

[managed:children]
web
database

[all:vars]
ansible_ssh_private_key_file=/root/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3
```

`[managed:children]` перечисляет **дочерние группы**, а не имена серверов.

### Файл `group_vars/web.yml`

```yaml
---
service_role: web
service_port: 8081
```

### Файл `group_vars/database.yml`

```yaml
---
service_role: database
service_port: 5432
```

Имена файлов соответствуют группам inventory. Храни `group_vars` рядом с inventory или playbook; в этом справочнике они находятся в одном проекте.

### Файл `node-vars.yml`

```yaml
---
- name: Configure grouped nodes
  hosts: managed
  gather_facts: false
  tasks:
    - name: Write service configuration
      ansible.builtin.copy:
        dest: /etc/devops404-service.conf
        content: |
          name={{ inventory_hostname }}
          role={{ service_role }}
          port={{ service_port }}
        mode: "0644"

    - name: Write common marker
      ansible.builtin.copy:
        dest: /etc/devops404-common.conf
        content: managed_by=ansible
        mode: "0644"

    - name: Write managed status
      ansible.builtin.copy:
        dest: /etc/devops404-status.conf
        content: status=managed
        mode: "0644"
```

```bash
ansible-inventory -i grouped.ini --graph
ansible -i grouped.ini managed -m ansible.builtin.ping
ansible-playbook -i grouped.ini node-vars.yml --syntax-check
ansible-playbook -i grouped.ini node-vars.yml --list-hosts
ansible-playbook -i grouped.ini node-vars.yml
ansible -i grouped.ini managed -m ansible.builtin.command -a 'cat /etc/devops404-service.conf'

# Посмотреть и затем применить только к веб-группе
ansible-playbook -i grouped.ini node-vars.yml --list-hosts --limit web
ansible-playbook -i grouped.ini node-vars.yml --limit web
```

**Ожидается:** `target1` получает `web`/`8081`, `target2` — `database`/`5432`. Общие marker/status одинаковы. `--limit` сужает выборку `hosts`, но не добавляет в play хосты вне неё.

### Файл `rolling.yml`

```yaml
---
- name: Demonstrate rollout batches
  hosts: managed
  gather_facts: false
  serial: 1                         # Для первого эксперимента убери эту строку
  tasks:
    - name: Record current batch
      ansible.builtin.copy:
        dest: /etc/devops404-batch.conf
        content: |
          batch={{ ansible_play_batch | join(',') }}
        mode: "0644"
```

Сначала запусти без `serial`, прочитай файл. Затем верни `serial: 1` и повтори обе команды:

```bash
ansible-playbook -i grouped.ini rolling.yml
ansible -i grouped.ini managed -m ansible.builtin.command -a 'cat /etc/devops404-batch.conf'
```

Без `serial` текущая партия содержит два активных хоста; порядок имён не следует считать контрактом. С `serial: 1` у каждого — только собственное имя. Ansible завершает play для одной партии и переходит к следующей.

### Файл `summary.yml`

```yaml
---
- name: Create orchestration summary
  hosts: managed
  gather_facts: false
  tasks:
    - name: Write summary on control node
      ansible.builtin.copy:
        dest: "{{ playbook_dir }}/orchestration-summary.txt"
        content: |
          managed={{ ansible_play_hosts | sort | join(',') }}
        mode: "0644"
      run_once: true                # Одна задача для текущей партии
      delegate_to: localhost        # Модуль выполняется на control node
      become: false                 # Не повышать права локально
```

```bash
ansible-playbook -i grouped.ini summary.yml

# Прочитать локальный результат через Ansible
ansible localhost -i localhost, -c local -m ansible.builtin.command -a 'cat orchestration-summary.txt'
```

В исходном стенде файл лежал в `/root/ansible`. `playbook_dir` сохраняет его рядом с playbook независимо от расположения проекта.

**Ожидается:** `managed=target1,target2`. В этом play нет `serial`, поэтому `run_once` даёт один запуск. При сочетании с `serial: 1` задача с `run_once` обычно выполняется один раз **на каждую партию**, а не один раз на весь play. Сводку поэтому удобнее держать в отдельном play без разбиения на партии.

<a id="lab08"></a>
## 8. Итоговый проект: web, app и db

**Цель:** развернуть три назначения через отдельные роли, проверить службы, настроить кэш facts, повторить запуск и собрать сводку.

Это три независимых учебных компонента: web раздаёт статическую страницу, app отвечает по HTTP, db создаёт локальную SQLite-базу. В исходном задании нет проксирования web → app или соединения app → db; SQLite не слушает сетевой порт.

### Файлы проекта

| Путь | Назначение |
| --- | --- |
| `inventory.ini`, `ansible.cfg` | Хосты и настройки Ansible |
| `group_vars/all.yml`, `web.yml`, `app.yml`, `db.yml` | Общие и групповые значения |
| `roles/common/tasks/main.yml` | Общая подготовка |
| `roles/web/tasks/main.yml`, `handlers/main.yml`, `templates/*` | Статический HTTP-сервер |
| `roles/app/tasks/main.yml`, `handlers/main.yml`, `templates/*` | Python HTTP-приложение |
| `roles/db/tasks/main.yml`, `templates/init_db.py.j2` | Инициализация SQLite |
| `site.yml`, `rolling.yml`, `summary.yml` | Общий запуск и оркестрация |
| `.facts/` | Локальный кэш, не исходный код |

### 8.1. Файл `inventory.ini`

```ini
[targets]
target1 ansible_host=192.0.2.11
target2 ansible_host=192.0.2.12
target3 ansible_host=192.0.2.13

[web]
target1

[app]
target2

[db]
target3

[managed:children]
web
app
db

[all:vars]
ansible_user=root
ansible_ssh_private_key_file=/root/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3
```

Повторное упоминание хоста в группе не создаёт вторую VM: Ansible связывает записи по имени inventory.

### Файл `group_vars/all.yml`

```yaml
---
base_dir: /opt/devops404
service_user: devopsdemo
```

### Файл `group_vars/web.yml`

```yaml
---
web_port: 8080
```

### Файл `group_vars/app.yml`

```yaml
---
app_port: 8081
```

### Файл `group_vars/db.yml`

```yaml
---
database_file: /opt/devops404/db/app.db
```

### 8.2. Файл `roles/common/tasks/main.yml`

```yaml
---
- name: Check Python runtime
  ansible.builtin.command: /usr/bin/python3 --version
  changed_when: false

- name: Create service group
  ansible.builtin.group:             # Явно создаём группу, не полагаемся на useradd
    name: "{{ service_user }}"
    system: true

- name: Create service user
  ansible.builtin.user:
    name: "{{ service_user }}"
    group: "{{ service_user }}"
    system: true                    # Системная учётная запись для сервиса
    create_home: false
    shell: /usr/sbin/nologin

- name: Create base directory
  ansible.builtin.file:
    path: "{{ base_dir }}"
    state: directory
    owner: "{{ service_user }}"
    group: "{{ service_user }}"
    mode: "0755"
```

Проверка `python3 --version` не устанавливает Python: он должен быть доступен уже для facts и большинства модулей.

### 8.3. Файл `roles/web/tasks/main.yml`

```yaml
---
- name: Create web directory
  ansible.builtin.file:
    path: "{{ base_dir }}/web"
    state: directory
    owner: "{{ service_user }}"
    group: "{{ service_user }}"
    mode: "0755"

- name: Render web page
  ansible.builtin.template:
    src: index.html.j2
    dest: "{{ base_dir }}/web/index.html"
    owner: "{{ service_user }}"
    group: "{{ service_user }}"
    mode: "0644"

- name: Install web service
  ansible.builtin.template:
    src: devops404-web.service.j2
    dest: /etc/systemd/system/devops404-web.service
    owner: root
    group: root
    mode: "0644"
  notify: Restart web service

- name: Enable and start web service
  ansible.builtin.systemd_service:
    name: devops404-web
    enabled: true                   # Запускать при загрузке ОС
    state: started                  # Запустить, если остановлен
    daemon_reload: true             # Перечитать unit-файлы systemd

- name: Apply pending web restart before checking readiness
  ansible.builtin.meta: flush_handlers

- name: Wait for web port
  ansible.builtin.wait_for:
    host: 127.0.0.1                  # Проверка выполняется на target1
    port: "{{ web_port }}"
    timeout: 30
  when: not ansible_check_mode
```

### Файл `roles/web/templates/index.html.j2`

```jinja2
Web node {{ inventory_hostname }}
```

### Файл `roles/web/templates/devops404-web.service.j2`

```ini
[Unit]
Description=DevOps404 web service
After=network.target

[Service]
User={{ service_user }}
ExecStart=/usr/bin/python3 -m http.server {{ web_port }} --directory {{ base_dir }}/web
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

`ExecStart` запускает встроенный Python HTTP-сервер. `User` — учётная запись процесса, `Restart` — политика systemd при аварии. Это учебная служба, без настройки TLS и промышленного веб-сервера.

### Файл `roles/web/handlers/main.yml`

```yaml
---
- name: Restart web service
  ansible.builtin.systemd_service:
    name: devops404-web
    state: restarted
    daemon_reload: true
```

Здесь внесено исправление исходника: `flush_handlers` расположен **перед** `wait_for`. При смене порта работающий процесс продолжает слушать старый порт до перезапуска; без flush проверка нового порта могла бы завершиться тайм-аутом до выполнения handler. Изменение статической страницы не требует перезапуска.

### 8.4. Файл `roles/app/tasks/main.yml`

```yaml
---
- name: Create app directory
  ansible.builtin.file:
    path: "{{ base_dir }}/app"
    state: directory
    owner: "{{ service_user }}"
    group: "{{ service_user }}"
    mode: "0755"

- name: Render application
  ansible.builtin.template:
    src: app.py.j2
    dest: "{{ base_dir }}/app/app.py"
    owner: "{{ service_user }}"
    group: "{{ service_user }}"
    mode: "0755"
  notify: Restart app service

- name: Install app service
  ansible.builtin.template:
    src: devops404-app.service.j2
    dest: /etc/systemd/system/devops404-app.service
    owner: root
    group: root
    mode: "0644"
  notify: Restart app service

- name: Enable and start app service
  ansible.builtin.systemd_service:
    name: devops404-app
    enabled: true
    state: started
    daemon_reload: true

- name: Apply pending app restart before checking readiness
  ansible.builtin.meta: flush_handlers

- name: Wait for app port
  ansible.builtin.wait_for:
    host: 127.0.0.1
    port: "{{ app_port }}"
    timeout: 30
  when: not ansible_check_mode
```

### Файл `roles/app/templates/app.py.j2`

```python
from http.server import BaseHTTPRequestHandler, HTTPServer


class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        body = b"application is running"
        self.send_response(200)
        self.send_header("Content-Type", "text/plain; charset=utf-8")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)


HTTPServer(("0.0.0.0", {{ app_port }}), Handler).serve_forever()
```

Это Python **шаблон**: до подстановки `app_port` он не является готовым Python-файлом. Приложение слушает все IPv4-интерфейсы; доступ снаружи также зависит от маршрутизации и firewall.

### Файл `roles/app/templates/devops404-app.service.j2`

```ini
[Unit]
Description=DevOps404 app service
After=network.target

[Service]
User={{ service_user }}
ExecStart=/usr/bin/python3 {{ base_dir }}/app/app.py
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### Файл `roles/app/handlers/main.yml`

```yaml
---
- name: Restart app service
  ansible.builtin.systemd_service:
    name: devops404-app
    state: restarted
    daemon_reload: true
```

Изменение Python-кода требует перезапуска процесса. Два `notify` не создают два перезапуска в одной точке flush.

### 8.5. Файл `roles/db/tasks/main.yml`

```yaml
---
- name: Create database directory
  ansible.builtin.file:
    path: "{{ base_dir }}/db"
    state: directory
    owner: root
    group: root
    mode: "0755"

- name: Render database initialization script
  ansible.builtin.template:
    src: init_db.py.j2
    dest: "{{ base_dir }}/db/init_db.py"
    owner: root
    group: root
    mode: "0755"

- name: Initialize database state
  ansible.builtin.command:
    argv:                           # Аргументы отдельно, без разбора shell
      - /usr/bin/python3
      - "{{ base_dir }}/db/init_db.py"
  register: db_init
  changed_when: db_init.stdout | trim == 'changed'
  when: not ansible_check_mode       # Скрипт действительно меняет базу
```

### Файл `roles/db/templates/init_db.py.j2`

```python
import sqlite3

connection = sqlite3.connect({{ database_file | to_json }})
connection.execute(
    "CREATE TABLE IF NOT EXISTS services (name TEXT PRIMARY KEY, status TEXT NOT NULL)"
)
existing = connection.execute(
    "SELECT name FROM services WHERE name = ?", ("demo",)
).fetchone()

changed = existing is None
if changed:
    connection.execute(
        "INSERT INTO services(name, status) VALUES (?, ?)",
        ("demo", "ready"),
    )

connection.commit()
connection.close()
print("changed" if changed else "ok")
```

`to_json` формирует корректно заключённую в кавычки строку пути для этого шаблона. Параметры SQL передаются через `?`. Первый запуск создаёт таблицу и строку `demo`; повторный выводит `ok`. Сравнение `stdout == 'changed'` точнее поиска подстроки: слово `unchanged` тоже содержит `changed`.

Это инициализация, а не миграционный механизм: если у существующей строки `demo` изменить `status`, исходная логика не вернёт `ready`. Отчёт об изменениях отражает вставку отсутствующей строки. Для полного управления схемой и состоянием нужна отдельная логика.

### 8.6. Файл `site.yml`

```yaml
---
- name: Apply common configuration
  hosts: managed
  gather_facts: true
  become: true
  roles:
    - common

- name: Configure web server
  hosts: web
  gather_facts: false
  become: true
  roles:
    - web

- name: Configure application server
  hosts: app
  gather_facts: false
  become: true
  roles:
    - app

- name: Configure database server
  hosts: db
  gather_facts: false
  become: true
  roles:
    - db
```

Plays идут по порядку: сначала общая подготовка всех хостов, затем web, app и db. При `--limit target1` первый play также ограничивается target1, поэтому common не теряется.

### Первое развёртывание и проверка

```bash
ansible-inventory -i inventory.ini --graph
ansible -i inventory.ini managed -m ansible.builtin.ping
ansible-playbook -i inventory.ini site.yml --syntax-check
ansible-playbook -i inventory.ini site.yml --list-hosts
ansible-playbook -i inventory.ini site.yml

# HTTP-запрос выполняется на target1 к его собственному loopback
ansible -i inventory.ini target1 -m ansible.builtin.uri -a 'url=http://127.0.0.1:8080 return_content=true status_code=200'

# Аналогичная проверка приложения на target2
ansible -i inventory.ini target2 -m ansible.builtin.uri -a 'url=http://127.0.0.1:8081 return_content=true status_code=200'

# Повторная инициализация должна вывести ok
ansible -i inventory.ini target3 -m ansible.builtin.command -a 'python3 /opt/devops404/db/init_db.py'

# Проверить фактическую строку БД чтением, без повторной инициализации
ansible -i inventory.ini target3 -m ansible.builtin.command -a "python3 -c 'import sqlite3; c=sqlite3.connect(\"file:/opt/devops404/db/app.db?mode=ro\", uri=True); print(c.execute(\"SELECT name, status FROM services\").fetchall())'"

# Полный повтор для проверки идемпотентности
ansible-playbook -i inventory.ini site.yml
```

Ожидается `Web node target1`, `application is running`, строка `('demo', 'ready')` и `changed=0` у повторного **playbook**. Ad-hoc `command` сам по себе не применяет `changed_when` из роли и может показывать `changed` даже при выводе `ok`.

Проверка `127.0.0.1` подтверждает работу на самом target, но не доступность с control или другой VM. Для проверки сетевого пути запрос должен выполняться с нужной исходной машины по реальному адресу target.

### 8.7. Файл `ansible.cfg`

```ini
[defaults]
inventory = inventory.ini
roles_path = roles
host_key_checking = True
forks = 3
gathering = smart
fact_caching = jsonfile
fact_caching_connection = .facts
fact_caching_timeout = 3600

[ssh_connection]
pipelining = True
```

| Настройка | Функция |
| --- | --- |
| `inventory` | Inventory по умолчанию, можно не писать `-i` |
| `roles_path` | Где искать роли |
| `host_key_checking` | Проверять подлинность SSH-хостов |
| `forks` | Максимум параллельных рабочих процессов Ansible |
| `gathering = smart` | Использовать доступные facts вместо лишнего повторного сбора |
| `fact_caching = jsonfile` | Сохранять facts в файловый кэш |
| `fact_caching_connection` | Каталог кэша |
| `fact_caching_timeout` | Срок годности кэша, секунды |
| `pipelining` | Уменьшать число операций SSH при выполнении модулей |

Относительные пути рассчитаны на запуск из каталога этого проекта. Проверить выбранный конфиг можно через `ansible --version` и `ansible-config dump --only-changed`. Ansible использует первый найденный конфиг: `ANSIBLE_CONFIG`, затем `ansible.cfg` текущего каталога, `~/.ansible.cfg`, `/etc/ansible/ansible.cfg`; они не объединяются в один файл. Конфиг текущего каталога с правами записи для всех может игнорироваться.

```bash
ansible --version
ansible-config dump --only-changed

# Теперь inventory берётся из ansible.cfg
ansible-playbook site.yml
ansible-playbook site.yml

# При необходимости сбросить кэш facts перед запуском
ansible-playbook site.yml --flush-cache
```

После обычного запуска в `.facts` ожидаются записи для трёх хостов. Кэш может устареть; его наличие не означает, что состояние ОС не изменилось. `pipelining` зависит от SSH/become-настроек и не является гарантированным ускорением в любой среде.

### 8.8. Файл `rolling.yml`

```yaml
---
- name: Update managed servers one by one
  hosts: managed
  gather_facts: false
  become: true
  serial: 1
  tasks:
    - name: Record rollout batch
      ansible.builtin.copy:
        dest: "{{ base_dir }}/rollout.conf"
        content: |
          batch={{ ansible_play_batch | join(',') }}
        mode: "0644"
```

```bash
ansible-playbook rolling.yml
ansible managed -m ansible.builtin.command -a 'cat /opt/devops404/rollout.conf'
```

Запускается после `site.yml`, создавшего `base_dir`. Хотя `forks=3`, `serial: 1` допускает в текущую партию только один сервер. На каждой VM в файле будет её собственное имя.

### 8.9. Файл `summary.yml`

```yaml
---
- name: Create deployment summary
  hosts: managed
  gather_facts: false
  tasks:
    - name: Write summary on control node
      ansible.builtin.copy:
        dest: "{{ playbook_dir }}/deployment-summary.txt"
        content: |
          managed={{ ansible_play_hosts | sort | join(',') }}
        mode: "0644"
      run_once: true
      delegate_to: localhost
      become: false
```

```bash
ansible-playbook summary.yml
ansible localhost -i localhost, -c local -m ansible.builtin.command -a 'cat deployment-summary.txt'
```

Ожидается `managed=target1,target2,target3`. Это список выбранных активных хостов **этого** play, а не доказательство, что все прошлые развёртывания прошли успешно.

### 8.10. Файл `.gitignore`

```gitignore
.vault-pass
.vault-pass*
.facts/
*.retry
*.log
*.pem
*.key
id_rsa*
id_ed25519*
tagged.syntax.txt
orchestration-summary.txt
deployment-summary.txt
```

В Git сохраняются playbooks, роли, шаблоны, конфиг и этот справочник. `secret-vars.yml` можно коммитить только после шифрования с собственным сильным паролем; пароль, приватные ключи, кэш facts и реальные секреты не публикуются. Inventory для публичного репозитория должен содержать примеры адресов. `.gitignore` не удаляет уже отслеживаемый секрет из истории Git.

<a id="commands"></a>
## Команды Ansible с назначением

Ниже отдельная шпаргалка. Имена `site.yml`, `inventory.ini`, групп и путей подставляй из соответствующей лабораторной. Команды используют полные имена модулей; короткие `-m ping`, `-m copy` также работают для встроенных модулей.

### Inventory и подключение

| Команда | Назначение |
| --- | --- |
| `ansible --version` | Версия Ansible, Python и выбранный конфиг |
| `ansible-inventory -i inventory.ini --graph` | Дерево групп и хостов |
| `ansible-inventory -i inventory.ini --list` | Inventory с переменными в JSON |
| `ansible-inventory -i inventory.ini --host target1` | Переменные конкретного хоста |
| `ansible -i inventory.ini targets --list-hosts` | Хосты, соответствующие шаблону |
| `ansible -i inventory.ini targets -m ansible.builtin.ping` | Проверка Ansible-подключения и Python |
| `ansible -i inventory.ini target1 -m ansible.builtin.ping -vvv` | Подробности подключения для диагностики |
| `ansible-config dump --only-changed` | Отличающиеся от стандартных настройки |
| `ansible-config view` | Содержимое выбранного ansible.cfg |

### Playbooks: просмотр, применение, выборка

| Команда | Назначение |
| --- | --- |
| `ansible-playbook -i inventory.ini site.yml --syntax-check` | Проверка синтаксиса без применения |
| `ansible-playbook -i inventory.ini site.yml --list-hosts` | На каких хостах выполнялись бы plays |
| `ansible-playbook -i inventory.ini site.yml --list-tasks` | Список известных на этапе разбора задач |
| `ansible-playbook -i inventory.ini tagged.yml --list-tags` | Доступные теги |
| `ansible-playbook -i inventory.ini site.yml --check --diff` | Предварительная оценка и различия файлов |
| `ansible-playbook -i inventory.ini site.yml` | Реальное применение |
| `ansible-playbook -i inventory.ini site.yml --limit target1` | Ограничить выполнение одним хостом |
| `ansible-playbook -i inventory.ini site.yml --limit web` | Ограничить выполнение группой |
| `ansible-playbook -i inventory.ini tagged.yml --tags config,marker` | Выбрать задачи с перечисленными тегами |
| `ansible-playbook -i inventory.ini tagged.yml --skip-tags marker` | Пропустить задачи с тегом marker |
| `ansible-playbook -i inventory.ini site.yml -b -K` | Включить become и запросить пароль sudo |
| `ansible-playbook -i inventory.ini site.yml -vv` | Более подробный вывод выполнения |
| `ansible-playbook -i inventory.ini site.yml --step` | Подтверждать задачи по мере выполнения |
| `ansible-playbook -i inventory.ini site.yml --flush-cache` | Сбросить fact cache для inventory перед запуском |
| `ansible-playbook -i inventory.ini role-site.yml -e '{"demo_app_port": 9092}'` | Переопределить значение через extra vars, сохранив числовой тип |

`--list-tasks` и `--list-tags` не раскрывают полностью динамические `include_tasks` до выполнения. `-k` — пароль подключения SSH; `-K` — пароль become; `--ask-vault-pass` — пароль Vault. Это три разных назначения.

### Файлы, facts и диагностика target

```bash
# Прочитать удалённый файл
ansible -i inventory.ini target1 -m ansible.builtin.command -a 'cat /etc/os-release'

# Проверить файл: exists, mode, uid/gid и другие свойства
ansible -i inventory.ini target1 -m ansible.builtin.stat -a 'path=/opt/ansible-demo/app.env'

# Создать каталог
ansible -i inventory.ini target1 -m ansible.builtin.file -a 'path=/opt/ansible-demo state=directory mode=0755'

# Записать строку в файл; родительский каталог должен существовать
ansible -i inventory.ini target1 -m ansible.builtin.copy -a 'dest=/tmp/note.txt content=hello mode=0644'

# Обеспечить отсутствие файла или каталога: внимательно проверь path
ansible -i inventory.ini target1 -m ansible.builtin.file -a 'path=/tmp/note.txt state=absent'

# Получить все facts либо только сведения о дистрибутиве
ansible -i inventory.ini target1 -m ansible.builtin.setup
ansible -i inventory.ini target1 -m ansible.builtin.setup -a 'filter=ansible_distribution*'

# Получить сведения о службах
ansible -i inventory.ini target1 -m ansible.builtin.service_facts

# Проверить HTTP с самого target
ansible -i inventory.ini target1 -m ansible.builtin.uri -a 'url=http://127.0.0.1:8080 return_content=true'

# Посмотреть журнал учебной службы через Ansible
ansible -i inventory.ini target1 -b -m ansible.builtin.command -a 'journalctl -u devops404-web -n 50 --no-pager'
```

### Vault

| Команда | Назначение |
| --- | --- |
| `ansible-vault create secret-vars.yml` | Создать зашифрованный файл через редактор |
| `ansible-vault encrypt secret-vars.yml` | Зашифровать существующий открытый файл |
| `ansible-vault view secret-vars.yml` | Показать расшифрованное содержимое |
| `ansible-vault edit secret-vars.yml` | Изменить зашифрованный файл через редактор |
| `ansible-vault rekey secret-vars.yml` | Сменить пароль шифрования |
| `ansible-vault decrypt secret-vars.yml` | Расшифровать файл на месте; на диске останется открытый текст |
| `ansible-playbook -i inventory.ini tagged.yml --ask-vault-pass` | Запустить playbook с интерактивным вводом Vault-пароля |
| `ansible-playbook -i inventory.ini tagged.yml --vault-password-file .vault-pass` | Прочитать Vault-пароль из защищённого файла |

### Встроенная справка

```bash
# Параметры, примеры и поддержка check mode установленной версии модуля
ansible-doc ansible.builtin.copy
ansible-doc ansible.builtin.template
ansible-doc ansible.builtin.systemd_service

# Список доступных модулей
ansible-doc -l
```

<a id="lookup"></a>
## Таблица модулей и ключевых слов

### Как выбрать модуль

| Задача | Модуль | Что помнить |
| --- | --- | --- |
| Проверить подключение | `ansible.builtin.ping` | Не проверяет ICMP |
| Каталоги, права, удаление | `ansible.builtin.file` | Не записывает произвольное содержимое |
| Текст из play или файл с control | `ansible.builtin.copy` | Нужен `content` или `src`, не оба сразу |
| Конфиг из `.j2` | `ansible.builtin.template` | Используются `src` и `dest`, параметра `content` нет |
| Проверить свойства файла | `ansible.builtin.stat` | Не изменяет файл |
| Выполнить программу | `ansible.builtin.command` | Нет shell-конвейеров и перенаправления; удобно `argv` |
| Команда с `|`, `>`, shell-переменными | `ansible.builtin.shell` | Используй, когда действительно нужна оболочка |
| Получить facts | `ansible.builtin.setup` | Обычно вызывается через gather_facts |
| Вывести переменную | `ansible.builtin.debug` | Не печатай секреты |
| Подключить файл tasks | `ansible.builtin.include_tasks` | Файл содержит список задач, а не play |
| Управлять группой/пользователем | `ansible.builtin.group`, `ansible.builtin.user` | Для системных изменений нужны права |
| Управлять systemd | `ansible.builtin.systemd_service` | `started` не равно `restarted`; старое имя `systemd` — alias |
| Дождаться TCP-порта | `ansible.builtin.wait_for` | Успех TCP не гарантирует работоспособность HTTP |
| Проверить HTTP | `ansible.builtin.uri` | Можно проверить status_code и содержимое |
| Выполнить накопленные handlers | `ansible.builtin.meta: flush_handlers` | Выполняет уведомлённые handlers текущего play |

### Свойства play и задачи

| Ключ | Уровень | Назначение |
| --- | --- | --- |
| `hosts` | Play | Выбрать хосты |
| `gather_facts` | Play | Автоматический сбор сведений |
| `vars`, `vars_files` | В примерах — play | Переменные внутри play или из файлов |
| `roles` | Play | Подключить роли |
| `serial` | Play | Разбить хосты на партии |
| `become` | Play / block / task | Повысить права выполнения |
| `loop` | Task | Повторить действие для элементов списка |
| `when` | Task / block | Условие выполнения |
| `register` | Task | Сохранить результат |
| `changed_when` | Task | Условие статуса changed |
| `failed_when` | Task | Условие ошибки |
| `until`, `retries`, `delay` | Task | Условие и параметры повторных попыток |
| `notify` | Task | Поставить handler в очередь при changed |
| `tags` | Play / block / task / role | Выбор частей запуска |
| `no_log`, `diff` | Task | Ограничить вывод, в том числе секретов |
| `run_once` | Task | Один запуск для текущей партии |
| `delegate_to` | Task | Выполнить действие на другом узле |
| `throttle` | Task / block / play | Дополнительный предел параллельности |

### Переменные, которые часто путают

| Переменная | Смысл |
| --- | --- |
| `inventory_hostname` | Имя хоста в inventory |
| `ansible_host` | Адрес подключения; может отсутствовать, тогда используется имя inventory |
| `ansible_facts.hostname` | Системное имя, полученное из facts |
| `ansible_play_batch` | Активные хосты текущей партии |
| `ansible_play_hosts` | Активные хосты текущего play, с учётом выбывших |
| `ansible_play_hosts_all` | Все хосты, выбранные для play |
| `playbook_dir` | Каталог текущего playbook на control node |
| `ansible_check_mode` | Признак запуска с --check |

<a id="troubleshooting"></a>
## Частые ошибки и диагностика

| Симптом | Что проверить |
| --- | --- |
| `UNREACHABLE` | IP/DNS, маршрут, SSH-порт, firewall, ключ, пользователя, known_hosts |
| `Missing sudo password` | Настройку sudo и необходимость `-K` |
| Ошибка Python interpreter | Наличие Python и `ansible_python_interpreter` |
| `Unsupported parameters ... tags` | `tags` ошибочно вложен внутрь параметров модуля |
| `conflicting action statements` | Два модуля в одной task или неверные отступы |
| `Undefined variable` | Имя переменной, группу хоста, путь group_vars и сбор facts |
| `Destination directory ... does not exist` | Задачу создания родительского каталога; режим --check |
| Handler не найден | Полное совпадение имени `notify` и handler |
| Handler не запускается | Было ли changed; дошёл ли play до точки выполнения handlers |
| Не выбираются задачи include по тегу | У include и вложенных tasks собственная логика тегов |
| `no hosts matched` | Название группы, путь inventory и пересечение с --limit |
| Повторный запуск всё время changed | command/shell без changed_when, state=restarted, динамическое содержимое |
| `wait_for` ждёт новый порт до тайм-аута | Выполнен ли перезапуск после изменения конфигурации |
| Ошибка расшифровки Vault | Правильный пароль, файл пароля и vault-id, если используется |
| Конфиг не применяется | Какой ansible.cfg выбран; нет ли переопределений CLI/env |

### `content` — нужен ли он?

- Для `copy` с текстом прямо в YAML нужен `content`.
- Для `copy` с готовым локальным файлом нужен `src`, а `content` не нужен.
- Для `template` нужен путь к `.j2` через `src`; содержимое лежит в шаблоне.
- Для `file` параметры `content` и `src` для записи текста не используются: модуль управляет состоянием и свойствами объекта.

### `command` или `shell`?

`command` запускает программу без shell-операторов. Для пайпа, перенаправления, `&&` и shell-скрипта нужен `shell`. Если существует профильный модуль (`file`, `copy`, `user`, `systemd_service`), он обычно лучше выражает желаемое состояние и умеет определить необходимость изменения.

### Теги у include

Тег на `include_tasks` сам по себе не присваивается автоматически всем вложенным задачам. Дополнительный пример:

```yaml
- name: Include config tasks with inherited tag
  ansible.builtin.include_tasks:
    file: config.yml
    apply:
      tags: [config]                # Назначить тег задачам внутри файла
  tags: [config]                    # Позволить выбрать само подключение
```

`config.yml` должен существовать и содержать список tasks. Это фрагмент для вставки внутрь `tasks`, а не самостоятельный playbook. Для статического `import_tasks` поведение наследования и предварительного разбора отличается.

### Что означает PLAY RECAP

| Поле | Значение |
| --- | --- |
| `ok` | Успешные задачи, включая те, которые сообщили changed |
| `changed` | Задачи, сообщившие об изменении |
| `unreachable` | Хосты/действия с проблемой подключения |
| `failed` | Необработанные ошибки |
| `skipped` | Пропущенные задачи |
| `rescued` | Ошибки, обработанные через rescue |
| `ignored` | Ошибки, проигнорированные настройками |

`changed=0` полезен только вместе с проверкой результата: ошибочное `changed_when: false` может скрыть реальные изменения. `--check` не является транзакцией или откатом, а задачи с `check_mode: false` выполняются реально даже при глобальном `--check`.

<a id="extra"></a>
## Дополнения: host_vars, Galaxy, Molecule, AWX

Эти темы упомянуты в заключении курса, но полноценные лабораторные для них во вложении отсутствуют. Ниже краткие дополнения, а не восстановленные задания.

### host_vars и приоритеты переменных

Для индивидуальной настройки хоста можно создать **`host_vars/target1.yml`**:

```yaml
---
service_port: 8082
```

В контексте лабораторной 7 это переопределит групповой `service_port` для target1. Упрощённая последовательность для показанных способов задания, от меньшего приоритета к большему:

| Приоритет | Источник |
| --- | --- |
| Ниже | `roles/.../defaults/main.yml` |
| Выше | Inventory group vars |
| Выше | Inventory host vars |
| Выше | Play vars |
| Наивысший среди переменных | `-e` / `--extra-vars` |

Это не полный перечень правил Ansible: task vars, role vars, include params и другие источники имеют свои позиции. Для настроек роли, которые пользователь должен менять, обычно подходят `defaults`, а не `vars/main.yml`.

### forks, serial и throttle

| Параметр | Что ограничивает | Пример |
| --- | --- | --- |
| `forks` | Общую параллельность рабочих процессов | `forks=10` |
| `serial` | Число хостов, проходящих play одной партией | `serial: 2` |
| `throttle` | Параллельное выполнение выбранной задачи/блока | `throttle: 1` |

Если `forks=10`, `serial=2`, текущая партия — два хоста. `throttle: 1` на задаче дополнительно ограничит её одним хостом за раз; увеличить параллельность сверх forks/serial он не может.

### Galaxy и requirements.yml

Роль — отдельный набор автоматизации. Collection может включать роли, модули, плагины и документацию. Для встроенных `ansible.builtin.*` из лабораторных сторонние collections не нужны.

Шаблон **`requirements.yml`** — демонстрационный, перед установкой замени имена и версии:

```yaml
---
collections:
  - name: namespace.collection_name
    version: "X.Y.Z"                 # Реальная проверенная версия collection
roles:
  - name: namespace.role_name
    version: "X.Y.Z"                 # Реальная версия роли
```

```bash
# Создать каркас новой роли
ansible-galaxy role init demo_app

# Установить указанные collections или роли из заполненного requirements.yml
ansible-galaxy collection install -r requirements.yml
ansible-galaxy role install -r requirements.yml

# Посмотреть установленные зависимости
ansible-galaxy collection list
ansible-galaxy role list
```

Не запускай `role init` поверх уже готовой роли из лабораторной. Каркас содержит больше стандартных каталогов, чем минимальная роль этого курса.

### Dynamic inventory

Вместо ручного INI список машин может возвращать inventory plugin для облака, виртуализации или CMDB. Источник и credentials настраиваются отдельно; результат всё равно проверяется `ansible-inventory --graph` и `--list`. Конкретного облачного плагина и его настроек во вложении нет.

### Molecule

Molecule помогает применять роль в подготовленном временном окружении и проверять её поведение. В типичном сценарии есть `molecule/default/molecule.yml` (настройка сценария), `converge.yml` (применение роли), `verify.yml` (проверки). Состав шагов зависит от версии и конфигурации сценария.

| Команда | Назначение в настроенном сценарии |
| --- | --- |
| `molecule converge` | Применить роль к тестовому окружению |
| `molecule verify` | Проверить результат |
| `molecule test` | Выполнить настроенную полную последовательность тестирования |

Это команды отдельного инструмента экосистемы Ansible. Они не заработают только от наличия role: нужны установленный Molecule, подходящий сценарий и среда выполнения. Для ролей с systemd обычный контейнер без init не равнозначен VM. В исходном курсе Molecule не разворачивался.

### AWX

AWX организует централизованные запуски Ansible с inventory, credentials, правами, расписанием и историей. Это не замена YAML-кода ролей.

| Сущность | Назначение |
| --- | --- |
| Project | Источник Ansible-кода, обычно Git-репозиторий |
| Inventory | Хосты, группы, переменные |
| Credential | Учётные данные подключения, Vault и других интеграций |
| Job Template | Настроенный запуск playbook |
| Workflow Template | Связанные задания и порядок их запуска |

Molecule проверяет автоматизацию в тестовой среде; AWX управляет её запуском. Установка AWX в исходных лабораторных не предусмотрена.

<a id="sources"></a>
## Проверка документа и источники

Основной источник заданий — приложенный файл «для копирования.txt». Полные решения написаны по его требованиям. Изменения относительно исходника: восстановлены обрезанные команды и раздел 4; разделены проекты; добавлены явные права и группа сервисного пользователя; уточнены `changed_when`, retries и ограничения check mode; handlers служб выполняются до проверки изменённого порта; дополнена защита вывода секретов.

Официальная документация для уточнения поведения:

- [Ключевые слова playbook](https://docs.ansible.com/projects/ansible/latest/reference_appendices/playbooks_keywords.html).
- [Handlers и flush_handlers](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_handlers.html).
- [Check mode и diff mode](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_checkmode.html).
- [Циклы и повторные попытки](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_loops.html).
- [Повторное использование tasks и ролей](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_reuse.html).

Точные параметры модулей для своей установленной версии проверяй через `ansible-doc`. Ссылки `latest` со временем обновляются.

### Что проверено при подготовке

- Все 34 YAML-блока успешно разобраны YAML-парсером.
- 12 итоговых playbooks, извлечённых из документа вместе с ролями и переменными, прошли `ansible-playbook --syntax-check` на `ansible-core 2.21.5`. Для синтаксической проверки Vault-пример использовался с открытой учебной переменной.
- Все 7 Jinja-шаблонов отрисованы с тестовыми значениями; получившиеся Python-скрипты прошли проверку синтаксиса.
- Скрипт SQLite действительно выполнен дважды на временной локальной базе: сначала `changed`, затем `ok`; проверена строка `demo` со статусом `ready`.
- Все 15 ссылок оглавления указывают на существующие разделы.

SSH-подключение к учебным VM, реальные systemd-службы, права на удалённых хостах и итоговый `changed=0` всего проекта здесь не проверялись. Для этого используй команды самопроверки на своём стенде. Локальная синтаксическая проверка не заменяет такое развёртывание.
