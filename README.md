# Лаб. П04 — Ansible basics: деплой веб-сервера с помощью Ansible

Практика по Ansible: вторая ВМ (managed node), отдельная NAT Network, пользователь `runner`
с SSH-ключом и passwordless sudo, control node с Ansible на первой ВМ, плейбук
установки nginx + копирования `index.html`, проверка через `curl`.

## Стенд

Две VM Ubuntu Server 24.04.3 LTS в **Oracle VirtualBox**, в отдельной **NAT Network**
`lab4net` (`10.0.2.0/24`, шлюз `10.0.2.1`):

| ВМ | Роль | IP в `lab4net` | SSH с хоста |
|---|---|---|---|
| `devops-lab01` | Control node (Ansible) | `10.0.2.15` | `127.0.0.1:2223` |
| `devops-lab04-vm2` | Managed node (nginx) | `10.0.2.16` | `127.0.0.1:2224` |

`devops-lab04-vm2` создана клонированием `devops-lab01` (`VBoxManage clonevm ... --register`,
MAC-адреса сгенерированы заново автоматически). HTTP с управляемого узла дополнительно
проброшен на хост: `127.0.0.1:8080` → `10.0.2.16:80`.

Работа выполнена пользователем `andrey` на control node; на managed node для Ansible
создан отдельный пользователь `runner` (passwordless sudo, вход по SSH-ключу).

## Содержимое

| Файл/папка | Назначение |
|---|---|
| `ansible/inventory.ini` | инвентарь Ansible (группа `webservers` → `10.0.2.16`) |
| `ansible/playbook.yml` | плейбук: установка nginx + копирование `index.html` |
| `ansible/index.html` | шаблон главной страницы, разворачиваемый модулем `copy` |
| `screenshots/` | подтверждение работы — реальный вывод терминала и `VBoxManage` |
| `README.md` | этот отчёт |

## Задание и как оно выполнено

> 0. Отключить скрипт `weather.sh` из прошлой работы (`crontab -e`).
> 1. Создать вторую ВМ (клон первой, новые MAC), настроить сеть (NAT Network,
>    статические IP, проброс портов).
> 2. Сгенерировать и скопировать SSH-ключ со второй ВМ (control node) на первую
>    (managed node) — пользователь `runner`.
> 3. На управляемой ВМ создать пользователя `runner` с passwordless sudo.
> 4. На control node установить Ansible, написать плейбук установки nginx, инвентарь,
>    шаблон `index.html`, прогнать плейбук.
> 5. Проверить результат через `curl`.

| # | Шаг | Как выполнено |
|---|---|---|
| 0 | Отключить `weather.sh` | `crontab -l \| sed '/weather.sh/ s/^/#/' \| crontab -` на `devops-lab01` — скриншот 1 |
| 1 | Вторая ВМ + сеть | `VBoxManage clonevm devops-lab01 --name devops-lab04-vm2 --register` (новый MAC `08002771CF2E`); создана `NAT Network lab4net` (`10.0.2.0/24`); обе ВМ переключены на неё (`--nic1 natnetwork --nat-network1 lab4net`); проброс портов `ssh-vm1`/`ssh-vm2`/`http-vm2` — скриншоты 1–2; статические IP через netplan (`10.0.2.15` / `10.0.2.16`), проверено `ping` между ВМ — скриншот 3 |
| 2 | SSH-ключ control → managed | на `devops-lab01`: `ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519`; публичный ключ добавлен в `/home/runner/.ssh/authorized_keys` на `devops-lab04-vm2`; проверен вход и `sudo` без пароля — скриншот 5 |
| 3 | Пользователь `runner` + sudo | на `devops-lab04-vm2`: `useradd -m runner`, `/etc/sudoers.d/runner` (`runner ALL=(ALL) NOPASSWD:ALL`), `visudo -c` — скриншот 4 |
| 4 | Ansible + плейбук | на `devops-lab01`: `apt install ansible` (core 2.16.3); `~/lab-ansible/{inventory.ini,playbook.yml,index.html}`; `ansible -m ping`, `ansible-playbook --check --diff`, затем реальный запуск (`changed=2`) — скриншоты 6–7 |
| 5 | Проверка | `curl http://10.0.2.16/` с control node и `curl http://127.0.0.1:8080/` с хоста — страница, задеплоенная Ansible, отдаётся в обоих случаях — скриншот 8 |

Все шаги выполнены реально на паре живых ВМ (не имитация): реальное клонирование
в VirtualBox, реальная отдельная NAT-сеть, реальный SSH-ключ и `runner`, реальная
установка Ansible и прогон плейбука против второй ВМ по сети `10.0.2.0/24`.

## Инвентарь и плейбук

`ansible/inventory.ini`:
```ini
[webservers]
10.0.2.16

[webservers:vars]
ansible_user=runner
```

`ansible/playbook.yml`:
```yaml
---
- name: Install and configure nginx web server
  hosts: webservers
  become: true
  gather_facts: true

  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Ensure nginx is running and enabled
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

    - name: Copy index.html to web root
      ansible.builtin.copy:
        src: index.html
        dest: /var/www/html/index.nginx-debian.html
        owner: www-data
        group: www-data
        mode: "0644"
```

## Развёртывание (команды, выполненные на хосте и на ВМ)

```bash
# --- хост: шаг 1, клонирование и сеть ---
# (ВМ devops-lab01 предварительно выключена: sudo shutdown -h now)
VBoxManage clonevm devops-lab01 --name devops-lab04-vm2 --register
VBoxManage natnetwork add --netname lab4net --network "10.0.2.0/24" --enable --dhcp on
VBoxManage modifyvm devops-lab01     --nic1 natnetwork --nat-network1 lab4net
VBoxManage modifyvm devops-lab04-vm2 --nic1 natnetwork --nat-network1 lab4net
VBoxManage natnetwork modify --netname lab4net --port-forward-4 "ssh-vm1:tcp:[127.0.0.1]:2223:[10.0.2.15]:22"
VBoxManage natnetwork modify --netname lab4net --port-forward-4 "ssh-vm2:tcp:[127.0.0.1]:2224:[10.0.2.16]:22"
VBoxManage natnetwork modify --netname lab4net --port-forward-4 "http-vm2:tcp:[127.0.0.1]:8080:[10.0.2.16]:80"
VBoxManage startvm devops-lab01     --type headless
VBoxManage startvm devops-lab04-vm2 --type headless

# --- на каждой ВМ: статический IP через netplan ---
# devops-lab01: addresses: [10.0.2.15/24]; devops-lab04-vm2: addresses: [10.0.2.16/24]
sudo netplan apply

# --- шаг 0: отключаем cron weather.sh (devops-lab01) ---
(crontab -l | sed '/weather.sh/ s/^/#/') | crontab -

# --- шаг 3: пользователь runner на devops-lab04-vm2 ---
sudo useradd -m -s /bin/bash runner
echo 'runner ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/runner
sudo chmod 440 /etc/sudoers.d/runner
sudo visudo -c

# --- шаг 2: SSH-ключ на devops-lab01, копируем на runner@devops-lab04-vm2 ---
ssh-keygen -t ed25519 -C "andrey@devops-lab01-ansible" -f ~/.ssh/id_ed25519 -N ""
ssh-copy-id -i ~/.ssh/id_ed25519.pub runner@10.0.2.16
ssh runner@10.0.2.16 'whoami && hostname && sudo whoami'   # без пароля

# --- шаг 4: Ansible на devops-lab01 (control node) ---
sudo apt-get install -y ansible
mkdir ~/lab-ansible && cd ~/lab-ansible
# inventory.ini, playbook.yml, index.html — см. ansible/ в этом репозитории

ansible webservers -i inventory.ini -m ping
ansible-playbook playbook.yml -i inventory.ini --check --diff   # сухой прогон
ansible-playbook playbook.yml -i inventory.ini                  # реальный запуск

# --- шаг 5: проверка ---
curl -s http://10.0.2.16/                 # с control node
curl -s http://127.0.0.1:8080/            # с хоста, через проброс порта
```

## Нюансы и наблюдения

- Поскольку вторая ВМ создана клонированием первой, на ней уже был установлен nginx
  (из П02). Чтобы прогон плейбука честно демонстрировал установку "с нуля" (как в
  лекции — `changed=1` на задаче установки), nginx был предварительно удалён на
  `devops-lab04-vm2` (`apt-get purge nginx nginx-common`) перед первым запуском плейбука.
- Сухой прогон `--check --diff` **до** установки nginx закономерно завершается ошибкой
  на задаче `Ensure nginx is running and enabled` (`Could not find the requested service
  nginx: host`) — в режиме `--check` пакет не устанавливается по-настоящему, поэтому
  systemd-юнит `nginx.service` ещё не существует и Ansible не может его опросить. Это
  ожидаемое ограничение dry-run при наличии зависимых друг от друга задач, а не ошибка
  плейбука — задача `diff` при этом корректно показывает, какой контент получит
  `index.nginx-debian.html` после реального запуска.
- Реальный запуск плейбука отрабатывает полностью чисто: `ok=4 changed=2 failed=0`
  (установка nginx + копирование `index.html`; запуск/enable сервиса — `ok`, так как
  `apt`-установка сама стартует и включает nginx).
- Проброс портов в VirtualBox **NAT Network** (в отличие от простого NAT на ВМ, как в
  П01/П02) настраивается один раз на уровне самой сети и явно указывает целевой
  внутренний IP (`VBoxManage natnetwork modify --port-forward-4 "имя:tcp:[host]:port:[guest]:port"`),
  поэтому пробросы `ssh-vm1`/`ssh-vm2`/`http-vm2` продолжают работать независимо от того,
  какая из двух ВМ запущена.
- SSH- и HTTP-доступ к `devops-lab01`, использовавшиеся в П01–П03 (`127.0.0.1:2223`),
  сохранены без изменений — несмотря на смену типа сетевого адаптера с обычного NAT на
  NAT Network, адрес и порт проброса для первой ВМ остались прежними.

## Скриншоты

1. `screenshots/01-step0-cron-and-clone.png` — отключение cron-задачи `weather.sh`;
   клонирование `devops-lab01` → `devops-lab04-vm2` с новым MAC-адресом.
2. `screenshots/02-nat-network-portforward.png` — создание NAT Network `lab4net`,
   переключение обеих ВМ на неё, проброс портов (ssh × 2, http).
3. `screenshots/03-static-ips-connectivity.png` — статические IP через netplan на
   обеих ВМ (`10.0.2.15` / `10.0.2.16`), успешный `ping` между ними.
4. `screenshots/04-runner-user-sudo.png` — создание пользователя `runner` на
   `devops-lab04-vm2`, `/etc/sudoers.d/runner`, `visudo -c`.
5. `screenshots/05-ssh-key-runner-access.png` — генерация SSH-ключа на control node,
   копирование на `runner@10.0.2.16`, вход и `sudo` без пароля.
6. `screenshots/06-ansible-install-playbook.png` — установка Ansible, содержимое
   `inventory.ini` и `playbook.yml`.
7. `screenshots/07-playbook-check-and-run.png` — `ansible -m ping`, сухой прогон
   `--check --diff`, реальный запуск плейбука (`changed=2`).
8. `screenshots/08-verify-curl.png` — проверка через `curl` с control node (`10.0.2.16`)
   и с хоста (`127.0.0.1:8080`) — задеплоенная Ansible-страница отдаётся в обоих случаях.
