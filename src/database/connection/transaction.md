# 事务管理

`Connection`还提供了事务处理的接口，其中`beginTransaction`用于开启一个事务

```php
//src/Illuminate/Database/Concerns/ManagesTransactions.php

/**
 * Start a new database transaction.
 *
 * @return void
 * @throws \Exception
 */
public function beginTransaction()
{
    $this->createTransaction();

    $this->transactions++;

    $this->fireConnectionEvent('beganTransaction');
}
```

它主要的工作就是向数据库发送事务开始的语句。`$this->transactions`是一个用来跟踪嵌套事务的数量，当它的值大于等于1的时候，就
表明当前的`Connection`是处于事务状态的。

```php
//src/Illuminate/Database/Concerns/ManagesTransactions.php

/**
 * Create a transaction within the database.
 *
 * @return void
 */
protected function createTransaction()
{
    if ($this->transactions == 0) {
        try {
            $this->getPdo()->beginTransaction();
        } catch (Exception $e) {
            $this->handleBeginTransactionException($e);
        }
    } elseif ($this->transactions >= 1 && $this->queryGrammar->supportsSavepoints()) {
        $this->createSavepoint();
    }
}
```

`createTransaction`除了发送开启事务的语句以外，在数据库支持savepoint的前提下，如果当前开启的是一个嵌套事务，那么
它还会发送保存savepoint的语句，这样在事务回滚的时候就只回滚该嵌套事务。还有就是开启事务的时候，如果发现连接丢失，那么
会重新开启事务，这只在开启的事务为非嵌套事务时才可用，`handleBeginTransactionException`就负责处理这部分工作。

```php
//src/Illuminate/Database/Concerns/ManagesTransactions.php

/**
 * Handle an exception from a transaction beginning.
 *
 * @param  \Throwable  $e
 * @return void
 *
 * @throws \Exception
 */
protected function handleBeginTransactionException($e)
{
    if ($this->causedByLostConnection($e)) {
        $this->reconnect();

        $this->pdo->beginTransaction();
    } else {
        throw $e;
    }
}
```

事务提交由`commit`方法完成

```php
//src/Illuminate/Database/Concerns/ManagesTransactions.php

/**
 * Commit the active database transaction.
 *
 * @return void
 */
public function commit()
{
    if ($this->transactions == 1) {
        $this->getPdo()->commit();
    }

    $this->transactions = max(0, $this->transactions - 1);

    $this->fireConnectionEvent('committed');
}
```

只有在最外层事务的时候，才会向数据库发送commit语句，然后减少嵌套事务的数量。

```php
//src/Illuminate/Database/Concerns/ManagesTransactions.php

/**
 * Rollback the active database transaction.
 *
 * @param  int|null  $toLevel
 * @return void
 */
public function rollBack($toLevel = null)
{
    // We allow developers to rollback to a certain transaction level. We will verify
    // that this given transaction level is valid before attempting to rollback to
    // that level. If it's not we will just return out and not attempt anything.
    $toLevel = is_null($toLevel)
                ? $this->transactions - 1
                : $toLevel;

    if ($toLevel < 0 || $toLevel >= $this->transactions) {
        return;
    }

    // Next, we will actually perform this rollback within this database and fire the
    // rollback event. We will also set the current transaction level to the given
    // level that was passed into this method so it will be right from here out.
    $this->performRollBack($toLevel);

    $this->transactions = $toLevel;

    $this->fireConnectionEvent('rollingBack');
}

/**
 * Perform a rollback within the database.
 *
 * @param  int  $toLevel
 * @return void
 */
protected function performRollBack($toLevel)
{
    if ($toLevel == 0) {
        $this->getPdo()->rollBack();
    } elseif ($this->queryGrammar->supportsSavepoints()) {
        $this->getPdo()->exec(
            $this->queryGrammar->compileSavepointRollBack('trans'.($toLevel + 1))
        );
    }
}
```

默认情况下，如果数据库支持savepoint，那么`rollback`是回滚至前一个savepoint，它也可以指定回滚的savepoint。
在不支持savepoint的数据库中，那么就回滚整个事务。

`Connection`中还提供了一个更友好的接口用作事务处理：`transaction`

```php
//src/Illuminate/Database/Concerns/ManagesTransactions.php

/**
 * Execute a Closure within a transaction.
 *
 * @param  \Closure  $callback
 * @param  int  $attempts
 * @return mixed
 *
 * @throws \Exception|\Throwable
 */
public function transaction(Closure $callback, $attempts = 1)
{
    for ($currentAttempt = 1; $currentAttempt <= $attempts; $currentAttempt++) {
        $this->beginTransaction();

        // We'll simply execute the given callback within a try / catch block and if we
        // catch any exception we can rollback this transaction so that none of this
        // gets actually persisted to a database or stored in a permanent fashion.
        try {
            return tap($callback($this), function ($result) {
                $this->commit();
            });
        }

        // If we catch an exception we'll rollback this transaction and try again if we
        // are not out of attempts. If we are out of attempts we will just throw the
        // exception back out and let the developer handle an uncaught exceptions.
        catch (Exception $e) {
            $this->handleTransactionException(
                $e, $currentAttempt, $attempts
            );
        } catch (Throwable $e) {
            $this->rollBack();

            throw $e;
        }
    }
}
```

通过`transaction`方法，开发者不需要手动`commit`，还能设置事务重试的次数，在发现死锁异常时候，事务可以重试。

```php
//src/Illuminate/Database/Concerns/ManagesTransactions.php

/**
 * Handle an exception encountered when running a transacted statement.
 *
 * @param  \Exception  $e
 * @param  int  $currentAttempt
 * @param  int  $maxAttempts
 * @return void
 *
 * @throws \Exception
 */
protected function handleTransactionException($e, $currentAttempt, $maxAttempts)
{
    // On a deadlock, MySQL rolls back the entire transaction so we can't just
    // retry the query. We have to throw this exception all the way out and
    // let the developer handle it in another way. We will decrement too.
    if ($this->causedByDeadlock($e) &&
        $this->transactions > 1) {
        $this->transactions--;

        throw $e;
    }

    // If there was an exception we will rollback this transaction and then we
    // can check if we have exceeded the maximum attempt count for this and
    // if we haven't we will return and try this query again in our loop.
    $this->rollBack();

    if ($this->causedByDeadlock($e) &&
        $currentAttempt < $maxAttempts) {
        return;
    }

    throw $e;
}
```

如果在嵌套事务中出现死锁异常，那么是不会触发重试的。如果是其他类型的异常的话，那么就将事务回滚，然后重试。

还有一个比较重要的点，就是在开启读写分离的时候，事务的处理。如果在事务操作中，读和写是分别作用在不同的数据库，
那么这个事务就有很大问题了。所以为了防止这种情况，在开启了事务的情况下，不管读和写都是作用在写库（或者说主库）的。

```php
//src/Illuminate/Database/Connection.php

/**
 * Get the current PDO connection used for reading.
 *
 * @return \PDO
 */
public function getReadPdo()
{
    if ($this->transactions > 0) {
        return $this->getPdo();
    }

    if ($this->getConfig('sticky') && $this->recordsModified) {
        return $this->getPdo();
    }

    if ($this->readPdo instanceof Closure) {
        return $this->readPdo = call_user_func($this->readPdo);
    }

    return $this->readPdo ?: $this->getPdo();
}
```

`getReadPdo`就解决了这个问题，当处于事务当中的时候，它使用的是主库的`PDO`实例。开启，回滚，提交这些操作也是。

```php
//src/Illuminate/Database/Connection.php

/**
 * Create a transaction within the database.
 *
 * @return void
 */
protected function createTransaction()
{
    if ($this->transactions == 0) {
        try {
            $this->getPdo()->beginTransaction();
        } catch (Exception $e) {
            $this->handleBeginTransactionException($e);
        }
    } elseif ($this->transactions >= 1 && $this->queryGrammar->supportsSavepoints()) {
        $this->createSavepoint();
    }
}

/**
 * Create a save point within the database.
 *
 * @return void
 */
protected function createSavepoint()
{
    $this->getPdo()->exec(
        $this->queryGrammar->compileSavepoint('trans'.($this->transactions + 1))
    );
}

/**
 * Commit the active database transaction.
 *
 * @return void
 */
public function commit()
{
    if ($this->transactions == 1) {
        $this->getPdo()->commit();
    }

    $this->transactions = max(0, $this->transactions - 1);

    $this->fireConnectionEvent('committed');
}

/**
 * Perform a rollback within the database.
 *
 * @param  int  $toLevel
 * @return void
 */
protected function performRollBack($toLevel)
{
    if ($toLevel == 0) {
        $this->getPdo()->rollBack();
    } elseif ($this->queryGrammar->supportsSavepoints()) {
        $this->getPdo()->exec(
            $this->queryGrammar->compileSavepointRollBack('trans'.($toLevel + 1))
        );
    }
}
```

总的来说，涉及事务的操作都是作用在主库。