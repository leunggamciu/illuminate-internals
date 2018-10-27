# from

查询构造器中，`from`方法用于指定要操作的表名

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Set the table which the query is targeting.
 *
 * @param  string  $table
 * @return $this
 */
public function from($table)
{
    $this->from = $table;

    return $this;
}
```

`from`语句最终使用`compileFrom`来转换成SQL语句

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile the "from" portion of the query.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  string  $table
 * @return string
 */
protected function compileFrom(Builder $query, $table)
{
    return 'from '.$this->wrapTable($table);
}
```

`compileFrom`调用了`wrapTable`把表名quote起来。

`from('foo')`最终生成出来的SQL语句就是`FROM "foo"`。