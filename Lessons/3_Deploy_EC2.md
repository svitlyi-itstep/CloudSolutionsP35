# Розгортання додатку з GitHub на AWS EC2 із systemd


## Зміст

1. [Підготовка EC2](#підготовка-ec2)
2. [Клонування репозиторію з GitHub](#клонування-репозиторію-з-github)
3. [Налаштування та тестовий запуск додатку](#налаштування-та-тестовий-запуск-додатку)
4. [Створення systemd сервісу](#створення-systemd-сервісу)
5. [Керування сервісом](#керування-сервісом)
6. [Автозапуск при старті сервера](#автозапуск-при-старті-сервера)
7. [Оновлення додатку (повторний деплой)](#оновлення-додатку-повторний-деплой)
8. [Довідкові матеріали](#довідкові-матеріали)

---

## Підготовка EC2

Підключіться до сервера через EC2 Instance Connect у консолі AWS: `EC2 → Instances → Connect → EC2 Instance Connect`.

Після входу оновіть систему та встановіть необхідні залежності. Наприклад, для python-додатку це буде виглядати наступним чином.

**Amazon Linux 2023:**
```bash
sudo dnf update -y
sudo dnf install git python3 python3-pip -y
```

**Ubuntu:**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install git python3 python3-pip python3-venv -y
```

**Перевірка встановленого:**
```bash
git --version
python3 --version
pip3 --version
```

---

## Клонування репозиторію з GitHub

### Публічний репозиторій

```bash
cd /home/ec2-user
git clone https://github.com/your-username/your-repo.git
cd your-repo
```

### Приватний репозиторій — через SSH-ключ (рекомендовано)

**Генерація ключа на EC2:**
```bash
ssh-keygen -t ed25519 -C "ec2-deploy" -f ~/.ssh/id_ed25519 -N ""
```

**Перегляд та копіювання публічного ключа:**
```bash
cat ~/.ssh/id_ed25519.pub
```

**Додайте ключ у GitHub:**  
GitHub → **Settings → SSH and GPG keys → New SSH key** → вставте скопійований ключ.

**Клонування:**
```bash
git clone git@github.com:your-username/your-repo.git
cd your-repo
```

### Приватний репозиторій — через Personal Access Token

**Генерація токена:**  
GitHub → **Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token** → scope: ✅ repo

```bash
# Клонування з токеном
git clone https://your-username:YOUR_TOKEN@github.com/your-username/your-repo.git
cd your-repo
```

---

## Налаштування та тестовий запуск додатку

### Python-додаток (FastAPI / Django / бот)

```bash
cd /home/ec2-user/your-repo

# Створення віртуального середовища
python3 -m venv venv

# Активація
source venv/bin/activate

# Встановлення залежностей
pip install -r requirements.txt
```

**Перевірте, що додаток запускається вручну:**
```bash
# FastAPI
python3 -m uvicorn main:app --host 0.0.0.0 --port 8000

# Django
python3 manage.py runserver 0.0.0.0:8000

# Telegram-бот або будь-який скрипт
python3 bot.py
```

> ✅ Якщо додаток запустився без помилок — натисніть `Ctrl+C` і переходьте до налаштування systemd.  

### Node.js-додаток

```bash
# Встановлення Node.js (Amazon Linux 2023)
sudo dnf install nodejs -y

# Або через nvm (рекомендовано для гнучкості версій)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install --lts
nvm use --lts

# Встановлення залежностей
cd /home/ec2-user/your-repo
npm install

# Тестовий запуск
node index.js
# або
npm start
```

### Змінні середовища (.env)

Якщо додаток використовує `.env` — створіть файл вручну на сервері (не зберігайте секрети в репозиторії):

```bash
nano /home/ec2-user/your-repo/.env
```

Приклад вмісту:
```env
BOT_TOKEN=your_telegram_bot_token
DATABASE_URL=postgresql://user:pass@localhost/dbname
SECRET_KEY=your_secret_key
DEBUG=False
PORT=8000
```

---

## Створення systemd сервісу

**systemd** — система ініціалізації Linux, яка керує процесами, забезпечує їх автозапуск та перезапуск при збоях.

### Структура .service файлу

```bash
sudo nano /etc/systemd/system/myapp.service
```

**Загальний шаблон:**
```ini
[Unit]
Description=Назва вашого додатку
After=network.target

[Service]
User=ec2-user
WorkingDirectory=/home/ec2-user/your-repo
ExecStart=команда_запуску
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

### Приклади для різних типів додатків

**FastAPI (uvicorn):**
```ini
[Unit]
Description=FastAPI Application
After=network.target

[Service]
User=ec2-user
WorkingDirectory=/home/ec2-user/your-repo
ExecStart=/home/ec2-user/your-repo/venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

**Django (gunicorn):**
```ini
[Unit]
Description=Django Application via Gunicorn
After=network.target

[Service]
User=ec2-user
WorkingDirectory=/home/ec2-user/your-repo
ExecStart=/home/ec2-user/your-repo/venv/bin/gunicorn \
          --workers 3 \
          --bind 0.0.0.0:8000 \
          myproject.wsgi:application
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

**Telegram-бот (Python):**
```ini
[Unit]
Description=Telegram Bot
After=network.target

[Service]
User=ec2-user
WorkingDirectory=/home/ec2-user/your-repo
ExecStart=/home/ec2-user/your-repo/venv/bin/python3 bot.py
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

**Node.js-додаток:**
```ini
[Unit]
Description=Node.js Application
After=network.target

[Service]
User=ec2-user
WorkingDirectory=/home/ec2-user/your-repo
ExecStart=/usr/bin/node index.js
Restart=always
RestartSec=5
Environment=NODE_ENV=production
Environment=PORT=3000
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

---

### Пояснення ключових параметрів

| Параметр | Опис |
|---|---|
| `Description` | Назва сервісу (відображається в логах і status) |
| `After=network.target` | Запускати лише після ініціалізації мережі |
| `User` / `Group` | Від імені якого користувача запускати |
| `WorkingDirectory` | Робоча директорія (аналог `cd` перед запуском) |
| `ExecStart` | Команда запуску (обов'язково **абсолютний** шлях) |
| `Restart=always` | Перезапускати при будь-якому завершенні процесу |
| `Restart=on-failure` | Перезапускати лише при збої (не при штатній зупинці) |
| `RestartSec=5` | Затримка перед перезапуском (секунди) |
| `StandardOutput=journal` | Направляти stdout у системний журнал |
| `StandardError=journal` | Направляти stderr у системний журнал |
| `Environment=KEY=VALUE` | Задати змінні середовища |
| `EnvironmentFile=.env` | Завантажити змінні з файлу |

**Завантаження змінних з .env файлу:**
```ini
[Service]
EnvironmentFile=/home/ec2-user/your-repo/.env
ExecStart=/home/ec2-user/your-repo/venv/bin/python3 bot.py
```

---

## Керування сервісом

### Активація нового сервісу

Після створення або зміни `.service` файлу **обов'язково** виконайте:

```bash
# Перечитати конфігурацію systemd
sudo systemctl daemon-reload

# Запустити сервіс
sudo systemctl start myapp

# Перевірити статус
sudo systemctl status myapp
```

### Основні команди управління

```bash
# Запуск
sudo systemctl start myapp

# Зупинка
sudo systemctl stop myapp

# Перезапуск
sudo systemctl restart myapp

# Перезавантаження конфігурації без зупинки (якщо підтримується)
sudo systemctl reload myapp

# Статус сервісу
sudo systemctl status myapp

# Список всіх активних сервісів
sudo systemctl list-units --type=service --state=active
```

### Перегляд логів

```bash
# Всі логи сервісу
journalctl -u myapp

# Останні 50 рядків
journalctl -u myapp -n 50

# Логи в реальному часі (live tail)
journalctl -u myapp -f

# Логи за сьогодні
journalctl -u myapp --since today

# Логи за останню годину
journalctl -u myapp --since "1 hour ago"

# Логи за конкретний проміжок часу
journalctl -u myapp --since "2024-06-15 10:00" --until "2024-06-15 12:00"
```

### Що означає вивід `systemctl status`

```
● myapp.service - FastAPI Application
     Loaded: loaded (/etc/systemd/system/myapp.service; enabled)
     Active: active (running) since Sun 2024-06-15 10:00:00 UTC; 2h ago
   Main PID: 1234 (python3)
      Tasks: 4 (limit: 1112)
     Memory: 45.2M
        CPU: 1.234s
     CGroup: /system.slice/myapp.service
             └─1234 /home/ec2-user/your-repo/venv/bin/python3 main.py
```

| Поле | Значення |
|---|---|
| `Loaded: enabled` | Автозапуск увімкнено ✅ |
| `Loaded: disabled` | Автозапуск вимкнено ❌ |
| `Active: active (running)` | Сервіс працює ✅ |
| `Active: failed` | Сервіс впав ❌ |
| `Active: activating` | Сервіс запускається ⏳ |
| `Main PID` | ID процесу |

---

## Автозапуск при старті сервера

### Увімкнення автозапуску

```bash
# Увімкнути автозапуск
sudo systemctl enable myapp

# Вивід:
# Created symlink /etc/systemd/system/multi-user.target.wants/myapp.service
# → /etc/systemd/system/myapp.service
```

```bash
# Вимкнути автозапуск (сервіс залишиться запущеним, але не стартує після ребуту)
sudo systemctl disable myapp
```

### Увімкнення + негайний запуск однією командою

```bash
sudo systemctl enable --now myapp
```

### Перевірка автозапуску

```bash
# Чи увімкнено автозапуск?
sudo systemctl is-enabled myapp
# enabled  ← автозапуск увімкнено
# disabled ← автозапуск вимкнено

# Перевірка після симульованого ребуту
sudo reboot

# Після перезавантаження підключіться знову і перевірте
sudo systemctl status myapp
```

### Як працює enable під капотом

`systemctl enable` створює **символічне посилання** у директорії `wants`:

```
/etc/systemd/system/multi-user.target.wants/myapp.service
    → /etc/systemd/system/myapp.service
```

Це означає, що при досягненні рівня запуску `multi-user.target` (нормальна робота сервера) — systemd автоматично запустить ваш сервіс.

---

## Оновлення додатку (повторний деплой)

### Ручне оновлення

```bash
cd /home/ec2-user/your-repo

# Отримати зміни з GitHub
git pull origin main

# Активувати venv та оновити залежності (якщо змінились)
source venv/bin/activate
pip install -r requirements.txt

# Перезапустити сервіс
sudo systemctl restart myapp

# Перевірити, що все працює
sudo systemctl status myapp
```

### Скрипт для швидкого деплою

Створіть файл `deploy.sh` у директорії проєкту:

```bash
nano /home/ec2-user/deploy.sh
```

```bash
#!/bin/bash

set -e  # Зупинити скрипт при будь-якій помилці

APP_DIR="/home/ec2-user/your-repo"
SERVICE_NAME="myapp"

echo "🚀 Починаємо деплой..."

# Оновлення коду
cd "$APP_DIR"
git pull origin main
echo "✅ Код оновлено"

# Оновлення залежностей
source venv/bin/activate
pip install -r requirements.txt --quiet
echo "✅ Залежності оновлено"

# Перезапуск сервісу
sudo systemctl restart "$SERVICE_NAME"
sleep 2

# Перевірка статусу
if sudo systemctl is-active --quiet "$SERVICE_NAME"; then
    echo "✅ Сервіс запущено успішно"
    echo "📋 Статус:"
    sudo systemctl status "$SERVICE_NAME" --no-pager -l
else
    echo "❌ Сервіс не запустився. Перевірте логи:"
    journalctl -u "$SERVICE_NAME" -n 20 --no-pager
    exit 1
fi

echo "🎉 Деплой завершено: $(date)"
```

```bash
# Зробити скрипт виконуваним
chmod +x /home/ec2-user/deploy.sh

# Запуск деплою
~/deploy.sh
```

### Дозвіл на restart без пароля

За замовчуванням `sudo systemctl restart` потребує пароля. Щоб скрипт деплою та GitHub Actions працювали без нього:

```bash
sudo visudo -f /etc/sudoers.d/myapp
```

Додайте рядок:
```
ec2-user ALL=(ALL) NOPASSWD: /bin/systemctl restart myapp, /bin/systemctl status myapp, /bin/systemctl start myapp, /bin/systemctl stop myapp
```

### Автоматичний деплой через GitHub Actions

Створіть файл `.github/workflows/deploy.yml` у репозиторії:

```yaml
name: Deploy to EC2

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd /home/ec2-user/your-repo
            git pull origin main
            source venv/bin/activate
            pip install -r requirements.txt --quiet
            sudo systemctl restart myapp
            sudo systemctl status myapp --no-pager
```

**Secrets для GitHub Actions:**

| Secret | Значення |
|---|---|
| `EC2_HOST` | Public IP адреса EC2 |
| `EC2_USER` | `ec2-user` |
| `EC2_SSH_KEY` | Вміст приватного `.pem` ключа |

Додати secrets: GitHub репозиторій → **Settings → Secrets and variables → Actions → New repository secret**.

## Довідкові матеріали

- [systemd Service Manual](https://www.freedesktop.org/software/systemd/man/systemd.service.html) — повний довідник параметрів `.service` файлу.
- [systemctl Manual](https://www.freedesktop.org/software/systemd/man/systemctl.html) — всі команди керування.
- [journalctl Manual](https://www.freedesktop.org/software/systemd/man/journalctl.html) — робота з логами.
- [GitHub Actions — SSH Deploy](https://github.com/appleboy/ssh-action) — action для деплою через SSH.

### Корисні команди

| Дія | Команда |
|---|---|
| Створити сервіс | `sudo nano /etc/systemd/system/myapp.service` |
| Перечитати конфіги | `sudo systemctl daemon-reload` |
| Запустити | `sudo systemctl start myapp` |
| Зупинити | `sudo systemctl stop myapp` |
| Перезапустити | `sudo systemctl restart myapp` |
| Статус | `sudo systemctl status myapp` |
| Увімкнути автозапуск | `sudo systemctl enable myapp` |
| Вимкнути автозапуск | `sudo systemctl disable myapp` |
| Увімкнути + запустити | `sudo systemctl enable --now myapp` |
| Перевірити автозапуск | `sudo systemctl is-enabled myapp` |
| Логи сервісу | `journalctl -u myapp -f` |
| Перевірити .service файл | `sudo systemd-analyze verify /etc/systemd/system/myapp.service` |

