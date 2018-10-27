# join

查询构造器提供`join`方法用来构建join语句。在join语句中，连接条件分为两种，一种是对比两个连接表的列，一种是将连接表中的列和用户提供的值
对比。`join`方法要很好地处理这两种情况。

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Add a join clause to the query.
 *
 * @param  string  $table
 * @param  string  $first
 * @param  string|null  $operator
 * @param  string|null  $second
 * @param  string  $type
 * @param  bool    $where
 * @return $this
 */
public function join($table, $first, $operator = null, $second = null, $type = 'inner', $where = false)
{
    $join = new JoinClause($this, $type, $table);

    // If the first "column" of the join is really a Closure instance the developer
    // is trying to build a join with a complex "on" clause containing more than
    // one condition, so we'll add the join and call a Closure with the query.
    if ($first instanceof Closure) {
        call_user_func($first, $join);

        $this->joins[] = $join;

        $this->addBinding($join->getBindings(), 'join');
    }

    // If the column is simply a string, we can assume the join simply has a basic
    // "on" clause with a single condition. So we will just build the join with
    // this simple join clauses attached to it. There is not a join callback.
    else {
        $method = $where ? 'where' : 'on';

        $this->joins[] = $join->$method($first, $operator, $second);

        $this->addBinding($join->getBindings(), 'join');
    }

    return $this;
}
```

`JoinClause`是`Builder`类的子类，它提供了一些join语句中要用到的方法，例如`on`，用来指定连接条件。在`JoinClause`实例中，无论是
使用`on`还是`where`指定连接条件，这部分数据都是存放在查询构造器的`$wheres`属性里，`Grammer`类在转换这部分数据的时候会再做判断。


```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile the "join" portions of the query.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $joins
 * @return string
 */
protected function compileJoins(Builder $query, $joins)
{
    return collect($joins)->map(function ($join) use ($query) {
        $table = $this->wrapTable($join->table);

        $nestedJoins = is_null($join->joins) ? '' : ' '.$this->compileJoins($query, $join->joins);

        return trim("{$join->type} join {$table}{$nestedJoins} {$this->compileWheres($join)}");
    })->implode(' ');
}

/**
 * Compile the "where" portions of the query.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @return string
 */
protected function compileWheres(Builder $query)
{
    // Each type of where clauses has its own compiler function which is responsible
    // for actually creating the where clauses SQL. This helps keep the code nice
    // and maintainable since each clause has a very small method that it uses.
    if (is_null($query->wheres)) {
        return '';
    }

    // If we actually have some where clauses, we will strip off the first boolean
    // operator, which is added by the query builders for convenience so we can
    // avoid checking for the first clauses in each of the compilers methods.
    if (count($sql = $this->compileWheresToArray($query)) > 0) {
        return $this->concatenateWhereClauses($query, $sql);
    }

    return '';
}

/**
 * Get an array of all the where clauses for the query.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @return array
 */
protected function compileWheresToArray($query)
{
    return collect($query->wheres)->map(function ($where) use ($query) {
        return $where['boolean'].' '.$this->{"where{$where['type']}"}($query, $where);
    })->all();
}

/**
 * Format the where clause statements into one string.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $sql
 * @return string
 */
protected function concatenateWhereClauses($query, $sql)
{
    $conjunction = $query instanceof JoinClause ? 'on' : 'where';

    return $conjunction.' '.$this->removeLeadingBoolean(implode(' ', $sql));
}
```

在`compileJoins`可以看到，它是使用`compileWheres`来生成连接条件的。在`concatenateWhereClauses`里面做了判断，当
`$query`是`JoinClause`的实例的时候，它用`"on"`来连接后面的条件，而不是`"where"`。

```php
//src/Illuminate/Database/Query/JoinClause.php

/**
 * Add an "on" clause to the join.
 *
 * On clauses can be chained, e.g.
 *
 *  $join->on('contacts.user_id', '=', 'users.id')
 *       ->on('contacts.info_id', '=', 'info.id')
 *
 * will produce the following SQL:
 *
 * on `contacts`.`user_id` = `users`.`id`  and `contacts`.`info_id` = `info`.`id`
 *
 * @param  \Closure|string  $first
 * @param  string|null  $operator
 * @param  string|null  $second
 * @param  string  $boolean
 * @return $this
 *
 * @throws \InvalidArgumentException
 */
public function on($first, $operator = null, $second = null, $boolean = 'and')
{
    if ($first instanceof Closure) {
        return $this->whereNested($first, $boolean);
    }

    return $this->whereColumn($first, $operator, $second, $boolean);
}
```

在`JoinClause`的`on`方法实现中也可以看到，它是用了`whereNested`和`whereColumn`这两个用来构建where条件的方法。

