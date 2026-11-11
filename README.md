# homelab

Домашняя инфраструктура: один сервер на Orange Pi Zero3, на нём nginx как
обратный прокси и сервисы в Docker. Репозиторий хранит желаемое состояние
конфигураций и Ansible, который их применяет.

## Сервер

| Параметр         | Значение                                            |
| ---------------- | --------------------------------------------------- |
| Хост             | `orangepizero3`                                     |
| Железо           | Orange Pi Zero3, Allwinner H618, ARM64              |
| ОС               | Armbian community, stable сборка на Ubuntu 26.04    |
| Пользователь SSH | `nyaners`                                           |
| LAN              | `192.168.1.0/24`, адрес `192.168.1.226` через DHCP  |

## Схема

```
Интернет
   |  80, 443
   v
Роутер (OpenWrt): проброс 80 и 443 на 192.168.1.226
   |
   v
orangepizero3 (192.168.1.226)
   |
   +-- nginx (systemd) :80 :443 -- TLS от certbot
   |     |
   |     +-- vaultwarden.nyaners.ru --> 127.0.0.1:8080
   |
   +-- docker
   |     +-- vaultwarden   127.0.0.1:8080 -> 80
   |     +-- findair       портов нет, ходит в Telegram API сам
   |
   +-- sshd :22  <-- ПК (Windows 11, Ansible из WSL) по LAN
   |
   +-- rclone --> Google Drive, архивы бэкапов в зашифрованном виде
```

## Сервисы

| Сервис      | Где работает | Порт на сервере | Домен                  | Конфиг и описание                                                |
| ----------- | ------------ | --------------- | ---------------------- | ---------------------------------------------------------------- |
| nginx       | systemd      | 80, 443         | vaultwarden.nyaners.ru | [services/network/nginx](services/network/nginx/README.md)       |
| vaultwarden | Docker       | 127.0.0.1:8080  | vaultwarden.nyaners.ru | [services/apps/vaultwarden](services/apps/vaultwarden/README.md) |
| find-air    | Docker       | нет             | нет                    | [services/apps/find-air](services/apps/find-air/README.md)       |

Кроме них на сервере слушает только `sshd` (22) и `systemd-resolved` (53,
только localhost). Сервер подключён к роутеру по Wi-Fi, адрес выдаёт DHCP
роутера с закреплением за сервером.

## Защита

- **Файрвол** nftables, своя таблица `inet homelab`: снаружи открыты только
  80 и 443, SSH только из `192.168.1.0/24`, остальное запрещено. Правила
  Docker живут в других таблицах и не затрагиваются.
- **SSH** только по ключу, root не пускается, X11 выключен.
- **Лишние службы** из образа Armbian остановлены и замаскированы: rpcbind,
  cups, avahi, bluetooth, ModemManager.
- **Автообновления безопасности** через unattended-upgrades.
- **Логи Docker** ротируются: по 10 МБ, 3 файла на контейнер.

## Бэкапы

Каждую ночь в 00:30 UTC (03:30 по Москве) таймер `homelab-backup.timer`
собирает архив данных каждого приложения и каталога `/etc/letsencrypt`
(сертификаты нужны сразу после восстановления, иначе ждать лимит Let's
Encrypt):

- базы SQLite копируются онлайн через `sqlite3 .backup`, без остановки
  контейнера, и проверяются `PRAGMA integrity_check`;
- остальные файлы из `data` копируются как есть, кроме кэшей;
- архивы лежат в `/var/backups/homelab/daily`, по воскресеньям копия
  уходит в `weekly`;
- у каждого приложения хранятся 7 последних ежедневных и 4 последних
  недельных архива. Старые удаляются только после успешного нового, поэтому
  неудачные запуски ничего не стирают;
- если бэкап упал, `homelab-notify@` присылает в Telegram имя службы и
  журнал её последнего запуска. Проверить доставку:
  `sudo systemctl start homelab-notify@test.service`.

### Копия в облаке

После успешного локального бэкапа systemd запускает
`homelab-offsite.service` (роль `offsite`). Он отправляет `daily` и `weekly`
в отдельную папку Google Drive через rclone:

- remote `gdrive-backups` — Google Drive со своим OAuth-клиентом,
  `scope = drive.file` и `root_folder_id`: rclone видит только свою папку;
- remote `gdrive-backups-encrypted` — `crypt` поверх него. Содержимое и
  имена файлов шифруются на сервере, Google хранит только шифр;
- команда `rclone copy`, а не `sync`: удаление локально не удаляет копию в
  облаке. Старые архивы удаляет отдельный `rclone delete --min-age` мимо
  корзины Drive, срок тот же, что локально: в облаке 7 ежедневных и 4
  недельных архива;
- если отправка упала, приходит сообщение в Telegram, как и для бэкапа.

Имена remote здесь и ниже — значения по умолчанию из
`offsite_rclone_remote_name` в `ansible/roles/offsite/defaults/main.yml`.
Зашифрованный remote называется так же с суффиксом `-encrypted`.

Отправить вручную и посмотреть, что лежит в облаке:

```sh
sudo homelab-offsite
journalctl -u homelab-offsite -n 20
sudo rclone lsl gdrive-backups-encrypted:backup
```

Скачать и расшифровать архив на ПК: собрать на ПК `rclone.conf` с теми же
двумя remote из набора для выживания (см. ниже) и выполнить

```sh
rclone copy <remote-crypt>:backup/daily/<app>_<дата>.tar.gz ~/Downloads
```

### Набор для выживания

Чтобы восстановиться, если сервер и vaultwarden потеряны одновременно,
нужны: пароль Ansible Vault, оба пароля crypt (`password` и `password2`),
OAuth-клиент и токен rclone, пароль от аккаунта Google и коды
восстановления 2FA. Без паролей crypt архивы в облаке не расшифровать
никак.

Набор существует и хранится офлайн. Его **нет** на сервере, в
vaultwarden, в этом репозитории, в Google Drive и на ПК в открытом виде.

Полная репетиция восстановления — этап 5 [roadmap](docs/roadmap.md).

Запустить бэкап вручную и посмотреть результат:

```sh
sudo systemctl start homelab-backup
journalctl -u homelab-backup -n 20
sudo ls -lh /var/backups/homelab/daily
```

Восстановить приложение из архива:

```sh
cd /opt/<app>
sudo docker compose down
sudo mv data data.broken
sudo tar -xzf /var/backups/homelab/daily/<app>_<дата>.tar.gz
sudo docker compose up -d
```

Архив содержит каталог `data` целиком, права и владельцы сохраняются.
Когда приложение проверено, `data.broken` можно удалить.

## Структура репозитория

```
services/
  network/nginx/      конфиги nginx, копия /etc/nginx
  apps/<сервис>/      docker-compose.yaml и templates/ с шаблонами env
ansible/              Ansible, см. ansible/README.md
```

Каждый сервис описан в своём README рядом с конфигами. Удаляешь сервис,
удаляешь папку целиком вместе с документацией.

## Соглашения

- Сервисы на сервере живут в `/opt/<сервис>`: там compose, отрендеренный
  env и каталог `data`.
- Секреты в git не попадают. Файлы `*.env` игнорируются, вместо них лежат
  шаблоны `templates/*.env.j2` с переменными Jinja, значения хранятся в
  Ansible Vault.
- Репозиторий описывает желаемое состояние, а не снимок сервера. Расхождения
  устраняются применением конфигов из репозитория, а не правкой репозитория
  под сервер.
- Рабочую конфигурацию на сервере не трогаем руками, пока не согласовано.
  Первый прогон Ansible только с `--check --diff`.

## План

Этапы, открытые задачи и отложенные решения в [docs/plan.md](docs/plan.md),
результаты ревью в [docs/review-2026-09-27.md](docs/review-2026-09-27.md),
учебный план развития инфраструктуры в [docs/roadmap.md](docs/roadmap.md).
