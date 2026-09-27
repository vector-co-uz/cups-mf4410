# 🖨️ CUPS + Canon i-SENSYS MF4410 (Docker)

Готовый Docker-образ CUPS с установленным драйвером Canon UFR II / UFRII LT для печати со старого принтера **Canon i-SENSYS MF4410** по USB.

![GHCR](https://img.shields.io/badge/ghcr.io-vector--co--uz%2Fcups--mf4410-blue?logo=github)
![CUPS](https://img.shields.io/badge/CUPS-2.x-lightgrey?logo=cups)
![Base](https://img.shields.io/badge/base-debian%3Abookworm--slim-red?logo=debian)
![Built with Claude](https://img.shields.io/badge/built%20with-Claude-D97757?logo=anthropic&logoColor=white)

---

## Содержание

- [Быстрый старт](#-быстрый-старт)
- [docker run](#-docker-run)
- [docker compose](#-docker-compose)
- [Сборка из исходников](#️-сборка-из-исходников)
- [Проброс USB-принтера](#-проброс-usb-принтера)
- [Добавление принтера в CUPS](#-добавление-принтера-в-cups)
- [Диагностика](#-диагностика)

---

## 🚀 Быстрый старт

Образ уже собран и лежит в GitHub Container Registry — сборка не нужна:

```bash
docker pull ghcr.io/vector-co-uz/cups-mf4410:latest
```

Дальше — любым из двух способов ниже.

---

## 🐳 docker run


```bash
#!/usr/bin/env bash
docker run -d \
  --name cups-canon-mf4410 \
  --restart unless-stopped \
  -p 631:631 \
  -v cups_config:/etc/cups \
  -v /dev/bus/usb:/dev/bus/usb \
  --cap-add SYS_ADMIN \
  ghcr.io/vector-co-uz/cups-mf4410:latest
```
---

## 📦 docker compose

Файл [`docker-compose.yml`](./docker-compose.yml):

```bash
docker compose up -d
```

По умолчанию используется готовый образ из `ghcr.io`. Если хотите собирать локально — см. раздел ниже и переключите `image:` на `build: .` в файле.

---

## 🛠️ Сборка из исходников

Нужна, только если хотите пересобрать образ сами (например, с другой версией драйвера).

> ⚠️ Архив драйвера Canon — проприетарный файл, в репозитории его нет (см. `.gitignore`). Скачайте вручную.

1. Скачайте драйвер **UFR II / UFRII LT для Linux** с официальной страницы модели:
   👉 https://www.canon-europe.com/support/consumer/products/printers/i-sensys/mf-series/i-sensys-mf4410.html
   (вкладка **Drivers** → фильтр ОС **Linux 32-bit / 64-bit**)

2. Положите скачанный файл **как есть** (`.zip`, `.tar.gz` или голые `.deb`) в папку `drivers/`:

   ```
   cups-mf4410/
   ├── Dockerfile
   ├── docker-compose.yml
   ├── README.md
   └── drivers/
       └── o151en_linux_UFRII_v310.zip   ← сюда
   ```

3. Соберите:

   ```bash
   docker compose build --no-cache
   ```

   Сборка сама распакует архив, найдёт `install.sh` и установит драйвер. В логе будет видна вся структура архива и полный вывод установщика — если модель не определится, ошибка будет видна сразу.

---

```
- Логин: `printadmin` / пароль: `printadmin` (обязательно смените в
  Dockerfile перед реальным использованием — там `chpasswd` в конце сборки)
```
---
## 🔌 Проброс USB-принтера

### Unraid

Docker на Unraid обычно видит `/dev/bus/usb` напрямую — достаточно смонтировать его как volume или проброс как "/dev/bus/usb/003/010:/dev/bus/usb/003/010".

---

## 🖨️ Добавление принтера в CUPS

Веб-интерфейс: **http://<host>:631** (логин `printadmin` / пароль `printadmin` — смените в проде).

Или через CLI:

```bash
docker exec -it cups-canon-mf4410 bash

lpinfo -v                                    # найти USB URI принтера
lpinfo -m | grep -i canon                    # список доступных PPD

lpadmin -p MF4410 -E \
  -v usb://Canon/MF4400%20series \
  -m <НАЙДЕННЫЙ_PPD_ИЗ_СПИСКА_ВЫШЕ>

lpadmin -d MF4410                            # сделать принтером по умолчанию
echo "test" | lp -d MF4410                   # тестовая печать
```

> Модель в CUPS обычно фигурирует как **MF4400 series** — это общий драйвер для всей линейки MF4400/MF4410/MF4420/MF4450, отдельного PPD именно под «4410» может не быть, и это нормально.

---

## 🔍 Диагностика

| Проблема | Проверка |
|---|---|
| Принтер не виден в контейнере | `docker exec -it cups-canon-mf4410 lsusb` |
| PPD с MF44xx не найден | `docker exec -it cups-canon-mf4410 bash -c "find / -iname '*.ppd*' 2>/dev/null \| xargs grep -l MF44"` |
| Docker в LXC не стартует | `features: nesting=1,keyctl=1` в конфиге LXC на хосте |
| `overlay2` не работает в LXC | storage-driver `vfs` в `/etc/docker/daemon.json` |
| USB-устройство пропадает после переподключения | пробрасывайте **всю шину** `/dev/bus/usb`, а не конкретный `bus/device` |

---

## 📄 Лицензия

Конфигурация (`Dockerfile`, `docker-compose.yml`) распространяется свободно. Сам драйвер принтера — собственность **Canon Inc.** и не входит в этот репозиторий; используется в соответствии с лицензией Canon.
