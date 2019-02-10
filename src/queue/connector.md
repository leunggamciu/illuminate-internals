# 连接器

连接器的职责是与消息中间件建立连接，它的接口只有一个`connect`方法

```php
//src/Illuminate/Queue/Connectors/ConnectorInterface.php

interface ConnectorInterface
{
    /**
     * Establish a queue connection.
     *
     * @param  array  $config
     * @return \Illuminate\Contracts\Queue\Queue
     */
    public function connect(array $config);
}
```

它接收配置信息，然后根据配置信息创建一个`\Illuminate\Contracts\Queue\Queue`实例，用来表示一个队列连接。

由于Laravel支持多种消息中间件，这里先看看把数据库当做消息中间件的情况

```php
//src/Illuminate/Queue/Connectors/DatabaseConnector.php

/**
 * Create a new connector instance.
 *
 * @param  \Illuminate\Database\ConnectionResolverInterface  $connections
 * @return void
 */
public function __construct(ConnectionResolverInterface $connections)
{
    $this->connections = $connections;
}

 /**
 * Establish a queue connection.
 *
 * @param  array  $config
 * @return \Illuminate\Contracts\Queue\Queue
 */
public function connect(array $config)
{
    return new DatabaseQueue(
        $this->connections->connection($config['connection'] ?? null),
        $config['table'],
        $config['queue'],
        $config['retry_after'] ?? 60
    );
} 
```

`ConnectionResolveInterface`是一个数据库连接解析器，它能根据配置返回对应的数据库连接，在容器中，它的标识就是`db`。