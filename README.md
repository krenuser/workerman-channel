# Channel
Компонент многопроцессорной связи на основе подписки, используемый для межпроцессного взаимодействия рабочего процесса или взаимодействия кластера серверов, аналогично механизму публикации подписки Redis. Разработан на основе workerman.

Канал обеспечивает две формы связи, а именно механизм событий публикации-подписки и механизм очереди сообщений.

Их основные отличия:

Механизм событий заключается в том, что после отправки сообщения все клиенты, подписавшиеся на событие, могут получить это сообщение.
Механизм очереди сообщений заключается в том, что после отправки сообщения только один из клиентов, подписавшихся на сообщение, получит сообщение.Если клиент занят, сообщение будет поставлено в очередь до тех пор, пока клиент не будет бездействовать, а затем получить сообщение.
Следует отметить, что канал предоставляет только способ связи и не предоставляет таких функций, как подтверждение сообщения, повторная попытка, задержка, сохранение и т. д. Пожалуйста, используйте его разумно в соответствии с реальной ситуацией.

# Руководство
[Руоводство по Channel](http://doc.workerman.net/components/channel.html)

# Сервер
```php
use Workerman\Worker;

//Соединение через Tcp
$channel_server = new Channel\Server('0.0.0.0', 2206);

//Соединение через Unix Domain Socket 
//$channel_server = new Channel\Server('unix:///tmp/workerman-channel.sock');

if(!defined('GLOBAL_START'))
{
    Worker::runAll();
}
```

# Клиент
```php
use Workerman\Worker;

$worker = new Worker();
$worker->onWorkerStart = function()
{
    // Подключение клиента Channel к серверу Channel
    Channel\Client::connect('<IP сервера Channel>', 2206);

    // Связь с использованием Unix Domain Socket
    //Channel\Client::connect('unix:///tmp/workerman-channel.sock');

    // Название события для подписки (имя может быть любой комбинацией цифр и строк)
    $event_name = 'event_xxxx';
    // Подписка на пользовательское событие и регистрация обратного вызова, который будет запускаться автоматически после получения события
    Channel\Client::on($event_name, function($event_data){
        var_dump($event_data);
    });
};
$worker->onMessage = function($connection, $data)
{
    // Название события для публикации
    $event_name = 'event_xxxx';
    // Данные события (формат данных может быть числами, строками, массивами), которые будут переданы в клиентскую callback-функцию в качестве параметра
    $event_data = array('some data.', 'some data..');
    // Опубликуйте пользовательское событие, клиент, который подпишется на это событие, получит данные события и вызовет соответствующий обратный вызов события на клиенте.
    Channel\Client::publish($event_name, $event_data);
};

if(!defined('GLOBAL_START'))
{
    Worker::runAll();
}
````

## Пример очереди сообщений
```php
use Workerman\Worker;
use Workerman\Timer;

$worker = new Worker();
$worker->name = 'Producer';
$worker->onWorkerStart = function()
{
    Client::connect();

    $count = 0;
    Timer::add(1, function() {
        Client::enqueue('queue', 'Hello World '.time());
    });
};

$mq = new Worker();
$mq->name = 'Consumer';
$mq->count = 4;
$mq->onWorkerStart = function($worker) {
    Client::connect();

    // Подписка на очередь сообщений
    Client::watch('queue', function($data) use ($worker) {
        echo "Worker {$worker->id} get queue: $data\n";
    });

    // Отписаться от этого сообщения через 10 секунд
    Timer::add(10, function() {
        Client::unwatch('queue');
    }, [], false);
};

Worker::runAll();
```
