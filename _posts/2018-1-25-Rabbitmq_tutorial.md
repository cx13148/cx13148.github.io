---
layout:     post
title:      "RabbitMQ官方教程整理"
subtitle:   ""
date:       2018-01-25
author:     "yohaha"
header-img: ""
---

> 源自RabbitMQ官方的PHP教程。


[0d9751f7]: https://www.rabbitmq.com/download.html "install"
[b7523268]: https://github.com/rabbitmq/rabbitmq-tutorials/tree/master/php "tutorial"


## 安装配置
#### 安装参考官方指南
先装对应版本的Erlang,然后再装rabbitmq，根据需要安装php扩展。
[Ref Page][0d9751f7]  

#### 默认参数
用户：`guest`，密码：`guest`。  
端口：  

    4369 (epmd), 25672 (Erlang distribution)
    5672, 5671 (AMQP 0-9-1 without and with TLS)
    15672 (if management plugin is enabled)
    61613, 61614 (if STOMP is enabled)
    1883, 8883 (if MQTT is enabled)

#### 打开web后台管理：

进入rabbitmq安装目录中的sbin目录执行`rabbitmq-plugins enable rabbitmq_management`  重启rabbitmq服务生效

## 官方教程
[Ref github page.][b7523268]
#### 1. Hello World
The simplest thing that does something  
此例直接将msg推到特定的queue中

1. 用composer安装依赖，新建composer.json：

    {
        "require": {
            "php-amqplib/php-amqplib": ">=2.6.1"
        }
    }

    运行composer install 生成vender文件夹。

2. send.php

    declare默认为创建操作，如果rabbit中已经存在相同属性和名字的实例时则相当于选中他。
    注意不能声明同名但属性不同的实例。  
    ```
        <?php

        require_once __DIR__ . '/vendor/autoload.php';
        use PhpAmqpLib\Connection\AMQPStreamConnection;
        use PhpAmqpLib\Message\AMQPMessage;

        $connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');

        $channel = $connection->channel();

        $channel->queue_declare('hello', false, false, false, false);

        $msg = new AMQPMessage('Hello World!');
        $channel->basic_publish($msg, '', 'hello');

        echo " [x] Sent 'Hello World!'\n";

        $channel->close();
        $connection->close();
        ?>
    ```
3. receive.php
    ```
        <?php

        require_once __DIR__ . '/vendor/autoload.php';
        use PhpAmqpLib\Connection\AMQPStreamConnection;

        $connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
        $channel = $connection->channel();

        $channel->queue_declare('hello', false, false, false, false);

        echo ' [*] Waiting for messages. To exit press CTRL+C', "\n";

        $callback = function($msg) {
          echo " [x] Received ", $msg->body, "\n";
        };


        $channel->basic_consume('hello', '', false, true, false, false, $callback);


        while(count($channel->callbacks)) {
            $channel->wait();
        }


        $channel->close();
        $connection->close();
        ?>
    ```

可以用`rabbitmqctl list_queues`命令来查看各个queue内的消息数量。


#### 2. Work queues  
Distributing tasks among workers  
默认情况下，rabbit会以轮询方式发送message，即每个consumer会获得等量的message，不管是否收到ack。我们可以用channel中basic_qos方法的prefetch_count设置为1来让rabbit只分配给每个工人最多1个message，收到ack之前不会再发给未反馈ack的worker，转而发给其他已经反馈ack的worker。  
`$channel->basic_qos(null, 1, null);`

1. new_task.php  

    ```
        <?php

        require_once __DIR__ . '/vendor/autoload.php';
        use PhpAmqpLib\Connection\AMQPStreamConnection;
        use PhpAmqpLib\Message\AMQPMessage;

        $connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
        $channel = $connection->channel();

        //第三个参数为durable
        //更改queue的声明选项需要在生产/消费两边同时更改
        $channel->queue_declare('task_queue', false, true, false, false);

        $data = implode(' ', array_slice($argv, 1));
        if(empty($data)) $data = "Hello World!";
        $msg = new AMQPMessage($data,
                                array('delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT)
                              );

        $channel->basic_publish($msg, '', 'task_queue');

        echo " [x] Sent ", $data, "\n";

        $channel->close();
        $connection->close();

        ?>
    ```


2. worker.php

    ```
        <?php

        require_once __DIR__ . '/vendor/autoload.php';
        use PhpAmqpLib\Connection\AMQPStreamConnection;

        $connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
        $channel = $connection->channel();

        $channel->queue_declare('task_queue', false, true, false, false);

        echo ' [*] Waiting for messages. To exit press CTRL+C', "\n";

        $callback = function($msg){
          echo " [x] Received ", $msg->body, "\n";
          sleep(3);
          echo " [x] Done", "\n";
          $msg->delivery_info['channel']->basic_ack($msg->delivery_info['delivery_tag']);
        };

        $channel->basic_qos(null, 1, null);

        //basic_consume()中第四个参数表示是否自动发送ack,false为需要手动发送ack
        //rabbit中只有收到ack才会将queue中的信息删除，没收到ack,在consumer die(channel/connection closed, or TCP connection is lost)了之后才会重发
        $channel->basic_consume('task_queue', '', false, false, false, false, $callback);

        while(count($channel->callbacks)) {
            $channel->wait();
        }

        $channel->close();
        $connection->close();

        ?>
    ```

    **Forgotten acknowledgment**  

    It's a common mistake to miss the ack. It's an easy error, but the consequences are serious. Messages will be redelivered when your client quits (which may look like random redelivery), but RabbitMQ will eat more and more memory as it won't be able to release any unacked messages.

    In order to debug this kind of mistake you can use rabbitmqctl to print the messages_unacknowledged field:

    `sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged`  

### 3. Publish/Subscribe  
Sending messages to many consumers at once  
本例和上一个类似，仅将信息改为publish到logs exchange来取代默认exchange,同时将分发方式改为fanout。
1. emit_log.php
    ```
        <?php

        require_once __DIR__ . '/vendor/autoload.php';
        use PhpAmqpLib\Connection\AMQPStreamConnection;
        use PhpAmqpLib\Message\AMQPMessage;

        $connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
        $channel = $connection->channel();


        $channel->exchange_declare('logs', 'fanout', false, false, false);

        $data = implode(' ', array_slice($argv, 1));
        if(empty($data)) $data = "info: Hello World!";
        $msg = new AMQPMessage($data);

        $channel->basic_publish($msg, 'logs');

        echo " [x] Sent ", $data, "\n";

        $channel->close();
        $connection->close();

        ?>

    ```


2. receive_logs.php
    ```
        <?php

        require_once __DIR__ . '/vendor/autoload.php';
        use PhpAmqpLib\Connection\AMQPStreamConnection;

        $connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
        $channel = $connection->channel();

        //讲道理消费者应该不接触exchange,但是exchange和queue的绑定在这里完成，再写一遍应该是防止先运行消费者的情况会报错
        $channel->exchange_declare('logs', 'fanout', false, false, false);

        //这里queue用""声明会自动生成个名字并存到$queue_name中,适用于生成非持久匿名queue
        list($queue_name, ,) = $channel->queue_declare("", false, false, true, false);

        $channel->queue_bind($queue_name, 'logs');

        echo ' [*] Waiting for logs. To exit press CTRL+C', "\n";

        $callback = function($msg){
          echo ' [x] ', $msg->body, "\n";
        };

        $channel->basic_consume($queue_name, '', false, true, false, false, $callback);

        while(count($channel->callbacks)) {
            $channel->wait();
        }

        $channel->close();
        $connection->close();

        ?>
    ```


    **Listing bindings**
    You can list existing bindings using, you guessed it,

        rabbitmqctl list_bindings

    **Listing exchanges**  
    To list the exchanges on the server you can run the ever useful rabbitmqctl:

        sudo rabbitmqctl list_exchanges
    In this list there will be some amq.* exchanges and the default (unnamed) exchange. These are created by default, but it is unlikely you'll need to use them at the moment.

    **The default exchange**  
    In previous parts of the tutorial we knew nothing about exchanges, but still were able to send messages to queues. That was possible because we were using a default exchange, which we identify by the empty string ("").

    Recall how we published a message before:

        $channel->basic_publish($msg, '', 'hello');
    Here we use the default or nameless exchange: messages are routed to the queue with the name specified by routing_key, if it exists. The routing key is the third argument to basic_publish
### 4. Routing  
Receiving messages selectively  
direct型exchange会将msg按照tag转发到对应的queue。
1. emit_log_direct.php

    ```
        <?php

        require_once __DIR__ . '/vendor/autoload.php';
        use PhpAmqpLib\Connection\AMQPStreamConnection;
        use PhpAmqpLib\Message\AMQPMessage;

        $connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
        $channel = $connection->channel();

        $channel->exchange_declare('direct_logs', 'direct', false, false, false);

        $severity = isset($argv[1]) && !empty($argv[1]) ? $argv[1] : 'info';

        $data = implode(' ', array_slice($argv, 2));
        if(empty($data)) $data = "Hello World!";

        $msg = new AMQPMessage($data);

        $channel->basic_publish($msg, 'direct_logs', $severity);

        echo " [x] Sent ",$severity,':',$data," \n";

        $channel->close();
        $connection->close();

        ?>
    ```
2. receive_logs_direct.php

    ```
        <?php

        require_once __DIR__ . '/vendor/autoload.php';
        use PhpAmqpLib\Connection\AMQPStreamConnection;

        $connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
        $channel = $connection->channel();

        $channel->exchange_declare('direct_logs', 'direct', false, false, false);

        list($queue_name, ,) = $channel->queue_declare("", false, false, true, false);

        $severities = array_slice($argv, 1);
        if(empty($severities )) {
        	file_put_contents('php://stderr', "Usage: $argv[0] [info] [warning] [error]\n");
        	exit(1);
        }

        //和exchange绑定，同时注册tag
        foreach($severities as $severity) {
            $channel->queue_bind($queue_name, 'direct_logs', $severity);
        }

        echo ' [*] Waiting for logs. To exit press CTRL+C', "\n";

        $callback = function($msg){
          echo ' [x] '.$msg->delivery_info['routing_key'].':'.$msg->body."\n";
        };

        $channel->basic_consume($queue_name, '', false, true, false, false, $callback);

        while(count($channel->callbacks)) {
            $channel->wait();
        }

        $channel->close();
        $connection->close();

        ?>
    ```
### 5. Topics  
Receiving messages based on a pattern (topics)  
可以在消费者端注册多个感兴趣的topic，和direct绑定方法一致，用channel的queue_bind方法指定queue,exchange名和tag，可以多次绑定（不同的exchange，不同的tag）。
效率排序为fanout > direct >> topic
`* (star)` can substitute for exactly one word.
`# (hash)` can substitute for zero or more words.
1. emit_log_topic.php

    ```
        <?php

        require_once __DIR__ . '/vendor/autoload.php';
        use PhpAmqpLib\Connection\AMQPStreamConnection;
        use PhpAmqpLib\Message\AMQPMessage;

        $connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
        $channel = $connection->channel();


        $channel->exchange_declare('topic_logs', 'topic', false, false, false);

        $routing_key = isset($argv[1]) && !empty($argv[1]) ? $argv[1] : 'anonymous.info';
        $data = implode(' ', array_slice($argv, 2));
        if(empty($data)) $data = "Hello World!";

        $msg = new AMQPMessage($data);

        $channel->basic_publish($msg, 'topic_logs', $routing_key);

        echo " [x] Sent ",$routing_key,':',$data," \n";

        $channel->close();
        $connection->close();

        ?>
    ```

2. receive_logs_topic.Php

    ```
        <?php

        require_once __DIR__ . '/vendor/autoload.php';
        use PhpAmqpLib\Connection\AMQPStreamConnection;

        $connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
        $channel = $connection->channel();

        $channel->exchange_declare('topic_logs', 'topic', false, false, false);

        list($queue_name, ,) = $channel->queue_declare("", false, false, true, false);

        $binding_keys = array_slice($argv, 1);
        if( empty($binding_keys )) {
        	file_put_contents('php://stderr', "Usage: $argv[0] [binding_key]\n");
        	exit(1);
        }

        foreach($binding_keys as $binding_key) {
        	$channel->queue_bind($queue_name, 'topic_logs', $binding_key);
        }

        echo ' [*] Waiting for logs. To exit press CTRL+C', "\n";

        $callback = function($msg){
          echo ' [x] ',$msg->delivery_info['routing_key'], ':', $msg->body, "\n";
        };

        $channel->basic_consume($queue_name, '', false, true, false, false, $callback);

        while(count($channel->callbacks)) {
            $channel->wait();
        }

        $channel->close();
        $connection->close();

        ?>

    ```
### 6. RPC
Request/reply pattern example 即远程调用


1. rpc_client.php

    ```
        <?php

        require_once __DIR__ . '/vendor/autoload.php';
        use PhpAmqpLib\Connection\AMQPStreamConnection;
        use PhpAmqpLib\Message\AMQPMessage;

        class FibonacciRpcClient {
        	private $connection;
        	private $channel;
        	private $callback_queue;
        	private $response;
        	private $corr_id;

        	public function __construct() {
        		$this->connection = new AMQPStreamConnection(
        			'localhost', 5672, 'guest', 'guest');
        		$this->channel = $this->connection->channel();
        		list($this->callback_queue, ,) = $this->channel->queue_declare(
        			"", false, false, true, false);
        		$this->channel->basic_consume(
        			$this->callback_queue, '', false, false, false, false,
        			array($this, 'on_response'));
        	}
        	public function on_response($rep) {
        		if($rep->get('correlation_id') == $this->corr_id) {
        			$this->response = $rep->body;
        		}
        	}

        	public function call($n) {
        		$this->response = null;
        		$this->corr_id = uniqid();

        		$msg = new AMQPMessage(
        			(string) $n,
        			array('correlation_id' => $this->corr_id,
        			      'reply_to' => $this->callback_queue)
        			);
        		$this->channel->basic_publish($msg, '', 'rpc_queue');
        		while(!$this->response) {
        			$this->channel->wait();
        		}
        		return intval($this->response);
        	}
        };

        $fibonacci_rpc = new FibonacciRpcClient();
        $response = $fibonacci_rpc->call(30);
        echo " [.] Got ", $response, "\n";

        ?>
    ```

2. rpc_server.php

    ```
        <?php

        require_once __DIR__ . '/vendor/autoload.php';
        use PhpAmqpLib\Connection\AMQPStreamConnection;
        use PhpAmqpLib\Message\AMQPMessage;

        $connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
        $channel = $connection->channel();

        $channel->queue_declare('rpc_queue', false, false, false, false);

        function fib($n) {
        	if ($n == 0)
        		return 0;
        	if ($n == 1)
        		return 1;
        	return fib($n-1) + fib($n-2);
        }

        echo " [x] Awaiting RPC requests\n";
        $callback = function($req) {
        	$n = intval($req->body);
        	echo " [.] fib(", $n, ")\n";

        	$msg = new AMQPMessage(
        		(string) fib($n),
        		array('correlation_id' => $req->get('correlation_id'))
        		);

        	$req->delivery_info['channel']->basic_publish(
        		$msg, '', $req->get('reply_to'));
        	$req->delivery_info['channel']->basic_ack(
        		$req->delivery_info['delivery_tag']);
        };

        $channel->basic_qos(null, 1, null);
        $channel->basic_consume('rpc_queue', '', false, false, false, false, $callback);

        while(count($channel->callbacks)) {
            $channel->wait();
        }

        $channel->close();
        $connection->close();

        ?>
    ```
