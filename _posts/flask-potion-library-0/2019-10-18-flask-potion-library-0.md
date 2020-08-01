---
title: Система контроля библиотеки на Flask-Potion. Часть 0.
date: 2019-10-18
modified: 2019-10-18
tags: [python, docker, flask, sql, sitri]
description: Система контроля библиотеки на Flask-Potion
---

#  Введение
В своей работе я уже некоторое время использую [Flask-Potion](https://potion.readthedocs.io) - фреймворк, основными достоинствами которого являются: весьма удобная интеграция с SQLAlchemy моделями, автогенерация crud-эндпоинтов, наличие клиента [potion-client](https://github.com/biosustain/potion-client) (весьма удобного, если пишешь API сервиса, использование которого понадобится в другом сервисе). 

Я заметил, что на русском языке о flask-potion почти ничего нет, но думаю кому-то это данный фреймворк может показаться интересным. 

Вместо простой обзорной статьи на этот фреймворк я решил написать несколько статей о создании системы контроля для библиотеки "Furfur" на основе Flask-Potion.

Данная система должна уметь делать следующее:

 - Хранить информацию о книгах (isbn, название, описание, автор и т.д.)
 - Хранить информацию о пользователях (читатели и библиотекари)
 - Оформлять выдачу книги из библиотеки на определённый срок с возможностью продления

В этой системе мы воспользуемся следующими инструментами:

 - PostgreSQL
 - Flask, Flask-SQLAlchemy, Flask-JWT, Flask-Potion, Flask-Migrate

#  Подготовка

##  Скелет
Чтобы не собирать скелет для проекта самим, воспользуемся cookiecutter-шаблоном [Valefor](https://clck.ru/JVaLP), который включает в себя все вышеперечисленные зависимости и даже больше.

```bash
cookiecutter gh:lemegetonx/valefor
```
Этот шаблон включает в себя два приложения:

 1. app - основное. Содержит в себе функции-обработчики для jwt, mixin классы для potion ресурсов и sqlalchemy моделей, а также пакет с конфигурациями для приложения.
 2. user - на старте шаблона, содержит только модель пользователя.

##  Установка зависимостей

В шаблоне используется poetry для разрешения зависимостей, но с недавних пор pip тоже поддерживает *pyproject.toml*, поэтому тут выбор за вами. Я воспользуюсь poetry.

```bash
poetry install
```

##  Конфигурация

Для упрощённой конфигурации в шаблоне применена библиотека sitri. Нам понадобится немного изменить настройку объекта Sitri.

 1. Изменим *app/config/provider.py*. Заменим *SystemCredentialProvider* на *YamlCredentialProvider*, чтобы данные для авторизации в сторонних системах брались из файла *credential.yaml*, добавлять который в коммиты мы не будем:

```python
from sitri import Sitri  
from sitri.contrib.yaml import YamlConfigProvider, YamlCredentialProvider
  
configuration = Sitri(  
    config_provider=YamlConfigProvider(yaml_path="./config.yaml"),  
    credential_provider=YamlCredentialProvider(yaml_path="./credential.yaml"),  
)
```
P.S. подробнее, что собственно здесь происходит легче прочитать в [документации](https://sitri.readthedocs.io), если коротко, то сейчас мы просто определили откуда будем брать данные для конфигурации и авторизации.

 2. Раз уж мы сделали по сути одинаковые провайдеры, то лучше в *database.py* заменить нижние подчеркивания в ключах в вызове *get_credential* на точки.

```python
DB_NAME = configuration.get_credential("db.name", path_mode=True)  
DB_HOST = configuration.get_credential("db.host", path_mode=True)  
DB_PASSWORD = configuration.get_credential("db.user.password", path_mode=True)  
DB_PORT = configuration.get_credential("db.port", path_mode=True)  
DB_USER = configuration.get_credential("db.user.name", path_mode=True)
```
Итак, файл *config.yaml* уже был в шаблоне, а вот *credential.yaml*  должны написать сами. В реальной жизни подобные файлы обязательно добавляются в .gitignore, но я добавлю шаблон *credential.yaml* в репозиторий, чтобы его структура была понятна любому, кто зайдёт в проект.

Базовый *credential.yaml*:
```yaml
db:  
  name: furfur_db  
  host: localhost  
  port: 5432  
  user:  
    password: passwd  
    name: admin
```

##  База данных

Следующий этап нашей подготовки - это развертывание СУБД, в данном случае PostgreSQL. Я для удобства сделаю *stack.yaml* файл, где опишу запуск контейнера postgres с нужными для нас данными.

```yaml
version: '3.1'

services:

  db:
    image: postgres
    restart: always
    environment:
      POSTGRES_PASSWORD: passwd
      POSTGRES_USER: admin
      POSTGRES_DB: furfur_db
    ports:
      - 5432:5432
```

Как говорилось ранее, в состав шаблона valefor входит базовая модель  User, нужная для работы JWT (хендлеров), поэтому заключительный этап подготовки БД - это миграция (создание таблицы пользователей).

Находясь в корне проекта исполняем следующие команды:
```bash
export FLASK_APP=furfur.app

flask db init
flask db migrate
flask db upgrade
```

Всё, с подготовкой БД, как и в целом основы для нашей системы, мы закончили.

#  Что дальше?
В следующей части мы поговорим о том, как организовать простую систему ролей и аутентификацию по JWT.

[Репозиторий проекта](https://github.com/Egnod/furfur)
\
[Всё, что изложено в данной части](https://github.com/Egnod/furfur/releases/tag/0.0.2)
