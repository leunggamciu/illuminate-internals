# union

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Add a union statement to the query.
 *
 * @param  \Illuminate\Database\Query\Builder|\Closure  $query
 * @param  bool  $all
 * @return \Illuminate\Database\Query\Builder|static
 */
public function union($query, $all = false)
{
    if ($query instanceof Closure) {
        call_user_func($query, $query = $this->newQuery());
    }

    $this->unions[] = compact('query', 'all');

    $this->addBinding($query->getBindings(), 'union');

    return $this;
}
```

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile the "union" queries attached to the main query.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @return string
 */
protected function compileUnions(Builder $query)
{
    $sql = '';

    foreach ($query->unions as $union) {
        $sql .= $this->compileUnion($union);
    }

    if (! empty($query->unionOrders)) {
        $sql .= ' '.$this->compileOrders($query, $query->unionOrders);
    }

    if (isset($query->unionLimit)) {
        $sql .= ' '.$this->compileLimit($query, $query->unionLimit);
    }

    if (isset($query->unionOffset)) {
        $sql .= ' '.$this->compileOffset($query, $query->unionOffset);
    }

    return ltrim($sql);
}

/**
 * Compile a single union statement.
 *
 * @param  array  $union
 * @return string
 */
protected function compileUnion(array $union)
{
    $conjunction = $union['all'] ? ' union all ' : ' union ';

    return $conjunction.$union['query']->toSql();
}
```

`compileUnions`将`$query->unions`转换成union语句，这里要注意`compileOrders`，`compileLimit`和`compileOffset`这三个语句，它们使用的
是`$query->unionOrders`，`$query->unionLimit`，和`$query->unionOffset`。在查询构造器中，如果调用了union，那么后续的`orderBy`，`limit`和`offset`，它们的参数都是存到对应的和union有关的数据结构中，分别是`$unionOrders`，`$unionLimit`和`$unionOffset`。


```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Set the "offset" value of the query.
 *
 * @param  int  $value
 * @return $this
 */
public function offset($value)
{
    $property = $this->unions ? 'unionOffset' : 'offset';

    $this->$property = max(0, $value);

    return $this;
}

/**
 * Set the "limit" value of the query.
 *
 * @param  int  $value
 * @return $this
 */
public function limit($value)
{
    $property = $this->unions ? 'unionLimit' : 'limit';

    if ($value >= 0) {
        $this->$property = $value;
    }

    return $this;
}

/**
 * Add an "order by" clause to the query.
 *
 * @param  string  $column
 * @param  string  $direction
 * @return $this
 */
public function orderBy($column, $direction = 'asc')
{
    $this->{$this->unions ? 'unionOrders' : 'orders'}[] = [
        'column' => $column,
        'direction' => strtolower($direction) == 'asc' ? 'asc' : 'desc',
    ];

    return $this;
}
```

`offset`，`limit`和`orderBy`这三个方法都有一段逻辑判断到底放到哪个属性中。