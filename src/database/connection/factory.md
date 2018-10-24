# 连接工厂

所有对数据库的操作都是通过数据库连接（connection），针对不同的数据库类型，它们支持的操作可能不一样，Laravel支持
多种数据库类型。连接工厂的目的就是根据配置文件，生产对应的数据库连接。简单来说，数据库连接是对操作方式的抽象。

不同的数据库，除了操作方式不同以外，它们建立连接的方式也不同，因此，Laravel还对建立连接的方式做了抽象，由数据库
连接器（connector）来负责建立对数据库的连接。

```php
//src/Illuminate/Database/Connectors/ConnectionFactory.php

/**
 * Establish a PDO connection based on the configuration.
 *
 * @param  array   $config
 * @param  string  $name
 * @return \Illuminate\Database\Connection
 */
public function make(array $config, $name = null)
{
    $config = $this->parseConfig($config, $name);

    if (isset($config['read'])) {
        return $this->createReadWriteConnection($config);
    }

    return $this->createSingleConnection($config);
}
```

上面的片段就是连接工厂用于生产连接的方法，很简单，分析了一下配置文件，然后就调用其他方法来构造连接。这里要注意的一点是，
Laravel支持同一个connection，读和写的操作可以分别作用于不同的数据库，所以，如果配置文件中配置了这一项，那么它就应用
`createReadWriteConnection`，而不是`createSingleConnection`。

```php
//src/Illuminate/Database/Connectors/ConnectionFactory.php

/**
 * Create a single database connection instance.
 *
 * @param  array  $config
 * @return \Illuminate\Database\Connection
 */
protected function createSingleConnection(array $config)
{
    $pdo = $this->createPdoResolver($config);

    return $this->createConnection(
        $config['driver'], $pdo, $config['database'], $config['prefix'], $config
    );
}

/**
 * Create a new Closure that resolves to a PDO instance.
 *
 * @param  array  $config
 * @return \Closure
 */
protected function createPdoResolver(array $config)
{
    return array_key_exists('host', $config)
                        ? $this->createPdoResolverWithHosts($config)
                        : $this->createPdoResolverWithoutHosts($config);
}

/**
 * Create a new connection instance.
 *
 * @param  string   $driver
 * @param  \PDO|\Closure     $connection
 * @param  string   $database
 * @param  string   $prefix
 * @param  array    $config
 * @return \Illuminate\Database\Connection
 *
 * @throws \InvalidArgumentException
 */
protected function createConnection($driver, $connection, $database, $prefix = '', array $config = [])
{
    if ($resolver = Connection::getResolver($driver)) {
        return $resolver($connection, $database, $prefix, $config);
    }

    switch ($driver) {
        case 'mysql':
            return new MySqlConnection($connection, $database, $prefix, $config);
        case 'pgsql':
            return new PostgresConnection($connection, $database, $prefix, $config);
        case 'sqlite':
            return new SQLiteConnection($connection, $database, $prefix, $config);
        case 'sqlsrv':
            return new SqlServerConnection($connection, $database, $prefix, $config);
    }

    throw new InvalidArgumentException("Unsupported driver [{$driver}]");
}
```

`Connection`类它的第一个参数是一个`PDO`实例或者是一个`Closure`实例，当它是一个`Closure`的时候，它被称为resolver。
Laravel支持惰性连接，也就是只有真正使用到数据库，才去建立连接，而实现惰性连接就是通过这个resolver。

```php
//src/Illuminate/Database/Connection.php

/**
 * Get the current PDO connection.
 *
 * @return \PDO
 */
public function getPdo()
{
    if ($this->pdo instanceof Closure) {
        return $this->pdo = call_user_func($this->pdo);
    }

    return $this->pdo;
}
```

上面的片段就是`Connection`中获取`PDO`实例的方法，在真正需要它的时候，才调用resolver来获得。

创建resolver有两种方式，分别是`createPdoResolverWithHosts`和`createPdoResolverWithoutHosts`，两个方法在字面上的不同就是
配置文件中到底有没有指定数据库的host。

```php
//src/Illuminate/Database/Connectors/ConnectionFactory.php

/**
 * Create a new Closure that resolves to a PDO instance with a specific host or an array of hosts.
 *
 * @param  array  $config
 * @return \Closure
 */
protected function createPdoResolverWithHosts(array $config)
{
    return function () use ($config) {
        foreach (Arr::shuffle($hosts = $this->parseHosts($config)) as $key => $host) {
            $config['host'] = $host;

            try {
                return $this->createConnector($config)->connect($config);
            } catch (PDOException $e) {
                if (count($hosts) - 1 === $key && $this->container->bound(ExceptionHandler::class)) {
                    $this->container->make(ExceptionHandler::class)->report($e);
                }
            }
        }

        throw $e;
    };
}
```

首先看`createPdoResolverWithHosts`，配置文件中的`host`字段可以是一个字符串，也可以是一个数组，用来指定多个host，如果
它是字符串，那么会被转换成只有一个元素的数组。基本的逻辑就是将host数组随机打乱，然后依次连接，返回首个连接成功的`PDO`实例。

因为Laravel支持数据库的读写分离，读的时候可能会通过多个host，这里在建立连接之前先把host数组打乱是为了保证读不会全部落到同一个
host上。

```php
//src/Illuminate/Database/Connectors/ConnectionFactory.php

/**
 * Create a new Closure that resolves to a PDO instance where there is no configured host.
 *
 * @param  array  $config
 * @return \Closure
 */
protected function createPdoResolverWithoutHosts(array $config)
{
    return function () use ($config) {
        return $this->createConnector($config)->connect($config);
    };
}
```

而`createPdoResolverWithoutHosts`主要是为了应对那些不需要指定host的数据库，例如SQLite。

```php
//src/Illuminate/Database/Connectors/ConnectionFactory.php

/**
 * Create a connector instance based on the configuration.
 *
 * @param  array  $config
 * @return \Illuminate\Database\Connectors\ConnectorInterface
 *
 * @throws \InvalidArgumentException
 */
public function createConnector(array $config)
{
    if (! isset($config['driver'])) {
        throw new InvalidArgumentException('A driver must be specified.');
    }

    if ($this->container->bound($key = "db.connector.{$config['driver']}")) {
        return $this->container->make($key);
    }

    switch ($config['driver']) {
        case 'mysql':
            return new MySqlConnector;
        case 'pgsql':
            return new PostgresConnector;
        case 'sqlite':
            return new SQLiteConnector;
        case 'sqlsrv':
            return new SqlServerConnector;
    }

    throw new InvalidArgumentException("Unsupported driver [{$config['driver']}]");
}
```

`createConnector`主要的作用就是根据配置的驱动，实例化对应的connector。这里有意思的地方是我们可以
通过在服务容器中绑定一个名为`db.connector.驱动名称`的解析函数来建立新的connector或者覆盖Laravel
中的connector实现。

再来看看`createReadWriteConnection`的实现

```php
//src/Illuminate/Database/Connectors/ConnectionFactory.php

/**
 * Create a single database connection instance.
 *
 * @param  array  $config
 * @return \Illuminate\Database\Connection
 */
protected function createReadWriteConnection(array $config)
{
    $connection = $this->createSingleConnection($this->getWriteConfig($config));

    return $connection->setReadPdo($this->createReadPdo($config));
}
```

从这个方法可以看出来，`Connection`中是有两个`PDO`实例的，一个是主要的`PDO`实例，在没有指定读写分离的时候
所有读写操作都通过这个实例，另外一个就是只读实例，当指定读写分离以后，所有查询操作都通过这个实例。