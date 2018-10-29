# lock

查询构造器提供`lock`方法用于指定锁的类型。

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Lock the selected rows in the table.
 *
 * @param  string|bool  $value
 * @return $this
 */
public function lock($value = true)
{
    $this->lock = $value;

    if (! is_null($this->lock)) {
        $this->useWritePdo();
    }

    return $this;
}

/**
 * Lock the selected rows in the table for updating.
 *
 * @return \Illuminate\Database\Query\Builder
 */
public function lockForUpdate()
{
    return $this->lock(true);
}

/**
 * Share lock the selected rows in the table.
 *
 * @return \Illuminate\Database\Query\Builder
 */
public function sharedLock()
{
    return $this->lock(false);
}
```

指定了锁的SQL操作都是作用在写库的。

```php
//src/Illuminate/Database/Query/Grammers/MySQLGrammer.php

/**
 * Compile the lock into SQL.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  bool|string  $value
 * @return string
 */
protected function compileLock(Builder $query, $value)
{
    if (! is_string($value)) {
        return $value ? 'for update' : 'lock in share mode';
    }

    return $value;
}
```