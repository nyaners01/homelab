# ansible

Применяет конфиги из `services/` на сервер. Запускается из WSL (Ubuntu 24.04)
на ПК.

## Структура

```
ansible.cfg                     настройки, inventory по умолчанию
requirements.yml                коллекции: community.docker
inventory/
  hosts.yml                     сервер orangepizero3
  group_vars/all/
    vars.yml                    открытые переменные и ссылки на секреты
    vault.yml                   секреты, зашифрованы Ansible Vault
vault.example.yml               какие ключи должны быть в vault.yml
vault-pass.sh                   отдаёт Ansible пароль от Vault из .ansible-vault-pass
playbooks/
  site.yml                      всё целиком, в порядке зависимостей
  docker.yml                    роль docker
  tls.yml                       роль tls
  nginx.yml                     роль nginx
  apps.yml                      роль compose_app для каждого приложения из apps
  backup.yml                    роли notify, backup и offsite
  hardening.yml                 роли ssh, services, firewall
roles/
  docker/                       репозиторий и пакеты Docker, daemon.json
  tls/                          certbot, таймер продления, проверка сертификатов
  nginx/                        конфиги nginx, сайты, nginx -t и reload
  compose_app/                  /opt/<app>: compose, env из шаблона, up
  notify/                       уведомления в Telegram об ошибках служб
  backup/                       скрипт, служба и таймер ночного бэкапа
  offsite/                      rclone, отправка бэкапов в Google Drive после backup
  ssh/                          настройки sshd, sshd -t и reload
  services/                     остановка и маскировка лишних служб
  firewall/                     nftables, применение со страховкой
```

Приложения перечислены в `apps` в `inventory/group_vars/all/vars.yml`.
Роль `compose_app` никогда не создаёт каталог `data` сама и падает, если
его нет: пустой каталог у vaultwarden означает пустое хранилище.

## Подготовка WSL, один раз

**Конфиг.** Репозиторий лежит на `/mnt/d`, а этот каталог доступен на запись
всем. Ansible в таком каталоге игнорирует `ansible.cfg`, поэтому путь к нему
задаётся явно:

```sh
echo 'export ANSIBLE_CONFIG=/mnt/d/my_projects/homelab/ansible/ansible.cfg' >> ~/.bashrc
```

**SSH-ключ.** Ключ для сервера лежит в Windows, в WSL его нет. С `/mnt/c`
ssh ключ не примет из-за прав доступа, поэтому его нужно скопировать:

```sh
mkdir -p ~/.ssh && chmod 700 ~/.ssh
cp /mnt/c/Users/artem/.ssh/id_ed25519 ~/.ssh/
chmod 600 ~/.ssh/id_ed25519
ssh-keyscan 192.168.1.226 >> ~/.ssh/known_hosts
```

**Коллекции.**

```sh
ansible-galaxy collection install -r requirements.yml
```

**Vault.** Секреты хранятся в `inventory/group_vars/all/vault.yml` в
зашифрованном виде, файл коммитится. Пароль от него в репозиторий не
попадает никогда. Он хранится в файле `ansible/.ansible-vault-pass`,
файл игнорируется git.

```sh
ansible-vault create inventory/group_vars/all/vault.yml
```

Ключи взять из `vault.example.yml`. Редактировать потом:

```sh
ansible-vault edit inventory/group_vars/all/vault.yml
```

## Запуск

Пароль от Vault Ansible берёт сам через `vault-pass.sh`, указывать его не
нужно:

```sh
cd /mnt/d/my_projects/homelab/ansible
ansible homelab -m ping
```

Напрямую `.ansible-vault-pass` в `vault_password_file` указать нельзя. На `/mnt/d`
любой файл выглядит исполняемым, и Ansible пытается запустить его как
скрипт. Поэтому между ними стоит `vault-pass.sh`.

Любое изменение на сервере сначала прогоняется с `--check --diff`:

```sh
ansible-playbook playbooks/site.yml --check --diff
ansible-playbook playbooks/site.yml --diff
```

Одно приложение: `-e app_filter=vaultwarden`. Пересоздать контейнеры,
например после смены настроек логов Docker: `-e app_recreate=true`.

**Файрвол применяется со страховкой.** Перед загрузкой новых правил ставится
таймер, который снимет их через 5 минут. После загрузки Ansible открывает
новое SSH-соединение. Если оно прошло, таймер отменяется. Если нет, правила
снимаются сразу. В режиме `--check` правила не применяются, показывается
только diff файла.
