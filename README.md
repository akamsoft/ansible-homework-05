# Домашнее задание к занятию 5 «Тестирование roles»

**Репозиторий с ролью:** https://github.com/akamsoft/vector-role
**Тег:** [v1.0.0](https://github.com/akamsoft/vector-role/releases/tag/v1.0.0)

## 📖 О чём это задание

Настроено тестирование роли `vector-role` с помощью **Molecule** и драйвера **Docker**. Сценарий проверяет установку и настройку Vector на двух дистрибутивах: **Oracle Linux 9** и **Ubuntu latest**.

## 🛠️ Что сделано

### 1. Установка Molecule

Molecule установлена через `pipx` (изолированное venv, не затрагивает системный Python):

```bash
pipx install molecule
pipx inject molecule "molecule-plugins[docker]"
ansible-galaxy collection install community.docker ansible.posix
```

Версия: `molecule 26.11.0`, `ansible-core 2.19.11`.

### 2. Сценарий тестирования `molecule/default/`

Структура:

```
molecule/default/
├── molecule.yml      # 2 платформы (oraclelinux:9, ubuntu:latest) + systemd
├── create.yml        # создание контейнеров + генерация inventory
├── destroy.yml       # удаление контейнеров
├── converge.yml      # применение роли vector-role
└── verify.yml        # 8 assert-проверок
```

**`molecule.yml`** — ключевые параметры:

```yaml
driver:
  name: default        # delegated driver (стабильнее docker-плагина)
platforms:
  - name: instance-oraclelinux-9
    image: "mpaivabarbosa/molecule-systemd-oraclelinux:9"
    privileged: true
    cgroupns_mode: host
    tmpfs: [/run]
    ansible_python_interpreter: /usr/bin/python3.9
  - name: instance-ubuntu-latest
    image: "buluma/docker-molecule-images:ubuntu_latest"
    privileged: true
    cgroupns_mode: host
    tmpfs: [/run]
```

`create.yml` создаёт контейнеры и **формирует inventory** с `ansible_connection: community.docker.docker` — так Ansible подключается через Docker API, а не SSH.

### 3. Verify-проверки (`verify.yml`)

8 assert-задач на каждом инстансе:

1. бинарник `vector` существует;
2. бинарник исполняем;
3. конфиг `/etc/vector/vector.yaml` существует;
4. сервис `vector` в состоянии `active`;
5. `vector validate` возвращает `rc=0` (конфиг валиден).

### 4. Кроссплатформенность

Задача `Install required packages` в `tasks/main.yml` использует `ansible.builtin.package` — универсальный модуль, который сам выбирает `dnf`/`apt` в зависимости от ОС.

## ✅ Результаты

### `molecule converge` — 8 изменений на обоих

```
instance-oraclelinux-9 : ok=11  changed=8  failed=0
instance-ubuntu-latest : ok=11  changed=8  failed=0
```

### `molecule verify` — 8 assert прошли

```
instance-oraclelinux-9 : ok=8  changed=0  failed=0
instance-ubuntu-latest : ok=8  changed=0  failed=0
```

### `molecule test` — полный цикл успешен

```
SCENARIO RECAP
default : actions=12  successful=7  missing=6  failed=0
```

Ключевые этапы:
- `converge` — ✅ `changed=8`;
- `idempotence` — ✅ **`changed=0`** (роль идемпотентна);
- `verify` — ✅ все 8 `ok`;
- `destroy` — ✅ контейнеры удалены.

## 🖼️ Скриншоты

- **converge:** `images/01-molecule-converge.png`
- **verify:** `images/02-molecule-verify.png`
- **molecule test (SCENARIO RECAP):** `images/03-molecule-test.png`

## 📌 Итог

- ✅ Сценарий Molecule создан (`molecule init scenario`)
- ✅ Две платформы: Oracle Linux 9 + Ubuntu latest
- ✅ Роль работает на обоих дистрибутивах (кроссплатформенность через `ansible.builtin.package`)
- ✅ `verify.yml` с 8 assert-проверками (бинарник, конфиг, сервис, `vector validate`)
- ✅ `molecule test` проходит полностью, включая **idempotence**
- ✅ Тег `v1.0.0` на коммите с рабочим сценарием

## Tox + облегчённый Podman-сценарий

### Что сделано

1. Установлены `tox`, `molecule`, `molecule-plugins[podman]`, `ansible-core` на хосте `dev-vm`.
2. Создан облегчённый сценарий `molecule/podman/` — только Ubuntu, минимум проверок:
   - `molecule.yml` — драйвер `podman`, одна платформа `instance-ubuntu`;
   - `create.yml` — создаёт контейнер через `containers.podman.podman_container` и генерирует inventory с `ansible_connection: containers.podman.podman`;
   - `destroy.yml` — удаляет контейнер (`force_delete: true`);
   - `converge.yml` — применяет роль `vector-role`;
   - `verify.yml` — 2 assert-проверки (бинарник + сервис).
3. Создан `tox.ini`:

   ```ini
   [tox]
   envlist = molecule
   skipsdist = true

   [testenv]
   deps =
       molecule
       molecule-plugins[podman]
       ansible-core
   commands =
       molecule test -s podman

## 📤 Ссылки

- **Роль с тестом:** https://github.com/akamsoft/vector-role/tree/main/molecule
- **Тег:** https://github.com/akamsoft/vector-role/releases/tag/v1.0.0
- **Этот отчёт:** https://github.com/akamsoft/ansible-homework-05/blob/main/README.md