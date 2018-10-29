# truncate

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Run a truncate statement on the table.
 *
 * @return void
 */
public function truncate()
{
    foreach ($this->grammar->compileTruncate($this) as $sql => $bindings) {
        $this->connection->statement($sql, $bindings);
    }
}
```

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile a truncate table statement into SQL.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @return array
 */
public function compileTruncate(Builder $query)
{
    return ['truncate '.$this->wrapTable($query->from) => []];
}
```

这里不太理解的是`compileTruncate`的实现，它完全可以这样写

```php
return ['truncate ' . $this->wrapTable($query->from), []];
```

然后在`truncate`方法里面这样调用

```php
$this->connection->statement(...$this->grammer->compileTruncate($this));
```