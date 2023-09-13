# KITTYGRAM – СОЦИАЛЬНАЯ СЕТЬ ВЛАДЕЛЬЦЕВ КОТИКОВ. УЧЕБНЫЙ ПРОЕКТ Я.ПРАКТИКУМ.

## ОПИСАНИЕ ПРОЕКТА
Пользователи могут регистрироваться, добавлять, редактировать, удалять свои записи о котиках с фотографиями и без, могут просматривать записи чужих котиков. Стандартная админка Django.  
Цель проекта: научиться работать с удаленным сервером, сделать проект доступным для пользователей в интернете.

## ТЕХНОЛОГИИ
- Python 3.9
- Django 3.2.3
- Django REST Framework 3.12.4
- Linux Ubuntu
- Yandex Cloud
- Nginx
- Gunicorn

## ЗАПУСТИТЬ ПРОЕКТ НА УДАЛЕННОМ СЕРВЕРЕ

### НАСТРОИТЬ GITHUB И КЛОНИРОВАТЬ ПРОЕКТ НА СЕРВЕР

Обновить пакетный менеджер
```sh
sudo apt update
```
  

Проверить версию git
```sh
git --version
```
  
 
Сгенерировать SSH для GitHub
```sh
ssh-keygen
```
Выбрать нужную директорию для хранения ключей на удаленном сервере, придумать пассфразу для ключа
  

Вывести открытый ключ в терминале
```sh
cat .ssh/id_rsa.pub
```
Скопировать ключ от символов ssh-rsa, включительно, и до конца
  

GitHub Setting -> SSH and GPG keys -> New SSH key  
Title: `<читаемый_заголовок_ключа>`  
Key: `<вставить_скопированный_из_терминала_ключ>`  
Add SSH key
  

Клонировать репозиторий на удаленный сервер
```sh
git clone https://github.com/beta-Lilith/infra_sprint1.git
```
  

### ПОДГОТОВИТЬ BACKEND

Установить на сервер пакетный менеджер и утилиту для создания виртуального окружения
```sh
sudo apt install python3-pip python3-venv -y
```
  

Установить и активировать окружение
```sh
python3 -m venv venv
source venv/script/activate
```
  

Обновить pip и установить зависимости
```sh
python -m pip install --upgrade pip
pip install -r requirements.txt
```
  

В директрории `/backend/kittygram_backend/` создать файл `.env` по шаблону
```sh
SECRET_KEY = <ваш_сгенерированный_ключ>
```
  

Выполнить миграции и создать суперюзера
```sh
python manage.py migrate
python manage.py createsuperuser
```
  

В файле `settings.py` отредактировать `ALLOWED_HOSTS` по шаблону
```sh
ALLOWED_HOSTS = ['внешний_IP_сервера', '127.0.0.1', 'localhost', 'домен']
```
  

Создать папку `media` в директории `/var/www/infra_sprint1/`
```sh
sudo mkdir /var/www/infra_sprint1/media
```
  

Назначить текущего пользователя владельцем директории `media`
```sh
sudo chown -R <имя_пользователя>/var/www/infra_sprint1/media/
```
  

Собрать и скопировать статику backend
```sh
python manage.py collectstatic
sudo cp -r home/<имя_пользователя>/infra_sprint1/backend/static_backend/ /var/www/infra_sprint1
```
  

### ПОДГОТОВИТЬ FRONTEND

Установить Node.js на сервер
```sh
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash - &&\
sudo apt-get install -y nodejs
```
  

Убедиться, что npm тоже установлен
```sh
npm -v
```
  

Установить зависимости из директории `infra_sprint1/frontend`
```sh
npm i
```
Предупреждения об устаревших зависимостях и уязвимостях устранять не нужно, написаны скрипты для сборки этого React-приложения
  

Собрать и скопировать статику frontend
```sh
npm run build
sudo cp -r home/<имя_пользователя>/infra_sprint1/frontend/build/. /var/www/infra_sprint1/
```
  

### УСТАНОВИТЬ И НАСТРОИТЬ WSGI-СЕРВЕР GUNICORN

При активированном виртуальном окружении установить Gunicorn
```sh
pip install gunicorn==20.1.0
```
  

Создать файл конфигурации юнита systemd для Gunicorn
```sh
sudo nano /etc/systemd/system/gunicorn_название_проекта.service
```
  

Подставить отредактированный код по шаблону
```sh
[Unit]

Description=gunicorn daemon
After=network.target

[Service]

User=<имя_пользователя>
WorkingDirectory=/home/<имя_пользователя>/infra_sprint1/backend/
ExecStart=/home/<имя_пользователя>/infra_sprint1/backend/venv/bin/gunicorn --bind 0.0.0.0:<свободный_порт> kittygram_backend.wsgi

[Install]

WantedBy=multi-user.target
```
  

Запустить и добавить в список автозапуска системы созданный юнит
```sh
sudo systemctl start gunicorn_название_проекта
sudo systemctl enable gunicorn_название_проекта
```
  

Проверить статус
```sh
sudo systemctl status gunicorn_название_проекта
```
  

### УСТАНОВИТЬ И НАСТРОИТЬ ВЕБ- И ПРОКСИ-СЕРВЕР NGINX

Установить и запустить Nginx
```sh
sudo apt install nginx -y
sudo systemctl start nginx
```
  

Открыть файл настроек Nginx
```sh
sudo nano /etc/nginx/sites-enabled/default
```
  

Подставить отредактированный код по шаблону
```sh
server { 
    listen 80;
    server_tokens off;
    server_name <домен и/или ip_вашего_сервера>; 
    client_max_body_size 20M;
    location /api/ { 
        proxy_pass http://127.0.0.1:<свободный_порт>; 
    } 
    location /admin/ { 
        proxy_pass http://127.0.0.1:<свободный_порт>; 
    } 
    location /media/ {
        alias /var/www/infra_sprint1/media/;
    }
    location / { 
        root /var/www/infra_sprint1; 
        index index.html index.htm; 
        try_files $uri /index.html; 
    } 
}
```
  

Проверить файл на корректность и перезагрузить конфигурацию Nginx
```sh
sudo nginx -t
sudo systemctl reload nginx
```
  

### НАСТРОИТЬ ФАЙРВОЛ UFW

Активировать разрешение принимать запросы только на порты 80, 443 и 22
```sh
sudo ufw allow 'Nginx Full'
sudo ufw allow OpenSSH
```
  

Включить файрвол и проверить статус
```sh
sudo ufw enable
sudo ufw status
```
  

### ПОЛУЧИТЬ И НАСТРОИТЬ SSL-СЕРТИФИКАТ
(на примере [Let’s Encrypt](https://letsencrypt.org/))

Установить certbot
```sh
sudo apt install snapd
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```
  

Запустить certbot и получить сертификат
```sh
sudo certbot –nginx
```
Выбрать номер нужного домена или не вводить ничего и нажать Enter
  

Проверить измененный файл конфигурации и перезапустить Nginx
```sh
sudo nano /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```
  

### ПРОВЕРИТЬ АВТОМАТИЧЕСКОЕ ОБНОВЛЕНИЕ SSL-СЕРТИФИКАТА

Узнать актуальный статус
```sh
sudo certbot certificates
```
  

Убедиться, что сертификат обновляется автоматически
```sh
sudo certbot renew --dry-run
```
  

Вручную сертификат можно обновить командой
```sh
sudo certbot renew --pre-hook "service nginx stop" --post-hook "service nginx start"
```
  

## АВТОР
Оскомова Ксения
