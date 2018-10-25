# 读写分离

Laravel支持数据库读写分离，select操作都落到读库，insert, update, delete这些操作都落到写库。除此以外，它还支持stick read，
也就是在写操作之后的读操作是作用在写库的，这是为了消除读写同步的时延所带来的影响。

`Connection`在内部维持着两个`PDO`，在工厂类生产`Connection`时候会根据配置文件来判断是否设置只读`PDO`

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

通过`Connection`类做select的时候，它就会尝试获取只读`PDO`

```php
//src/Illuminate/Database/Connection.php

/**
 * Get the PDO connection to use for a select query.
 *
 * @param  bool  $useReadPdo
 * @return \PDO
 */
protected function getPdoForSelect($useReadPdo = true)
{
    return $useReadPdo ? $this->getReadPdo() : $this->getPdo();
}
```

stick select的实现也很简单，它在对数据库作出修改以后，将`Connection`的状态置为已修改，后续的select操作就判断这个状态，
如果是已修改，那么就不使用只读`PDO`，而使用用于写的`PDO`。

在能对数据库作出修改的方法中的`recordsHaveBeenModified`调用就是用来记录这个状态。


```php
//src/Illuminate/Database/Connection.php

/**
 * Execute an SQL statement and return the boolean result.
 *
 * @param  string  $query
 * @param  array   $bindings
 * @return bool
 */
public function statement($query, $bindings = [])
{
    return $this->run($query, $bindings, function ($query, $bindings) {
        if ($this->pretending()) {
            return true;
        }

        $statement = $this->getPdo()->prepare($query);

        $this->bindValues($statement, $this->prepareBindings($bindings));

        $this->recordsHaveBeenModified();

        return $statement->execute();
    });
}
```