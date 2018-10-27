# order by

```php
//src/Illuminate/Database/Query/Builder.php

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

这里先不考虑`$this->unions`为`"unionOrders"`的情况，它的逻辑主要是把参数保存到`$this->orders`中。

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile the "order by" portions of the query.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $orders
 * @return string
 */
protected function compileOrders(Builder $query, $orders)
{
    if (! empty($orders)) {
        return 'order by '.implode(', ', $this->compileOrdersToArray($query, $orders));
    }

    return '';
}

/**
 * Compile the query orders to an array.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $orders
 * @return array
 */
protected function compileOrdersToArray(Builder $query, $orders)
{
    return array_map(function ($order) {
        return ! isset($order['sql'])
                    ? $this->wrap($order['column']).' '.$order['direction']
                    : $order['sql'];
    }, $orders);
}
```

`Grammer`类中的`compileOrders`方法负责将其转换成SQL语句，逻辑也很简单，最主要就是要用`$this->wrap`把列名作quote，以及加上统一
的表前缀。