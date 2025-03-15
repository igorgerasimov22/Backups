# Ansible Playbook для настройки резервного сервера с использованием BorgBackup

Этот Ansible playbook предназначен для автоматизации настройки резервного сервера с использованием BorgBackup. Он включает в себя установку необходимых пакетов, создание пользователей и групп, конфигурацию SSH, и настройку задач резервного копирования.

## **Содержание**
- Требования**
- Структура Playbook**
- Задачи**
- Использование**
- Лицензия**

## **Требования**

Перед использованием этого playbook убедитесь, что на ваших серверах установлены следующие компоненты:
- Ansible
- BorgBackup
- Доступ по SSH к серверам

## **Структура Playbook**

Playbook состоит из нескольких задач, сгруппированных по этапам, для настройки резервного сервера и клиента:
- **Обновление кэша пакетов и установка BorgBackup**
- **Создание группы и пользователя для Borg**
- **Настройка SSH для доступа без пароля**
- **Инициализация репозитория Borg**
- **Настройка задачи резервного копирования с помощью systemd**

## **Задачи**

### **1. Обновление кэша пакетов и установка BorgBackup**

```yaml
- name: Update package cache
  ansible.builtin.apt:
    update__cache: true
    cache__valid_time: 3600
- name: install borgbackup
  ansible.builtin.apt:
    name: borgbackup
    state: present
```

### 2. Создание группы и пользователя для Borg
```yaml
- name: Add borg group
  ansible.builtin.group:
    name: borg
    state: present
- name: Add borg user
  ansible.builtin.user:
    name: borg
    group: borg
    state: present
    home: /home/borg
    shell: /bin/bash
- name: Add user folder
  ansible.builtin.file:
    path: /home/borg
    owner: borg
    group: borg
```
### 3. Настройка SSH для доступа без пароля
```yaml
- name: Create authorized__keys files
  ansible.builtin.file:
    path: /home/borg/.ssh/authorized__keys
    state: file
    owner: borg
    group: borg
    mode: '0600'
- name: Add SSH folder
  ansible.builtin.file:
    path: /home/borg/.ssh
    state: directory
    owner: borg
    group: borg
    mode: '0700'
- name: Generate SSH key
  community.crypto.openssh__keypair:
    path: /home/borg/.ssh/id__ed25519
    type: ed25519
    owner: borg
    group: borg
- name: Set public SSH key variable
  ansible.builtin.set__fact:
    public__ssh__key: "{{ key__result.public__key }}"
- name: Получение открытого ключа пользователя Borg
  slurp:
    src: /home/borg/.ssh/id__ed25519.pub
  register: borg__ssh__key
- name: Декодирование открытого ключа
  set__fact:
    ssh__key__borg: "{{ borg__ssh__key.content | b64decode }}"
- name: Добавление открытого ключа на сервер 2
  authorized__key:
    user: borg
    state: present
    key: "{{ ssh__key__borg }}"
  delegate_to: backup-server
```
### 4. Инициализация репозитория Borg
```yaml
- name: Initialize borg repo
  ansible.builtin.shell: |
    su - borg -c "borg init --encryption=none borg@192.168.88.230:/var/backup/"
  register: borg__init__result
  ignore__errors: yes
- name: Check if repository already exists
  ansible.builtin.debug:
    msg: "Warning: A repository already exists at borg@192.168.88.230:/var/backup/."
  when: borg__init_result.rc == 2
```
### 5. Настройка задачи резервного копирования с помощью systemd
```yaml
- name: Run backup
  ansible.builtin.shell: |
    su - borg -c "borg create --stats --list borg@192.168.88.230:/var/backup/::"etc-{now:%Y-%m-%d__%H:%M:%S}" /etc"
  ignore__errors: true
- name: Copy borg-backup.service
  ansible.builtin.copy:
    src: ~/ansible/Backups/borg-backup.service
    dest: /etc/systemd/system/borg-backup.service
    mode: '0755'
  notify: Enable timer
- name: Copy borg-backup.timer
  ansible.builtin.copy:
    src: ~/ansible/Backups/borg-backup.timer
    dest: /etc/systemd/system/borg-backup.timer
    mode: '0755'
  notify: Enable timer
```

## Использование
Чтобы запустить этот playbook, используйте следующую команду, заменив ```your_inventory_file``` на путь к вашему инвентарному файлу:
```bash
ansible-playbook your_playbook.yml -i your_inventory_file --ask-become-pass
```