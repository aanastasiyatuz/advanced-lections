# RabbitMQ
> RabbitMQ - это брокер сообщений

> Его установка это отдельная морока конечно, но вот все команды, которые нужны при установке

```bash
sudo apt-get install curl gnupg apt-transport-https -y
```

```bash
curl -1sLf "https://keys.openpgp.org/vks/v1/by-fingerprint/0A9AF2115F4687BD29803A206B73A36E6026DFCA" | sudo gpg --dearmor | sudo tee /usr/share/keyrings/com.rabbitmq.team.gpg > /dev/null
curl -1sLf "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0xf77f1eda57ebb1cc" | sudo gpg --dearmor | sudo tee /usr/share/keyrings/net.launchpad.ppa.rabbitmq.erlang.gpg > /dev/null
curl -1sLf "https://packagecloud.io/rabbitmq/rabbitmq-server/gpgkey" | sudo gpg --dearmor | sudo tee /usr/share/keyrings/io.packagecloud.rabbitmq.gpg > /dev/null
```

```bash
sudo nano /etc/apt/sources.list.d/rabbitmq.list
```

```txt
deb [signed-by=/usr/share/keyrings/net.launchpad.ppa.rabbitmq.erlang.gpg] http://ppa.launchpad.net/rabbitmq/rabbitmq-erlang/ubuntu jammy main
deb-src [signed-by=/usr/share/keyrings/net.launchpad.ppa.rabbitmq.erlang.gpg] http://ppa.launchpad.net/rabbitmq/rabbitmq-erlang/ubuntu jammy main
deb [signed-by=/usr/share/keyrings/io.packagecloud.rabbitmq.gpg] https://packagecloud.io/rabbitmq/rabbitmq-server/ubuntu/ jammy main
deb-src [signed-by=/usr/share/keyrings/io.packagecloud.rabbitmq.gpg] https://packagecloud.io/rabbitmq/rabbitmq-server/ubuntu/ jammy main
```

```
sudo apt-get install -y erlang-base \
    erlang-asn1 erlang-crypto erlang-eldap erlang-ftp erlang-inets \
    erlang-mnesia erlang-os-mon erlang-parsetools erlang-public-key \
    erlang-runtime-tools erlang-snmp erlang-ssl \
    erlang-syntax-tools erlang-tftp erlang-tools erlang-xmerl
```

```
sudo apt-get install rabbitmq-server -y --fix-missing
```

> После этого можем попробовать написать на питоне небольшой код на библеотеке `pika`

# Небольшой тутор по pika и rabbitmq
> Пример есть в папке rabbitmq-tutor

> Установите `pika` в виртуальное окружение

```py
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('127.0.0.1'))
channel = connection.channel()

channel.queue_declare(queue='hello')

channel.basic_publish(exchange='', routing_key='hello', body='Hello World!')

print("[x] Sent 'Hello World!'")

connection.close()
```

> если при запуске вам вышло сообщение, которое в принте, то вы все правильно установили. Давайте разбираться, что у нас происходило в данном коде

> RabbitMQ работает по системе `отправитель -> маршрутизатор -> очередь -> получатель`

> `publisher -> exchange -> queue -> reciever`

> итак, в начале мы с вами установили подключение в сервер rabbitmq на нашем компьютере
```py
connection = pika.BlockingConnection(pika.ConnectionParameters('127.0.0.1'))
channel = connection.channel()
```
> после этого нам нужно создать очередь, в которую будут приходить сообщения и храниться там, пока получатель их не получит
```py
channel.queue_declare(queue='hello')
# мы назвали нашу очередь hello
```
> теперь мы можем отправить сообщение в данную очередь. по идее наше сообщение не должно попадать сразу в очередь. как я описала выше между отправителем и очередью есть еще и маршрутизатор. но пока мы не будем его подключать.
```py
channel.basic_publish(exchange='', routing_key='hello', body='Hello World!')
```
> так как мы не используем маршрутизатор, в `exchange` передаем пустую строку

> `routing_key` - очередь, в которую мы отправляем наше сообщение

> `body` - непосредственно сообщение, которое мы отправляем

> ну и после всего закрываем соеденение с rabbitmq
```py
connection.close()
```

> теперь в rabbitmq в очереди `hello` хранится наше сообщение, которое мы можем достать.

> чтобы проверить действительно ли там оно хранится, можете ввести команду
```bash
sudo rabbitmqctl list_queues
```

> для получения этого сообщения, лучше создайте другой файл и напишите этот код
```py
import pika, sys

def main():
    connection = pika.BlockingConnection(pika.ConnectionParameters('127.0.0.1'))
    channel = connection.channel()

    channel.queue_declare(queue='hello')

    def callback(ch, method, properties, body):
        print(f"[x] Received {body}")

    channel.basic_consume(queue='hello', auto_ack=True, on_message_callback=callback)

    print('[*] Waiting for messages. To exit press CTRL+C')
    channel.start_consuming()

if __name__ == '__main__':
    try:
        main()
    except KeyboardInterrupt:
        sys.exit(0)
```
> Запустите этот код (чтобы остановить ctrl+c), вы увидите то самое сообщение, которое мы отправили недавно в другом коде

> Можете снова проверить `sudo rabbitmqctl list_queues` и вы увидите, что сообщение больше не хранится в очереди hello

> Для большего интереса откройте 2 терминала, в одном из них запустите последний код по приему сообщений, а во втором несколько раз запустите первый код по отправке сообщений

> Во втором коде мы также создали подключение и очередь `hello` (мы ее заново создали, чтобы не было ошибок, если она каким-то образом пропадет из rabbitmq). В любом случае она будет создана лишь в первый раз.

> После мы создали функцию `callback`, которая будет принимать наше сообщение и принтить, какое сообщение лежало в очереди

> эту функцию мы передали нашей очереди `hello` через метод `channel.basic_consume`

> после мы запустили метод `channel.start_consuming`, который держит соеденение с rabbitmq и ждет новые сообщения в очереди

> ну а для тех, кто дочитал до сюда)

> все это можно нати в официальной документации rabbitmq (`https://www.rabbitmq.com/tutorials/tutorial-one-python.html`)

> там аж 6 страниц этого туториала, я здесь разобрала только первую часть.

> там в конце каждой страницы есть ссылка на следующую страницу, но если ее тяжело найти, то просто в url замените `one` на нужную вам страницу (типо `two`, `tree` ... `six`)
