# SQL操作

所有SQL操作都是在数据库连接中进行的。`Connection`类提供了`select`，`insert`，`update`，`delete`来对数据库做CRUD操作。另外还有一个
`statement`方法，一般用来执行一些DDL。

```php
//src/Illuminate/Database/Connection.php

/**
 * Run a select statement against the database.
 *
 * @param  string  $query
 * @param  array  $bindings
 * @param  bool  $useReadPdo
 * @return array
 */
public function select($query, $bindings = [], $useReadPdo = true)
{
    return $this->run($query, $bindings, function ($query, $bindings) use ($useReadPdo) {
        if ($this->pretending()) {
            return [];
        }

        // For select statements, we'll simply execute the query and return an array
        // of the database result set. Each element in the array will be a single
        // row from the database table, and will either be an array or objects.
        $statement = $this->prepared($this->getPdoForSelect($useReadPdo)
                          ->prepare($query));

        $this->bindValues($statement, $this->prepareBindings($bindings));

        $statement->execute();

        return $statement->fetchAll();
    });
}

/**
 * Run a SQL statement and log its execution context.
 *
 * @param  string    $query
 * @param  array     $bindings
 * @param  \Closure  $callback
 * @return mixed
 *
 * @throws \Illuminate\Database\QueryException
 */
protected function run($query, $bindings, Closure $callback)
{
    $this->reconnectIfMissingConnection();

    $start = microtime(true);

    // Here we will run this query. If an exception occurs we'll determine if it was
    // caused by a connection that has been lost. If that is the cause, we'll try
    // to re-establish connection and re-run the query with a fresh connection.
    try {
        $result = $this->runQueryCallback($query, $bindings, $callback);
    } catch (QueryException $e) {
        $result = $this->handleQueryException(
            $e, $query, $bindings, $callback
        );
    }

    // Once we have run the query we will calculate the time that it took to run and
    // then log the query, bindings, and execution time so we will report them on
    // the event that the developer needs them. We'll log time in milliseconds.
    $this->logQuery(
        $query, $bindings, $this->getElapsedTime($start)
    );

    return $result;
}
```

Laravel会记录SQL操作的日志，包括SQL语句，执行的耗时。它把这些操作都统一到`run`方法里面。`select`里面的操作都是基本的针对`PDO`的操作，要
留意的一点是它会尝试获取只读的`PDO`来执行select操作。

下面看看`insert`

```php
//src/Illuminate/Database/Connection.php

/**
 * Run an insert statement against the database.
 *
 * @param  string  $query
 * @param  array   $bindings
 * @return bool
 */
public function insert($query, $bindings = [])
{
    return $this->statement($query, $bindings);
}

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

`insert`直接调用了`statement`，它的基本逻辑和`select`差不多，不过它多了一步：`$this->recordsHaveBeenModified()`，
这是为了实现stick select。

当开启了读写分离以后，如果开启了stick选项，那么如果对写库作出的修改，该请求后续的select都作用在写库。这是为了消除读库和写库同步
时延带来的影响。

接着是`update`和`delete`

```php
//src/Illuminate/Database/Connection.php

/**
 * Run an update statement against the database.
 *
 * @param  string  $query
 * @param  array   $bindings
 * @return int
 */
public function update($query, $bindings = [])
{
    return $this->affectingStatement($query, $bindings);
}

/**
 * Run a delete statement against the database.
 *
 * @param  string  $query
 * @param  array   $bindings
 * @return int
 */
public function delete($query, $bindings = [])
{
    return $this->affectingStatement($query, $bindings);
}

/**
 * Run an SQL statement and get the number of rows affected.
 *
 * @param  string  $query
 * @param  array   $bindings
 * @return int
 */
public function affectingStatement($query, $bindings = [])
{
    return $this->run($query, $bindings, function ($query, $bindings) {
        if ($this->pretending()) {
            return 0;
        }

        // For update or delete statements, we want to get the number of rows affected
        // by the statement and return that back to the developer. We'll first need
        // to execute the statement and then we'll use PDO to fetch the affected.
        $statement = $this->getPdo()->prepare($query);

        $this->bindValues($statement, $this->prepareBindings($bindings));

        $statement->execute();

        $this->recordsHaveBeenModified(
            ($count = $statement->rowCount()) > 0
        );

        return $count;
    });
}
```

`affectingStatement`和`statement`主要的区别就是它们的返回值，前者返回受影响的行的数量，后者返回一个布尔值。