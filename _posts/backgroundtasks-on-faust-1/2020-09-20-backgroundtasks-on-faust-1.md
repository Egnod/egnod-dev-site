---
title: Фоновые задачи на Faust, Часть I: Введение
date: 2020-09-20
modified: 2020-09-20
tags: [python, faust, kafka, kafka-streams]
description: Фоновые задачи на Faust, Часть I: Введение
---

![https://habrastorage.org/webt/wo/6b/ui/wo6buieqgfwzr4y5tczce4js0rc.png](https://habrastorage.org/webt/wo/6b/ui/wo6buieqgfwzr4y5tczce4js0rc.png)

1.  Часть I: Введение
2.  [Часть II: Агенты и Команды](https://egnod.dev/backgroundtasks-on-faust-2)

## Как я дошёл до жизни такой?

Не так давно мне пришлось работать над бэкендом высоко нагруженного проекта, в котором нужно было организовать регулярное выполнение большого количества фоновых задач со сложными вычислениями и запросами на сторонние сервисы. Проект асинхронный и до того, как я пришёл, в нём был простой механизм крон-запуска задач: цикл с проверкой текущего времени и запуск групп корутин через gather - такой подход оказался приемлем до момента, пока таких корутин были десятки и сотни, однако, когда их количество перевалило через две тысячи, пришлось думать об организации нормальной очереди задач с брокером, несколькими воркерами и прочим.
<cut>

Сначала я решил опробовать Celery, которым пользовался ранее. В связи с асинхронностью проекта, я погрузился в вопрос и увидел [статью](https://habr.com/ru/post/502380/), а так же [проект](https://github.com/kai3341/celery-pool-asyncio), созданный автором статьи.

Скажу так, проект очень интересный и вполне успешно работает в других приложениях нашей команды, да и сам автор говорит о том, что смог выкатить в прод, заюзав асинхронный пул. Но, к сожалению, мне это не очень подошло, так как обнаружилась [проблема](https://github.com/kai3341/celery-pool-asyncio/issues/22) с групповым запуском задач (см. [group](https://docs.celeryproject.org/en/stable/reference/celery.html#celery.group)). На момент написания статьи [issue](https://github.com/kai3341/celery-pool-asyncio/issues/22) уже закрыта, однако, работа велась на протяжении месяца. В любом случае, автору удачи и всех благ, так как рабочие штуки на либе уже есть... в общем, дело во мне и для меня оказался инструмент сыроват. Вдобавок, в некоторых задачах было по 2-3 http-запроса к разным сервисам, таким образом даже при оптимизации задач мы создаём 4 тысячи tcp соединений, примерно каждые 2 часа - не очень... Хотелось бы создавать сессию на один тип задач при запуске воркеров. Чуть подробнее о большом кол-ве запросов через aiohttp [тут](https://pawelmhm.github.io/asyncio/python/aiohttp/2016/04/22/asyncio-aiohttp.html).
<cut/>
В связи с этим, я начал искать ***альтернативы*** и нашёл! Создателями celery, а конкретно, как я понял [Ask Solem](https://github.com/ask), была создана [Faust](https://github.com/robinhood/faust), изначально для проекта [robinhood](http://robinhood.com). Faust написана под впечатлением от Kafka Streams и работает с Kafka в качестве брокера, также для хранения результатов от работы агентов используется rocksdb, а самое главное - это то, что библиотека асинхронна.
<cut/>
Также, можете глянуть [краткое сравнение](https://faust.readthedocs.io/en/latest/playbooks/vscelery.html) celery и faust от создателей последней: их различия, различия брокеров, реализацию элементарной задачи. Всё весьма просто, однако, в faust привлекает внимание приятная особенность - типизированные данные для передачи в топик. 
<cut/>
## Что будем делать?

Итак, в небольшой серии статей я покажу, как собирать данные в фоновых задачах с помощью Faust. Источником для нашего пример-проекта будет, как следует из названия, [alphavantage.co](http://alphavantage.co/). Я продемонстрирую, как писать агентов (sink, топики, партиции), как делать регулярное (cron) выполнение, удобнейшие cli-комманды faust (обёртка над click), простой кластеринг, а в конце прикрутим datadog (работающий из коробки) и попытаемся, что-нибудь увидеть. Для хранения собранных данных будем использовать mongodb и motor для подключения.

P.S. Судя по уверенности, с которой написан пункт про мониторинг, думаю, что читатель в конце последней статьи всё-таки будет выглядеть, как-то так:

![https://habrastorage.org/webt/e5/v1/pl/e5v1plkcyvxyoawde4motgq7vpm.png](https://habrastorage.org/webt/e5/v1/pl/e5v1plkcyvxyoawde4motgq7vpm.png)
<cut/>
## Требования к проекту

В связи с тем, что я уже успел наобещать, составим небольшой списочек того, что должен уметь сервис:

1. Выгружать ценные бумаги и overview по ним (в т.ч. прибыли и убытки, баланс, cash flow - за последний год) - регулярно
2. Выгружать исторические данные (для каждого торгового года находить экстремумы цены закрытия торгов) - регулярно
3. Выгружать последние торговые данные - регулярно
4. Выгружать настроенный список индикаторов для каждой ценной бумаги - регулярно

Как полагается, выбираем имя проекту с потолка: *horton*
<cut/>
## Готовим инфраструктуру

Заголовок конечно сильный, однако, всё что нужно сделать - это написать небольшой конфиг для docker-compose с kafka (и zookeeper - в одном контейнере), kafdrop (если нам захочется посмотреть сообщения в топиках), mongodb. Получаем *[docker-compose.yml*](https://github.com/Egnod/horton/blob/562fa5ec14df952cd74760acf76e141707d2ef58/docker-compose.yml) следующего вида:

```yaml
version: '3'

services:
  db:
    container_name: horton-mongodb-local
    image: mongo:4.2-bionic
    command: mongod --port 20017
    restart: always
    ports:
      - 20017:20017
    environment:
      - MONGO_INITDB_DATABASE=horton
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=admin_password

  kafka-service:
    container_name: horton-kafka-local
    image: obsidiandynamics/kafka
    restart: always
    ports:
      - "2181:2181"
      - "9092:9092"
    environment:
      KAFKA_LISTENERS: "INTERNAL://:29092,EXTERNAL://:9092"
      KAFKA_ADVERTISED_LISTENERS: "INTERNAL://kafka-service:29092,EXTERNAL://localhost:9092"
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: "INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT"
      KAFKA_INTER_BROKER_LISTENER_NAME: "INTERNAL"
      KAFKA_ZOOKEEPER_SESSION_TIMEOUT: "6000"
      KAFKA_RESTART_ATTEMPTS: "10"
      KAFKA_RESTART_DELAY: "5"
      ZOOKEEPER_AUTOPURGE_PURGE_INTERVAL: "0"

  kafdrop:
    container_name: horton-kafdrop-local
    image: 'obsidiandynamics/kafdrop:latest'
    restart: always
    ports:
      - '9000:9000'
    environment:
      KAFKA_BROKERCONNECT: kafka-service:29092
    depends_on:
      - kafka-service
```
<cut/>
Тут вообще ничего сложного. Для kafka объявили два listener'а: одного (internal) для использования внутри композной сети, а второго (external) для запросов из вне, поэтому пробросили его наружу. 2181 - порт zookeeper'а. По остальному, я думаю, ясно.
<cut/>
## Готовим скелет проекта

В базовом варианте структура нашего проекта должна выглядеть так:

```yaml
horton
├── docker-compose.yml
└── horton
    ├── agents.py *
    ├── alphavantage.py *
    ├── app.py *
    ├── config.py
    ├── database
    │   ├── connect.py
    │   ├── cruds
    │   │   ├── base.py
    │   │   ├── __init__.py
    │   │   └── security.py *
    │   └── __init__.py
    ├── __init__.py
    ├── records.py *
    └── tasks.py *
```

**Всё что я <u>отметил</u>, мы пока не трогаем, а просто создаём пустые файлы.** 

Создали структуру. Теперь добавим необходимые зависимости, напишем конфиг и подключение к mongodb. Полный текст файлов приводить в статье не буду, чтобы не затягивать, а сделаю ссылки на нужные версии.

Начнём с зависимостей и мета о проекте - *[pyproject.toml](https://github.com/Egnod/horton/blob/7e1d2b41f7d091b3fc6d4627a9be7ff6f76b0dd8/pyproject.toml)*

Далее, запускаем установку зависимостей и создание virtualenv (либо, можете сами создать папку venv и активировать окружение):

```bash
pip3 install poetry (если ещё не установлено)
poetry install
```

Теперь создадим [config.yml](https://github.com/Egnod/horton/blob/7e1d2b41f7d091b3fc6d4627a9be7ff6f76b0dd8/config.yml) - креды и куда стучаться. Сразу туда можно разместить и данные для alphavantage. Ну и переходим к [config.py](https://github.com/Egnod/horton/blob/7e1d2b41f7d091b3fc6d4627a9be7ff6f76b0dd8/horton/config.py) - извлекаем данные для приложения из нашего конфига. Да, каюсь, заюзал свою либу - [sitri](https://github.com/LemegetonX/sitri).

По подключению с монго - совсем всё просто. Объявили [класс клиента](https://github.com/Egnod/horton/blob/7e1d2b41f7d091b3fc6d4627a9be7ff6f76b0dd8/horton/database/connect.py) для подключения и [базовый класс](https://github.com/Egnod/horton/blob/7e1d2b41f7d091b3fc6d4627a9be7ff6f76b0dd8/horton/database/cruds/base.py) для крудов, чтобы проще было делать запросы по коллекциям.
<cut/>
## Что будет дальше?

Статья получилась не очень большая, так как здесь я говорю только о мотивации и подготовке, поэтому не обессудьте - обещаю, что в следующей части будет экшн ~~и графика.~~

Итак, а в этой самой следующей части мы: 

1. Напишем небольшой клиентик для alphavantage на aiohttp с запросами на нужные нам эндпоинты.
2. Сделаем агента, который будет собирать данные о ценных бумагах и мета информацию по ним.

[Код проекта](https://github.com/Egnod/horton)

[Код этой части](https://github.com/Egnod/horton/tree/7e1d2b41f7d091b3fc6d4627a9be7ff6f76b0dd8)
