# Библиотека для реализации паттерна pub/sub для фреймворка Phalcon

Библиотека реализует событийную архитектуру приложений (Event-Driven Architecture). Работает с фреймворком Phalcon 3.x, но при желании можно легко адаптировать под другие фреймворки.  

Рабочий пример можно посмотреть вот здесь: https://github.com/chocofamilyme/pubsub/tree/master/examples

### Возможности
- Транзакционное сохранение моделей ORM и публикация события
- Публикация событий без транзакции
- Подписка на события 
- Повторная отправка события в ту же очередь при необходимости
- Сохранение в общую очередь всех не обработанных и истекших сообщений. Из этой очереди потом можно сохранить куда-то в БД и обработать индивидуально

### Требования
- Phalcon 3.x+
- PHP 7.0+

### Установка
```
composer require chocofamilyme/pubsub
```

### Настройка

На данный момент библиотека работает только с RabbitMQ, при желаении можно добавить другие. 

#### Настройка конфигов
```php
return [
    'eventsource' => [
        'default' => 'rabbitmq',
        
        'drivers' => [
            'rabbitmq' => [
                'adapter'    => 'RabbitMQ',
                'hosts' => [
                    [
                        'host'     => 'host',
                        'port'     => 5672,
                        'user'     => 'user',
                        'password' => 'password',
                        'vhost'    => '/',
                    ],
                ],
    
                // Не объязательные параметры
                'heartbeat'          => 60,
                'read_write_timeout' => 60,
                'connection_timeout' => 60,
                'wait_timeout'       => 0,
            ],
        ],
    ],
];
```

Полный список смотрите - https://github.com/php-amqplib/php-amqplib

#### Добавляем брокер в DI контейнер

```php
$di = \Phalcon\Di::getDefault();
$config      = $di->get('config')->eventsource;
$config      = $config->drivers[$config->default];

$serviceName = $di->get('config')->domain;
$cache       = $di->get('cache');

$di->setShared('eventsource',
    function () use ($config, $serviceName, $cache) {
        $adapter = $config->adapter;
        $config  = array_merge($config->toArray(), ['app_id' => $serviceName]);
        $class   = 'Chocofamily\PubSub\Provider\\'.$adapter;

        $repeater = new Repeater($cache);
        return $class::getInstance($config, $repeater);
    }
);
```

Здесь `$cache` объект реализующий интерефейс `Phalcon\Cache\BackendInterface`. Кэш используется для подсчета количества повторной обработки определенного сообщения.

#### Таблица параметров с настройками

| Ключ                      | Значение                  | Описание  |
| ------------------------- |:------------------------- | :---------|
| connection                | По умолчанию PhpAmqpLib\Connection\AMQPLazyConnection::class | [php-amqplib](https://github.com/php-amqplib/php-amqplib/tree/master/PhpAmqpLib/Connection) |
| connection_timeout        | По умолчанию 3.0 (сек)    | Максимальное время на соединение с сервером Rabbitmq  |
| read_write_timeout        | По умолчанию 3.0 (сек)    | Максимальное время на получение  |
| heartbeat                 | По умолчанию 60 (сек)     | [RabbitMQ Doc](https://www.rabbitmq.com/heartbeats.html) |
| keepalive                 | По умолчанию false        | [RabbitMQ Doc](https://www.rabbitmq.com/heartbeats.html#tcp-keepalives) |
| context                   | По умолчанию null         | [RabbitMQ Doc](https://www.rabbitmq.com) |
| prefetch_count            | По умолчанию 1            | [RabbitMQ Doc](https://www.rabbitmq.com/consumer-prefetch.html) |
| no_ack                    | По умолчанию false        | [RabbitMQ Doc](https://www.rabbitmq.com/nack.html) |
| durable                   | По умолчанию true         | [RabbitMQ Doc](https://www.rabbitmq.com/queues.html) |
| exclusive                 | По умолчанию false        | [RabbitMQ Doc](https://www.rabbitmq.com/queues.html) |
| queue                     | По умолчанию []           | [RabbitMQ Doc](https://www.rabbitmq.com/resources/specs/amqp0-9-1.pdf) |
| basic_consume_exclusive   | По умолчанию false        | [RabbitMQ Doc](https://www.rabbitmq.com/consumers.html) |
| wait_allowed_methods      | По умолчанию null         | [php-amqplib](https://github.com/php-amqplib/php-amqplib/blob/master/PhpAmqpLib/Channel/AbstractChannel.php) |
| wait_non_blocking         | По умолчанию true         | [php-amqplib](https://github.com/php-amqplib/php-amqplib/blob/master/PhpAmqpLib/Channel/AbstractChannel.php) |
| wait_timeout              | По умолчанию 0            | Максимальное время ожидания до получения первого сообщения |
| exchange_type             | По умолчанию topic        | [RabbitMQ Doc](https://www.rabbitmq.com/tutorials/amqp-concepts.html)) |
| app_id                    | По умолчанию ''           | Индификатор приложения которое создает сообщение |
| sleep                     | По умолчанию 0.1 секунды  | Если wait_non_blocking равно true, то если нету задачи ждать время sleep|

### Использование

#### Публикация
Публиковать сообщения можно используя класс `Chocofamily\PubSub\Publisher`. Минимальный рабочий пример:
````php
$publisher = new Publisher($di->getShared('eventsource'));

$payload = [
	'event_id' => 11995,
	'name' => 'docx',
	'age' => 25
];

$routeKey = 'order.created';

$publisher->send($payload, $routeKey);
````
Для RabbitMQ переменная `$routeKey` должна состоять минимум из двух частей разделенных точкой `.`. Пример `order.created`. Имя Exchange будет содержать первый блок, т.е. `order`. После этого если зайдете в админку rabbitmq должен создаться exchange с именем `order`.
Обновленно: начиная с версии 2.* можно указать `exchange`, которому привяжется маршрут `$routeKey, пример:
````php
$publisher = new Publisher($di->getShared('eventsource'));

$payload = [
	'event_id' => 11995,
	'name' => 'docx',
	'age' => 25
];

$exchange = 'order';
$routeKey = 'order.created';

$publisher->send($payload, $routeKey, $exchange);
````


#### Подписка на событие
Для подписки на события используется класс `Chocofamily\PubSub\Subscriber`. Минимальный рабочий пример:

````php
$params = [
    'queue_name' => 'restapi_orderx',
];

$taskName = 'your_task_name';

$subscriber = new Subscriber($di->getShared('eventsource'), 'order.created.*', $params, $taskName);

$subscriber->subscribe(function ($headers, $body) {
    echo print_r($headers, 1). PHP_EOL;
    echo print_r($body, 1). PHP_EOL;
});
````

Обновленно: начиная с версии 2.* можно указать `exchange` и связать с ним маршрут. Теперь можно указать массив 
маршрутов. Пример:
````php
$params = [
    'queue_name' => 'restapi_orderx',
];

$taskName = 'your_task_name';

$routeKeys = [
    'order.created',
    'order.payed',
];

$exchange = 'order';

$subscriber = new Subscriber($di->getShared('eventsource'), $routeKeys, $params, $taskName, $exchange);

$subscriber->subscribe(function ($headers, $body) {
    echo print_r($headers, 1). PHP_EOL;
    echo print_r($body, 1). PHP_EOL;
});
````

Чтобы обратно отправить сообщение в очередь необходимо в кэлбэк функции кинуть исключение `Chocofamily\PubSub\Exceptions\RetryException`. Сообщение может максимум 5 раз обработаться повторно, после этого он попадает в очередь мертвых сообщений (exchange = DLX). 

В подписчик можно передавать следующие настройки:
````
durable: bool — сохранять на диск данные
queue: array — настройки самой очереди
prefetch_count: int — количество предзагрузки сообщений
no_ack: — требуется ли подтверждение сообщений
app_id — уникальный ID приложения. Можно использовать для идентификации откуда собите пошло изначально
````


#### Публикация используя транзакции БД
Этот способ необходим для атомарности сохранения сущности в БД и публикования события. Следующая картинка хорошо иллюстрирует как это работает:
![alt text](https://image.ibb.co/nvznx9/richardson_microservices_part5_local_transaction_e1449165852332.jpg)

Для этого необходимо создать таблицу events:

````sql
create table events
(
	id serial not null
		constraint events_pkey
			primary key,
	type smallint not null,
	model_id int not null,
	model_type varchar(100) not null,
	exchange   varchar(100) not null,
	routing_key varchar(100) not null,
	payload json not null,
	status smallint not null,
	created_at timestamp default now() not null,
	updated_at timestamp
);
````


Пример использования:
````php
use Chocofamily\PubSub\Services\EventPrepare;

...

$order = new Order([
    'user_id' => 11166541,
    'status'  => 0,
    'total'   => 5852,
]);

$eventSource = $di->get('eventsource');

$event = new EventPrepare($order, new OrderSerialize(['name' => 'docx']), 1);
$event->up($eventSource, 'order.created.-5');
	
````

Модель **Order** должна реализовывать итерфейс ModelInterface.

Обновленно: начиная с версии 2.* можно указать `exchange` и связпть с ним маршрут. Привер:
````php
use Chocofamily\PubSub\Services\EventPrepare;

...

$order = new Order([
    'user_id' => 11166541,
    'status'  => 0,
    'total'   => 5852,
]);

$eventSource = $di->get('eventsource');

$routeKey = 'order.created.-5';
$exchange = 'order';

$event = new EventPrepare($order, new OrderSerialize(['name' => 'docx']), 1);
$event->up($eventSource, $routeKey, $exchange);
	
````

Метод `up` работает так
- db transaction start
- order->save();
- eventModel->save()
- db transaction commit
- event publish

#### Повторная отправка события
Для повторной отправке событие  используется класс `Chocofamily\PubSub\Services\EventRepeater`. Рабочий пример:
````php
use Chocofamily\PubSub\Services\EventRepeater;

...

$dateStart = \DateTime::createFromFormat('Y-m-d', '2018-01-01');

$eventDataProvider = new Chocofamily\PubSub\Provider\Event($di->get('eventsource'), $dateStart);

try {
    $event = new EventRepeater($eventDataProvider);
    $event->retry();
} catch (\Exception $e) {
    $message = sprintf('%d %s in %s:%s', $e->getCode(), $e->getMessage(), $e->getFile(), $e->getLine());
    $di->get('logger')->error($message);
}
````


#### Очистка журнала событий
Для очистки событий  используется класс `Chocofamily\PubSub\Services\EventCleaner` с методом `clean`. 

Рабочий пример:
````php
use Chocofamily\PubSub\Services\EventCleaner;

...

try {
    $event = new EventCleaner($di->get('modelsManager'));
    $event->clean();
} catch (ModelException $e) {
    $message = sprintf('%d %s in %s:%s', $e->getCode(), $e->getMessage(), $e->getFile(), $e->getLine());
    $di->get('logger')->error($message);
}

````
По умолчанию удаляетя событие больше 1 месяца.
Если передать дату как второй параметр в конструкторе, то будет удалятся все событие до указонной даты:
````php
use Chocofamily\PubSub\Services\EventCleaner;

...

$dateTime  = new \DateTime();
$dateTime = $dateTime->modify('-1 day');

try {
    $event = new EventCleaner($di->get('modelsManager'), $dateTime);
    $event->clean();
} catch (ModelException $e) {
    $message = sprintf('%d %s in %s:%s', $e->getCode(), $e->getMessage(), $e->getFile(), $e->getLine());
    $di->get('logger')->error($message);
}

````



@todo
- Написать интерфейс для транзакций и убрать зависимость от фреймворка


