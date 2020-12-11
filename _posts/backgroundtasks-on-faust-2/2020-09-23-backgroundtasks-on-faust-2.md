---
title: Фоновые задачи на Faust, Часть II. Агенты и Команды
date: 2020-09-23
modified: 2020-09-23
tags: [python, faust, kafka, kafka-streams]
description: Фоновые задачи на Faust, Часть II. Агенты и Команды
external_link: https://habr.com/ru/post/520274/
external_link_name: HABR Originals
---

<img src="https://habrastorage.org/webt/is/mh/8k/ismh8kk9mjw7weuaopnqw07kp4m.jpeg">

### Оглавление

1.  [Часть I: Введение](https://egnod.dev/backgroundtasks-on-faust-1)

2.  Часть II: Агенты и Команды

### Что мы тут делаем?

Итак-итак, вторая часть. Как и писалось ранее, в ней мы сделаем следующее:

1.  Напишем небольшой клиентик для alphavantage на aiohttp с запросами на нужные нам эндпоинты.

2.  Сделаем агента, который будет собирать данные о ценных бумагах и мета информацию по ним.

Но, это то, что мы сделаем для самого проекта, а в плане исследования faust мы узнаем, как писать агентов, обрабатывающих стрим событий из kafka, а так же как написать команды (обёртка на click), в нашем случаи - для ручного пуша сообщения в топик, за которым следит агент.

### Подготовка

#### Клиент AlphaVantage

Для начала, напишем небольшой aiohttp клиентик для запросов на alphavantage.

[alphavantage.py](https://github.com/Egnod/horton/blob/3b50581b3639ce7d6fe0d53edbf4ac0976f3fd67/horton/alphavantage.py)

Собственно по нему всё ясно:

1.  API AlphaVantage достаточно просто и красиво спроектирована, поэтому все запросы я решил проводить через метод `construct_query` где в свою очередь идёт http вызов.

2.  Все поля я привожу к `snake_case` для удобства.

3.  Ну и декорация logger.catch для красивого и информативного вывода трейсбека.

P.S. Незабываем локально добавить токен alphavantage в config.yml, либо экспортировать переменную среды `HORTON_SERVICE_APIKEY`. Получаем токен [тут](https://www.alphavantage.co/support/#api-key).

#### CRUD-класс

У нас будет коллекция securities для хранения мета информации о ценных бумагах.

[database/security.py](https://github.com/Egnod/horton/blob/363d601030da16733940bf2532a2d9e493e05401/horton/database/cruds/security.py)

Тут по-моему ничего пояснять не нужно, а базовый класс сам по себе достаточно прост.

#### get_app()

Добавим функцию создания объекта приложения в [app.py](https://github.com/Egnod/horton/blob/3126e66dafa2c20edf4f657119fc2a7f4833cc6c/horton/app.py)

Пока у нас будет самое простое создание приложения, чуть позже мы его расширим, однако, чтобы не заставлять вас ждать, вот [референсы](https://faust.readthedocs.io/en/latest/reference/faust.app.html#faust-app) на App-класс. На класс settings тоже советую взглянуть, так как именно он отвечает за большую часть настроек.

### Основная часть

#### Агент сбора и сохранения списка ценных бумаг
```python
app = get_app()

collect_securities_topic = app.topic("collect_securities", internal=True)

@app.agent(collect_securities_topic)
async def collect_securities(stream: StreamT[None]) -> AsyncIterable[bool]:
    pass
```

Так, сначала получаем объект faust-приложения - это достаточно просто. Далее, мы явно объявляем топик для нашего агента... Тут стоит упомянуть, что это такое, что за параметр internal и как это можно устроить по-другому.

1.  Топики в kafka, если мы хотим узнать точное определение, то лучше прочитать [офф. доку](https://kafka.apache.org/documentation/#intro_concepts_and_terms), либо можно прочитать [конспект](https://habr.com/ru/post/354486/) на хабре на русском, где так же всё достаточно точно отражено :)

2.  [Параметр internal](https://faust.readthedocs.io/en/latest/reference/faust.html?highlight=internal#faust.TopicT.internal), достаточно хорошо описанный в доке faust, позволяет нам настраивать топик прямо в коде, естественно, имеются ввиду параметры, предусмотренные разработчиками faust, например: retention, retention policy (по-умолчанию delete, но можно установить и [compact](https://faust.readthedocs.io/en/latest/reference/faust.html?highlight=internal#faust.TopicT.compacting)), кол-во партиций на топик ([partitions](https://faust.readthedocs.io/en/latest/reference/faust.topics.html?highlight=partitions#faust.topics.Topic.partitions), чтобы сделать, например, меньшее чем [глобальное значение](https://faust.readthedocs.io/en/latest/reference/faust.app.html?highlight=partitions#faust.app.App.Settings.topic_partitions) приложения faust).

3.  Вообще, агент может создавать сам управляемый топик с глобальными значениями, однако, я люблю объявлять всё явно. К тому же, некоторые параметры (например, кол-во партиций или retention policy) топика в объявлении агента настроить нельзя.

    Вот как это могло было выглядеть без ручного определения топика:

```python
app = get_app()

@app.agent()
async def collect_securities(stream: StreamT[None]) -> AsyncIterable[bool]:
    pass

```

Ну а теперь, опишем, что будет делать наш агент :)

```python
app = get_app()

collect_securities_topic = app.topic("collect_securities", internal=True)

@app.agent(collect_securities_topic)
async def collect_securities(stream: StreamT[None]) -> AsyncIterable[bool]:
    async with aiohttp.ClientSession() as session:
        async for _ in stream:
            logger.info("Start collect securities")

            client = AlphaVantageClient(session, API_KEY)

            securities = await client.get_securities()

            for security in securities:
                await SecurityCRUD.update_one(
                    {"symbol": security["symbol"], "exchange": security["exchange"]}, security, upsert=True
                )

            yield True
```

Итак, в начале агента мы открываем aiohttp сессию для запросов через наш клиент. Таким образом, при запуске воркера, когда будет запущен наш агент, сразу же будет открыта сессия - одна, на всё время работы воркера (или несколько, если изменить параметр [concurrency](https://faust.readthedocs.io/en/latest/userguide/settings.html?highlight=concurrency#advanced-agent-settings) у агента с дефолтной единички).

Далее, мы идём по стриму (сообщение мы помещаем в `_`, так как нам, в данном агенте, безразлично содержание) сообщений из нашего топика, если они есть при текущем сдвиге (offset), иначе, наш цикл будет ожидать их поступления. Ну а внутри нашего цикла, мы логируем поступление сообщения, получаем список активных (get_securities возвращает по-умолчания только active, см. код клиента) ценных бумаг и сохраняем его в базу, проверяя при этом, есть ли бумага с таким тикером и биржей в БД, если есть, то она (бумага) просто обновится.

Запустим наше творение!

```bash
docker-compose up -d
# ... Запуск контейнеров ...
faust -A horton.agents worker --without-web -l info
```

P.S. Возможности [веб-компонента](https://faust.readthedocs.io/en/latest/userguide/tasks.html#web-views) faust я рассматривать в статьях не буду, поэтому выставляем соответствующий флаг.

В нашей команде запуска мы указали faust'у, где искать объект приложения и что делать с ним (запустить воркер) с уровнем вывода логов info. Получаем следующий вывод:

```text
┌ƒaµS† v1.10.4┬───────────────────────────────────────────────────┐
│ id          │ horton                                            │
│ transport   │ [URL('kafka://localhost:9092')]                   │
│ store       │ memory:                                           │
│ log         │ -stderr- (info)                                   │
│ pid         │ 1271262                                           │
│ hostname    │ host-name                                         │
│ platform    │ CPython 3.8.2 (Linux x86_64)                      │
│ drivers     │                                                   │
│   transport │ aiokafka=1.1.6                                    │
│   web       │ aiohttp=3.6.2                                     │
│ datadir     │ /path/to/project/horton-data                      │
│ appdir      │ /path/to/project/horton-data/v1                   │
└─────────────┴───────────────────────────────────────────────────┘
    ... логи, логи, логи ...

    ┌Topic Partition Set─────────┬────────────┐
    │ topic                      │ partitions │
    ├────────────────────────────┼────────────┤
    │ collect_securities         │ {0-7}      │
    │ horton-__assignor-__leader │ {0}        │
    └────────────────────────────┴────────────┘ 
```

<s>Оно живое!!!</s>

Посмотрим на partition set. Как мы видим, был создан топик с именем, которое мы обозначили в коде, кол-во партиций дефолтное (8, взятое из [topic_partitions](https://faust.readthedocs.io/en/latest/userguide/settings.html?highlight=topic_partitions#std:setting-topic_partitions) - параметра объекта приложения), так как у нашего топика мы индивидуальное значение (через partitions) не указывали. Запущенному агенту в воркере отведены все 8 партициций, так как он единственный, но об этом будет подробнее в части про кластеринг.

Что же, теперь можем зайти в другое окно терминала и отправить пустое сообщение в наш топик:

```bash
faust -A horton.agents send @collect_securities
-> {"topic": "collect_securities", "partition": 6, "topic_partition": ["collect_securities", 6], "offset": 0, "timestamp": ..., "timestamp_type": 0}

```

P.S. с помощью `@` мы показываем, что посылаем сообщение в топик с именем "collect_securities".

В данном случае, сообщение ушло в 6 партицию - это можно проверить, зайдя в kafdrop на `localhost:9000`

Перейдя в окно терминала с нашим воркером, мы увидим радостное сообщение, посланное с помощью loguru:

```bash
2020-09-23 00:26:37.304 | INFO     | horton.agents:collect_securities:40 - Start collect securities
```

Так же, можем заглянуть в mongo (с помощью Robo3T или Studio3T) и увидеть, что ценные бумаги в базе:

_Я не миллиардер, а потому, довольствуемся первым вариантом просмотра._

<img src="https://habrastorage.org/getpro/habr/upload_files/48b/5ad/c11/48b5adc118587bb7a99ac3c299ef8bef">
<img src="https://habrastorage.org/getpro/habr/upload_files/0b7/efa/5ec/0b7efa5ec51937472240628289c28908">
<br/>

Счастье и радость - первый агент готов :)

#### Агент готов, да здравствует новый агент!

Да, господа, нами пройдена только 1/3 пути, уготованного этой статьёй, но не унывайте, так как сейчас будет уже легче.

Итак, теперь нам нужен агент, который собирает мета информацию и складывает её в документ коллекции:

```python
collect_security_overview_topic = app.topic("collect_security_overview", internal=True)

@app.agent(collect_security_overview_topic)
async def collect_security_overview(
    stream: StreamT[?],
) -> AsyncIterable[bool]:
    async with aiohttp.ClientSession() as session:
        async for event in stream:
            ...
```

Так как этот агент будет обрабатывать информацию о конкретной security, нам нужно в сообщении указать тикер (symbol) этой бумаги. Для этого в faust существуют [Records](https://faust.readthedocs.io/en/latest/userguide/models.html?highlight=Records#model-types) - классы, декларирующие схему сообщения в топике агента.

В таком случае перейдём в [records.py](https://github.com/Egnod/horton/blob/3126e66dafa2c20edf4f657119fc2a7f4833cc6c/horton/records.py) и опишем, как должно выглядеть сообщение у этого топика:

```python
import faust

class CollectSecurityOverview(faust.Record):
    symbol: str
    exchange: str
```

Как вы уже могли догадаться, faust для описания схемы сообщения использует аннотацию типов в python, поэтому и минимальная версия, поддерживаемая библиотекой - [3.6](https://github.com/robinhood/faust/blob/cf0ea8ce38d7808bd81378f21e6fc6072123d82e/setup.py#L209).

Вернёмся к агенту, установим типы и допишем его:

```python
collect_security_overview_topic = app.topic(
    "collect_security_overview", internal=True, value_type=CollectSecurityOverview
)


@app.agent(collect_security_overview_topic)
async def collect_security_overview(
    stream: StreamT[CollectSecurityOverview],
) -> AsyncIterable[bool]:
    async with aiohttp.ClientSession() as session:
        async for event in stream:
            logger.info(
                "Start collect security [{symbol}] overview", symbol=event.symbol
            )

            client = AlphaVantageClient(session, API_KEY)

            security_overview = await client.get_security_overview(event.symbol)

            await SecurityCRUD.update_one(
                {"symbol": event.symbol, "exchange": event.exchange}, security_overview
            )

            yield True
```

Как видите, мы передаём в метод инициализации топика новый параметр со схемой - value_type. Далее, всё по той же самой схеме, поэтому останавливаться на чём то ещё - смысла не вижу.

Ну что же, последний штрих - добавим в collect_securitites вызов агента сбора мета информации:

```python
....
for security in securities:
    await SecurityCRUD.update_one(
        {"symbol": security["symbol"], "exchange": security["exchange"]},
        security,
        upsert=True,
    )

    await collect_security_overview.cast(
        CollectSecurityOverview(
            symbol=security["symbol"], exchange=security["exchange"]
        )
    )

....
```

Используем ранее объявлению схему для сообщения. В данном случае, я использовал метод .cast, так как нам не нужно ожидать результат от агента, но стоит упомянуть, что [способов](https://faust.readthedocs.io/en/latest/userguide/agents.html?highlight=.cast#cast-or-ask) послать сообщение в топик:

1.  cast - не блокирует, так как не ожидает результата. Нельзя послать результат в другой топик сообщением.

2.  send - не блокирует, так как не ожидает результата. Можно указать агента в топик которого уйдёт результат.

3.  ask - ожидает результата. Можно указать агента в топик которого уйдёт результат.

Итак, на этом с агентами на сегодня всё!

#### Команда мечты

Последнее, что я обещал написать в этой части - команды. Как уже говорилось ранее, команды в faust - это обёртка над click. Фактически faust просто присоединяет нашу кастомную команду к своему интерфейсу при указании ключа -A

После объявленных агентов в [agents.py](https://github.com/Egnod/horton/blob/b27b5160d3fd47cf53899b74b6ed222a0fdbfe45/horton/agents.py) добавим функцию с декоратором _app.command_, вызывающую метод _cast_ у _collect_securitites_:

```python
@app.command()
async def start_collect_securities():
    """Collect securities and overview."""

    await collect_securities.cast()

```

Таким образом, если мы вызовем список команд, в нём будет и наша новая команда:

```bash
> faust -A horton.agents --help

    ....
    Commands:
      agents                    List agents.
      clean-versions            Delete old version directories.
      completion                Output shell completion to be evaluated by the...
      livecheck                 Manage LiveCheck instances.
      model                     Show model detail.
      models                    List all available models as a tabulated list.
      reset                     Delete local table state.
      send                      Send message to agent/topic.
      start-collect-securities  Collect securities and overview.
      tables                    List available tables.
      worker                    Start worker instance for given app.
```

Ею мы можем воспользоваться, как любой другой, поэтому перезапустим faust воркер и начнём полноценный сбор ценных бумаг:
```bash
> faust -A horton.agents start-collect-securities
```

### Что будет дальше?

В следующей части мы, на примере оставшихся агентов, рассмотрим, механизм sink для поиска экстремум в ценах закрытия торгов за год и cron-запуск агентов.

На сегодня всё! Спасибо за прочтение :)

[Код этой части](https://github.com/Egnod/horton/tree/b27b5160d3fd47cf53899b74b6ed222a0fdbfe45)

<img src="https://habrastorage.org/getpro/habr/upload_files/d0b/290/1bc/d0b2901bcb6af16d5e14310329b596bf">

P.S. Под прошлой частью меня спросили про faust и confluent kafka ([какие есть у confluent фичи](https://stackoverflow.com/a/39709900)). Кажется, что confluent во многом функциональнее, но дело в том, что faust не имеет полноценной поддержки клиента для confluent - это следует из [описания ограничений клиентов в доке](https://faust.readthedocs.io/en/latest/userguide/settings.html?highlight=confluent#available-transports).
