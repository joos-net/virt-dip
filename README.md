# Домашнее задание к занятию 5. «Практическое применение Docker»
# `Островский Евгений`

### Инструкция к выполнению

1. Для выполнения заданий обязательно ознакомьтесь с [инструкцией](https://github.com/netology-code/devops-materials/blob/master/cloudwork.MD) по экономии облачных ресурсов. Это нужно, чтобы не расходовать средства, полученные в результате использования промокода.
3. Своё решение к задачам оформите в вашем GitHub репозитории.
4. В личном кабинете отправьте на проверку ссылку на .md-файл в вашем репозитории.
5. Сопроводите ответ необходимыми скриншотами.

---
## Примечание: Ознакомьтесь со схемой виртуального стенда [по ссылке](https://github.com/netology-code/shvirtd-example-python/blob/main/schema.pdf)

---

## Задача 1
1. Сделайте в своем github пространстве fork репозитория ```https://github.com/netology-code/shvirtd-example-python/blob/main/README.md```.   
2. Создайте файл с именем ```Dockerfile.python``` для сборки данного проекта. Используйте базовый образ ```python:3.9-slim```. Протестируйте корректность сборки. Не забудьте dockerignore.
```Dockerfile
FROM python:3.9-slim

RUN python3 -m venv venv
RUN . venv/bin/activate

COPY ./requirements.txt .
RUN pip install -r requirements.txt

COPY ./main.py .
CMD python main.py

EXPOSE "5000"
```
```
docker build -t ex-python -f Dockerfile.python .
```
3. (Необязательная часть, *) Изучите инструкцию в проекте и запустите web-приложение без использования docker в venv. (Mysql БД можно запустить в docker run).
```
docker run -d -p 3306:3306 --name mysql -e MYSQL_ROOT_PASSWORD=root mysql:8
python main.py
curl http://127.0.0.1:5000
```
![1-test](https://github.com/joos-net/virt-dip/blob/main/1-test.png)
4. (Необязательная часть, *) По образцу предоставленного python кода внесите в него исправление для управления названием используемой таблицы через ENV переменную.
<details>
  <summary>main.py</summary>
  
```Python
from flask import Flask
from flask import request
import os
import mysql.connector
from datetime import datetime

app = Flask(__name__)
db_host=os.environ.get('DB_HOST')
db_user=os.environ.get('DB_USER')
db_password=os.environ.get('DB_PASSWORD')
db_database=os.environ.get('DB_NAME')
db_table=os.environ.get('DB_TABLE')

mydb = mysql.connector.connect(
host=db_host,
user=db_user,
password=db_password,
)
mycursor = mydb.cursor()

# # SQL-запрос для создания базы в БД
mycursor.execute(f"""CREATE DATABASE IF NOT EXISTS {db_database}""")

# Подключение к базе данных MySQL
db = mysql.connector.connect(
host=db_host,
user=db_user,
password=db_password,
database=db_database,
autocommit=True )
cursor = db.cursor()

# SQL-запрос для создания таблицы в БД
create_table_query = f"""
CREATE TABLE IF NOT EXISTS {db_database}.{db_table} (
id INT AUTO_INCREMENT PRIMARY KEY,
request_date DATETIME,
request_ip VARCHAR(255)
)
"""
cursor.execute(create_table_query)

@app.route('/')
def index():
    # Получение IP-адреса пользователя
    ip_address = request.headers.get('X-Forwarded-For')

    # Запись в базу данных
    now = datetime.now()
    current_time = now.strftime("%Y-%m-%d %H:%M:%S")
    query = f"""INSERT INTO {db_table} (request_date, request_ip) VALUES (%s, %s)"""
    values = (current_time, ip_address)
    cursor.execute(query, values)
    db.commit()

    return f'TIME: {current_time}, IP: {ip_address}'


if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0')
```
</details>

## Задача 2 (*)
1. Создайте в yandex cloud container registry с именем "test" с помощью "yc tool" . [Инструкция](https://cloud.yandex.ru/ru/docs/container-registry/quickstart/?from=int-console-help)
2. Настройте аутентификацию вашего локального docker в yandex container registry.
3. Соберите и залейте в него образ с python приложением из задания №1.
4. Просканируйте образ на уязвимости.
5. В качестве ответа приложите отчет сканирования.
```
yc container registry create --name my-first-registry
yc container registry configure-docker
docker tag ex-python cr.yandex/crp.../ex-py:1.0.0
docker push cr.yandex/crp.../ex-py:1.0.0
```
[Отчет](https://github.com/joos-net/virt-dip/blob/main/vulnerabilities.csv)

## Задача 3
1. Создайте файл ```compose.yaml```. Опишите в нем следующие сервисы: 

- ```web```. Образ приложения должен ИЛИ собираться при запуске compose из файла ```Dockerfile.python``` ИЛИ скачиваться из yandex cloud container registry(из задание №2 со *). Контейнер должен работать в bridge-сети с названием ```backend``` и иметь фиксированный ipv4-адрес ```172.20.0.5```. Сервис должен всегда перезапускаться в случае ошибок.
Передайте необходимые ENV-переменные для подключения к Mysql базе данных по сетевому имени сервиса ```web``` 

- ```db```. image=mysql:8. Контейнер должен работать в bridge-сети с названием ```backend``` и иметь фиксированный ipv4-адрес ```172.20.0.10```. Явно перезапуск сервиса в случае ошибок. Передайте необходимые ENV-переменные для создания: пароля root пользователя, создания базы данных, пользователя и пароля для web-приложения.Обязательно используйте .env file для назначения секретных ENV-переменных!

2. Запустите проект локально с помощью docker compose , добейтесь его стабильной работы.Протестируйте приложение с помощью команд ```curl -L http://127.0.0.1:8080``` и ```curl -L http://127.0.0.1:8090```.

3. Подключитесь к БД mysql с помощью команды ```docker exec <имя_контейнера> mysql -uroot -p<пароль root-пользователя>``` . Введите последовательно команды (не забываем в конце символ ; ): ```show databases; use <имя вашей базы данных(по-умолчанию example)>; show tables; SELECT * from requests LIMIT 10;```.
4. Остановите проект. В качестве ответа приложите скриншот sql-запроса.

![3-ip](https://github.com/joos-net/virt-dip/blob/main/3-ip.png)

## Задача 4
1. Запустите в Yandex Cloud ВМ (вам хватит 2 Гб Ram).
2. Подключитесь к Вм по ssh и установите docker.
```
https://docs.docker.com/engine/install/ubuntu/
```
3. Напишите bash-скрипт, который скачает ваш fork-репозиторий в каталог /opt и запустит проект целиком.
```bash
#!/bin/bash

set -e

sudo rm -rf /opt/shvirtd
sudo git clone https://github.com/joos-net/shvirtd-example-python /opt/shvirtd
cd /opt/shvirtd
sudo bash -c 'cat << EOF > db.env
MYSQL_ROOT_PASSWORD=root
MYSQL_DATABASE=db
EOF'
sudo bash -c 'cat << EOF > web.env
DB_HOST=172.20.0.10
DB_USER=root
DB_PASSWORD=root
DB_NAME=db
DB_TABLE=it
EOF'
docker compose up -d
```
4. Зайдите на сайт проверки http подключений, например(или аналогичный): ```https://check-host.net/check-http``` и запустите проверку вашего сервиса ```http://<внешний_IP-адрес_вашей_ВМ>:8090```. Таким образом трафик будет направлен в ingress-proxy.
5. (Необязательная часть) Дополнительно настройте remote ssh context к вашему серверу. Отобразите список контекстов и результат удаленного выполнения ```docker ps -a```
```
docker context create test --docker "host=ssh://<username>@<ip-of-server>"
docker context ls
docker context use test
docker ps -a
docker context use default
```

![context](https://github.com/joos-net/virt-dip/blob/main/4-context.png)

6. В качестве ответа повторите  sql-запрос и приложите скриншот с данного сервера, bash-скрипт и ссылку на fork-репозиторий.

## Задача 5 (*)
1. Напишите и задеплойте на вашу облачную ВМ bash скрипт, который произведет резервное копирование БД mysql в директорию "/opt/backup" с помощью запуска в сети "backend" контейнера из образа ```schnitzler/mysqldump``` при помощи ```docker run ...``` команды. Подсказка: "документация образа."
2. Протестируйте ручной запуск
3. Настройте выполнение скрипта раз в 1 минуту через cron, crontab или systemctl timer.
4. Предоставьте скрипт, cron-task и скриншот с несколькими резервными копиями в "/opt/backup"
```
docker run --rm --entrypoint "" -v `pwd`/backup:/backup --network compose_backend --link="compose-db-1" jooos/mysqldump mysqldump --opt -h db -u root -p"root" "--result-file=/backup/dumps.sql" example
```

## Задача 6
Скачайте docker образ ```hashicorp/terraform:latest``` и скопируйте бинарный файл ```/bin/terraform``` на свою локальную машину, используя dive и docker save.
Предоставьте скриншоты  действий .
```
docker pull hashicorp/terraform:latest
docker save -o test.tar hashicorp/terraform:latest
tar -xvf e27646a588fb9ce75afee962a7ed397636930b084890f7e50016939448e3936c/layer.tar bin/terraform
```

## Задача 6.1
Добейтесь аналогичного результата, используя docker cp.  
Предоставьте скриншоты  действий .
```
docker cp $(docker create --name tc hashicorp/terraform:latest):/bin/terraform ./terraform
```

## Задача 6.2 (**)
Предложите способ извлечь файл из контейнера, используя только команду docker build и любой Dockerfile.  
Предоставьте скриншоты  действий .
```
docker build -o - . > out.tar
tar -xvf out.tar bin/terraform
```

## Задача 7 (***)
Запустите ваше python-приложение с помощью runC, не используя docker или containerd.  
Предоставьте скриншоты  действий .
