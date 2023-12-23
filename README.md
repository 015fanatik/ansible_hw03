# Домашнее задание к занятию 3 «Использование Ansible»

## Подготовка к выполнению

1. Подготовьте в Yandex Cloud три хоста: для `clickhouse`, для `vector` и для `lighthouse`.
2. Репозиторий LightHouse находится [по ссылке](https://github.com/VKCOM/lighthouse).

## Основная часть

1. Допишите playbook: нужно сделать ещё один play, который устанавливает и настраивает LightHouse.
2. При создании tasks рекомендую использовать модули: `get_url`, `template`, `yum`, `apt`.
3. Tasks должны: скачать статику LightHouse, установить Nginx или любой другой веб-сервер, настроить его конфиг для открытия LightHouse, запустить веб-сервер.
4. Подготовьте свой inventory-файл `prod.yml`.
5. Запустите `ansible-lint site.yml` и исправьте ошибки, если они есть.
6. Попробуйте запустить playbook на этом окружении с флагом `--check`.
7. Запустите playbook на `prod.yml` окружении с флагом `--diff`. Убедитесь, что изменения на системе произведены.
8. Повторно запустите playbook с флагом `--diff` и убедитесь, что playbook идемпотентен.
9. Подготовьте README.md-файл по своему playbook. В нём должно быть описано: что делает playbook, какие у него есть параметры и теги.
10. Готовый playbook выложите в свой репозиторий, поставьте тег `08-ansible-03-yandex` на фиксирующий коммит, в ответ предоставьте ссылку на него.

---

### Как оформить решение задания

Выполненное домашнее задание пришлите в виде ссылки на .md-файл в вашем репозитории.

## **Основные компоненты**  

  # Структура каталогов
В процессе установки будем использовать 
 ```
  -/group_vars  
      /clickhouse
      /vector   
      -vector.yaml.j2  
  -/invetnory  
      -prod.yml  
  -site.yaml   
  ```

  # Назначение файлов в структуре
- *В директории group_vars мы описываем переменные и шаблоны*  
- *В файле inventory указываем группы хостов*
- *site.yml основной плейбук Ansible*
___

## **Конфигурация основного плейбука**
  
```yaml

---
---
- name: Install Clickhouse
  hosts: clickhouse
  handlers:
    - name: Start clickhouse service
      become: true
      ansible.builtin.service:
        name: clickhouse-server
        state: restarted
  tasks:
    - block:
        - name: Get clickhouse distrib
          ansible.builtin.get_url:
            url: "https://packages.clickhouse.com/rpm/stable/{{ item }}-{{ clickhouse_version }}.noarch.rpm"
            dest: "./{{ item }}-{{ clickhouse_version }}.rpm"
          with_items: "{{ clickhouse_packages }}"
      rescue:
        - name: Get clickhouse distrib
          ansible.builtin.get_url:
            url: "https://packages.clickhouse.com/rpm/stable/clickhouse-common-static-{{ clickhouse_version }}.x86_64.rpm"
            dest: "./clickhouse-common-static-{{ clickhouse_version }}.rpm"

    - name: Install clickhouse packages
      become: true
      ansible.builtin.yum:
        name:
          - clickhouse-common-static-{{ clickhouse_version }}.rpm
          - clickhouse-client-{{ clickhouse_version }}.rpm
          - clickhouse-server-{{ clickhouse_version }}.rpm
      notify: Start clickhouse service

    - name: Flush handlers
      meta: flush_handlers

- name: Install nginx
  hosts: lighthouse
  become: true
  become_user: root
  handlers:
    - name: nginx systemd
      systemd:
        name: nginx
        enabled: yes
        state: started
  tasks:
    - name: Install EPEL Repo
      yum:
        name=epel-release
        state=present

    - name: Install Nginx Web Server
      yum:
        name=nginx

    - name: Install git
      yum:
        name=git

    - name: write using jinja2
      ansible.builtin.template:
         src: ./group_vars/nginx.yaml.j2
         mode: 0644
         dest: /etc/nginx/nginx.conf
         owner: nginx
         group: nginx

    - name: Rm folder
      ansible.builtin.command: "rm -rf /tmp/nginx/"

    - name: Git clone
      ansible.builtin.command: "git clone https://github.com/VKCOM/lighthouse.git /tmp/nginx/"

    - name: Rm site folder
      ansible.builtin.command: "rm -rf /usr/share/nginx/index"

    - name: Cp
      ansible.builtin.command: "cp -R /tmp/nginx/ /usr/share/nginx/index"
      notify: nginx systemd


    - name: Create database
      ansible.builtin.command: "clickhouse-client -q 'create database logs;'"
      register: create_db
      failed_when: create_db.rc != 0 and create_db.rc !=82
      changed_when: create_db.rc == 0

- name: Install Vector
  hosts: vector
  become: true
  become_user: root
  handlers:
    - name: Start vector service
      become: true
      ansible.builtin.service:
        name: vector
        state: restarted

  tasks:
    - block:
        - name: Get vector distrib
          ansible.builtin.get_url:
            url: "https://packages.timber.io/vector/{{ vector_version }}/vector-{{ vector_version }}-1.{{ ansible_architecture }}.rpm"
            dest: "/home/centos/vector-{{ vector_version }}.rpm"

    - name: Install Vector packages
      become: true
      ansible.builtin.yum:
        name:
          - vector-{{ vector_version }}.rpm

    - name: write using jinja2
      ansible.builtin.template:
         src: ./group_vars/vector.yaml.j2
         mode: 0644
         dest: /etc/vector/vector.yaml
         owner: vector
         group: vector
      notify: Start vector service



```



---
