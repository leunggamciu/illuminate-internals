# 连接丢失处理

客户端与数据库服务器之间的连接有可能会丢失，Laravel会在连接丢失后尝试重连。

具体地，`Connection`在处理数据库操作的时候，会捕获所有异常

```php
//src/Illuminate/Database/Connection.php

/**
 * Run a SQL statement.
 *
 * @param  string    $query
 * @param  array     $bindings
 * @param  \Closure  $callback
 * @return mixed
 *
 * @throws \Illuminate\Database\QueryException
 */
protected function runQueryCallback($query, $bindings, Closure $callback)
{
    // To execute the statement, we'll simply call the callback, which will actually
    // run the SQL against the PDO connection. Then we can calculate the time it
    // took to execute and log the query SQL, bindings and time in our memory.
    try {
        $result = $callback($query, $bindings);
    }

    // If an exception occurs when attempting to run a query, we'll format the error
    // message to include the bindings with SQL, which will make this exception a
    // lot more helpful to the developer instead of just the database's errors.
    catch (Exception $e) {
        throw new QueryException(
            $query, $this->prepareBindings($bindings), $e
        );
    }

    return $result;
}
```

并将这些异常转换成更具体的`QueryException`，在上层方法中会捕获这些异常

```php
//src/Illuminate/Database/Connection.php

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

在`handleQueryException`中尝试分析异常发生的原因，如果是由于服务器连接丢失，那么就触发重连，并尝试再次操作。

```php
//src/Illuminate/Database/Connection.php

/**
 * Handle a query exception.
 *
 * @param  \Exception  $e
 * @param  string  $query
 * @param  array  $bindings
 * @param  \Closure  $callback
 * @return mixed
 * @throws \Exception
 */
protected function handleQueryException($e, $query, $bindings, Closure $callback)
{
    if ($this->transactions >= 1) {
        throw $e;
    }

    return $this->tryAgainIfCausedByLostConnection(
        $e, $query, $bindings, $callback
    );
}

/**
 * Handle a query exception that occurred during query execution.
 *
 * @param  \Illuminate\Database\QueryException  $e
 * @param  string    $query
 * @param  array     $bindings
 * @param  \Closure  $callback
 * @return mixed
 *
 * @throws \Illuminate\Database\QueryException
 */
protected function tryAgainIfCausedByLostConnection(QueryException $e, $query, $bindings, Closure $callback)
{
    if ($this->causedByLostConnection($e->getPrevious())) {
        $this->reconnect();

        return $this->runQueryCallback($query, $bindings, $callback);
    }

    throw $e;
}
```

这里有一点要注意的是，如果当前的数据库操作是处于事务当中，那么即使连接丢失，也不会触发重连和重试。

判断是否由连接丢失导致的异常的逻辑很简单，就是看客户端返回的是不是连接丢失的提示

```php
//src/Illuminate/Database/DetectsLostConnections.php

/**
 * Determine if the given exception was caused by a lost connection.
 *
 * @param  \Throwable  $e
 * @return bool
 */
protected function causedByLostConnection(Throwable $e)
{
    $message = $e->getMessage();

    return Str::contains($message, [
        'server has gone away',
        'no connection to the server',
        'Lost connection',
        'is dead or not enabled',
        'Error while sending',
        'decryption failed or bad record mac',
        'server closed the connection unexpectedly',
        'SSL connection has been closed unexpectedly',
        'Error writing data to the connection',
        'Resource deadlock avoided',
        'Transaction() on null',
        'child connection forced to terminate due to client_idle_limit',
        'query_wait_timeout',
        'reset by peer',
    ]);
}
```