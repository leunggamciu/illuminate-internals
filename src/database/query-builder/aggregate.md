# 聚合函数

查询构造器提供`max`，`min`，`avg`，`sum`， `count`这几个方法用于构建带有聚合函数的SQL语句。以`max`方法举例，它的用法如下

```php
$builder->from('foo')->max('id');
```

它生成出来的SQL语句就是

```SQL
SELECT max(id) FROM foo;
```

下面是`max`方法的实现

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Retrieve the maximum value of a given column.
 *
 * @param  string  $column
 * @return mixed
 */
public function max($column)
{
    return $this->aggregate(__FUNCTION__, [$column]);
}

/**
 * Execute an aggregate function on the database.
 *
 * @param  string  $function
 * @param  array   $columns
 * @return mixed
 */
public function aggregate($function, $columns = ['*'])
{
    $results = $this->cloneWithout(['columns'])
                    ->cloneWithoutBindings(['select'])
                    ->setAggregate($function, $columns)
                    ->get($columns);

    if (! $results->isEmpty()) {
        return array_change_key_case((array) $results[0])['aggregate'];
    }
}
```

这里要注意的是聚合函数这些方法返回的是数据库的记录，它和`get`方法的作用是一样的。`aggregate`方法将查询构造器对象本身复制一遍，但是去掉了
`$columns`属性和select类的数据绑定，`$cloumns`属性记录的是通过`select`方法指定的要查询的列，select类绑定记录的是`selectRaw`时记录的
数据绑定。`aggregate`只返回一行数据，所以类似`SELECT a, sum(b) FROM c group by a`这种语句没法用`aggregate`来查询。

`setAggregate`记录的是要应用的聚合函数名以及要聚合的列名

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Set the aggregate property without running the query.
 *
 * @param  string  $function
 * @param  array  $columns
 * @return $this
 */
protected function setAggregate($function, $columns)
{
    $this->aggregate = compact('function', 'columns');

    if (empty($this->groups)) {
        $this->orders = null;

        $this->bindings['order'] = [];
    }

    return $this;
}
```

从`setAggregate`可以看出，在应用聚合函数时，如果缺乏`group by`语句，那么`order by`语句也不能用。 因为当没有应用`group by`语句时，说明
你把所有行当做一个分组，聚合以后就只有一行，这时候应用`order by`是没有意义的。

下面来看看`Grammer`类如何将其转换成SQL语句

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile an aggregated select clause.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $aggregate
 * @return string
 */
protected function compileAggregate(Builder $query, $aggregate)
{
    $column = $this->columnize($aggregate['columns']);

    // If the query has a "distinct" constraint and we're not asking for all columns
    // we need to prepend "distinct" onto the column name so that the query takes
    // it into account when it performs the aggregating operations on the data.
    if ($query->distinct && $column !== '*') {
        $column = 'distinct '.$column;
    }

    return 'select '.$aggregate['function'].'('.$column.') as aggregate';
}
```

它把要应用聚合函数的列用`columnize`处理一遍，这个方法的作用之前已经说过，就是将标识符quote，为表名加统一前缀。
然后根据是否去掉重复行来决定加不加`distinct`标识符。