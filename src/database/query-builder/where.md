# where

查询构造器中的where方法簇用于构建SQL的where语句，它是查询构造器中比较复杂的实现之一。在查询构造器中，
where语句被划分成以下多种类型

* Basic
* Nested
* Sub
* In和NotIn
* InSub和NotInSub
* Exists和NotExists
* Null和NotNull
* Between
* Time, Date, Day, Month, Year
* Column
* RowValues
* Raw

查询构造器将这些类型连同操作数，操作符以及连接条件一起，放到`$wheres`数组中，最后由`Grammer`类将这个数组转化成SQL语句。

## Basic

常用的用于构建Basic类型语句的方法就是`where`，`WHERE a = 1`这种类型的SQL语句就是Basic类型的，如果用查询构造器，那么对应的
调用就是`where('a', '=', 1)`。

在`where`方法中，这种类型的语句是直接放到`$wheres`数组中的，在这个例子中，`$wheres`数组的值就是

```php
$wheres = [
    ['type' => 'Basic', 'column' => 'a', 'operator' => '=', 'value' => 1, 'boolean' => 'and']
]
```

下面是`where`方法中的片段

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Add a basic where clause to the query.
 *
 * @param  string|array|\Closure  $column
 * @param  mixed   $operator
 * @param  mixed   $value
 * @param  string  $boolean
 * @return $this
 */
public function where($column, $operator = null, $value = null, $boolean = 'and')
{
    //....


    // Now that we are working with just a simple query we can put the elements
    // in our array and add the query binding to our array of bindings that
    // will be bound to each SQL statements when it is finally executed.
    $type = 'Basic';

    $this->wheres[] = compact(
        'type', 'column', 'operator', 'value', 'boolean'
    );

    if (! $value instanceof Expression) {
        $this->addBinding($value, 'where');
    }

    return $this;
}
```

然后由`Grammer`类中的`whereBasic`来负责转换这种类型的where语句。

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile a basic where clause.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $where
 * @return string
 */
protected function whereBasic(Builder $query, $where)
{
    $value = $this->parameter($where['value']);

    return $this->wrap($where['column']).' '.$where['operator'].' '.$value;
}
```

它调用`parameter`来将值转换成占位符，然后用`wrap`对列名作quote，最后把它们连接起来

因此，`where('a', '=', 1)`最终生成的SQL就是`WHERE "a" = ?`。

## Nested

在where语句中，我们会使用圆括号来控制运算顺序，例如`WHERE a = 1 AND ( b = 2 OR c = 3 )`，对应的`where`调用就是

```php
where('a', '=', 1)->where(function($query) {
    $query->where('b', '=', 2)->orWhere('c', '=', 3);
});
```

下面是`where`方法中针对Nested类型的where语句的片段

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Add a basic where clause to the query.
 *
 * @param  string|array|\Closure  $column
 * @param  mixed   $operator
 * @param  mixed   $value
 * @param  string  $boolean
 * @return $this
 */
public function where($column, $operator = null, $value = null, $boolean = 'and')
{
    //.....

    // If the columns is actually a Closure instance, we will assume the developer
    // wants to begin a nested where statement which is wrapped in parenthesis.
    // We'll add that Closure to the query then return back out immediately.
    if ($column instanceof Closure) {
        return $this->whereNested($column, $boolean);
    }

    //.....
}
```

它直接调用了`whereNested`

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Add a nested where statement to the query.
 *
 * @param  \Closure $callback
 * @param  string   $boolean
 * @return \Illuminate\Database\Query\Builder|static
 */
public function whereNested(Closure $callback, $boolean = 'and')
{
    call_user_func($callback, $query = $this->forNestedWhere());

    return $this->addNestedWhereQuery($query, $boolean);
}

/**
 * Add another query builder as a nested where to the query builder.
 *
 * @param  \Illuminate\Database\Query\Builder|static $query
 * @param  string  $boolean
 * @return $this
 */
public function addNestedWhereQuery($query, $boolean = 'and')
{
    if (count($query->wheres)) {
        $type = 'Nested';

        $this->wheres[] = compact('type', 'query', 'boolean');

        $this->addBinding($query->getBindings(), 'where');
    }

    return $this;
}
```

`whereNested`建立一个新的查询构造器对象，作为参数传递给`$callback`，然后将这个新对象，连同其他参数一起放入到`$wheres`数组中。

`Grammer`类中使用`whereNested`方法来将`$wheres`转换成SQL语句。

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile a nested where clause.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $where
 * @return string
 */
protected function whereNested(Builder $query, $where)
{
    // Here we will calculate what portion of the string we need to remove. If this
    // is a join clause query, we need to remove the "on" portion of the SQL and
    // if it is a normal query we need to take the leading "where" of queries.
    $offset = $query instanceof JoinClause ? 3 : 6;

    return '('.substr($this->compileWheres($where['query']), $offset).')';
}
```

`compileWheres`实际上就是一个用来生成where语句的方法，它生成的语句不带`WHERE`，`whereNested`再将生成出来的where语句加上圆括号。

## Sub

Sub类型对应的是where语句中出现的子查询，例如`WHERE a = (SELECT b FROM c)`（不考虑SQL语句是否合法），对应查询构造器的调用就是

```php
where('a', '=', function($query) {
    $query->table('c')->select(['b']);
});
```

`where`方法的实现如下

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Add a basic where clause to the query.
 *
 * @param  string|array|\Closure  $column
 * @param  mixed   $operator
 * @param  mixed   $value
 * @param  string  $boolean
 * @return $this
 */
public function where($column, $operator = null, $value = null, $boolean = 'and')
{
    //.....

    // If the value is a Closure, it means the developer is performing an entire
    // sub-select within the query and we will need to compile the sub-select
    // within the where clause to get the appropriate query record results.
    if ($value instanceof Closure) {
        return $this->whereSub($column, $operator, $value, $boolean);
    }

    //.....
}

/**
 * Add a full sub-select to the query.
 *
 * @param  string   $column
 * @param  string   $operator
 * @param  \Closure $callback
 * @param  string   $boolean
 * @return $this
 */
protected function whereSub($column, $operator, Closure $callback, $boolean)
{
    $type = 'Sub';

    // Once we have the query instance we can simply execute it so it can add all
    // of the sub-select's conditions to itself, and then we can cache it off
    // in the array of where clauses for the "main" parent query instance.
    call_user_func($callback, $query = $this->forSubQuery());

    $this->wheres[] = compact(
        'type', 'column', 'operator', 'query', 'boolean'
    );

    $this->addBinding($query->getBindings(), 'where');

    return $this;
}
```

`whereSub`和`whereNested`有点类似，也是新建一个查询构造器对象，然后和其他参数一起放到`$wheres`数组中。然后由`Grammer`类中的
`whereSub`负责转换成SQL。

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile a where condition with a sub-select.
 *
 * @param  \Illuminate\Database\Query\Builder $query
 * @param  array   $where
 * @return string
 */
protected function whereSub(Builder $query, $where)
{
    $select = $this->compileSelect($where['query']);

    return $this->wrap($where['column']).' '.$where['operator']." ($select)";
}
```

`whereSub`和`whereNested`的逻辑差不多，

## In和NotIn

`whereIn`和`whereNotIn`是用来构建In和NotIn类型的where语句，最终生成的SQL语句类似于`WHERE a IN (1, 2, 3)`。


```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Add a "where in" clause to the query.
 *
 * @param  string  $column
 * @param  mixed   $values
 * @param  string  $boolean
 * @param  bool    $not
 * @return $this
 */
public function whereIn($column, $values, $boolean = 'and', $not = false)
{
    $type = $not ? 'NotIn' : 'In';

    //........


    // Next, if the value is Arrayable we need to cast it to its raw array form so we
    // have the underlying array value instead of an Arrayable object which is not
    // able to be added as a binding, etc. We will then add to the wheres array.
    if ($values instanceof Arrayable) {
        $values = $values->toArray();
    }

    $this->wheres[] = compact('type', 'column', 'values', 'boolean');

    // Finally we'll add a binding for each values unless that value is an expression
    // in which case we will just skip over it since it will be the query as a raw
    // string and not as a parameterized place-holder to be replaced by the PDO.
    foreach ($values as $value) {
        if (! $value instanceof Expression) {
            $this->addBinding($value, 'where');
        }
    }

    return $this;
}
```

`Grammer`类中对应的转换方法是`whereIn`和`whereNotIn`

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile a "where in" clause.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $where
 * @return string
 */
protected function whereIn(Builder $query, $where)
{
    if (! empty($where['values'])) {
        return $this->wrap($where['column']).' in ('.$this->parameterize($where['values']).')';
    }

    return '0 = 1';
}

/**
 * Compile a "where not in" clause.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $where
 * @return string
 */
protected function whereNotIn(Builder $query, $where)
{
    if (! empty($where['values'])) {
        return $this->wrap($where['column']).' not in ('.$this->parameterize($where['values']).')';
    }

    return '1 = 1';
}
```

这里有意思的地方是如果`$where['values']`是空值的话，那么返回的是`0 = 1`和`1 = 1`，这样不会因为空值而导致SQL语法错误。

## InSub和NotInSub

InSub和NotInSub产生的SQL语句是类似于`WHERE a IN (SELECT id FROM B)`，对应的查询构造器方法也是`whereIn`和`whereNotIn`

```php
whereIn('a', function($query) {
    $query->from('B')->select(['id']);
});
```
或者

```php
whereIn('a', (new Builder)->from('B')->select(['id']))
```

下面是`whereIn`方法中处理这些逻辑的片段

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Add a "where in" clause to the query.
 *
 * @param  string  $column
 * @param  mixed   $values
 * @param  string  $boolean
 * @param  bool    $not
 * @return $this
 */
public function whereIn($column, $values, $boolean = 'and', $not = false)
{
    $type = $not ? 'NotIn' : 'In';

    if ($values instanceof EloquentBuilder) {
        $values = $values->getQuery();
    }

    // If the value is a query builder instance we will assume the developer wants to
    // look for any values that exists within this given query. So we will add the
    // query accordingly so that this query is properly executed when it is run.
    if ($values instanceof self) {
        return $this->whereInExistingQuery(
            $column, $values, $boolean, $not
        );
    }

    // If the value of the where in clause is actually a Closure, we will assume that
    // the developer is using a full sub-select for this "in" statement, and will
    // execute those Closures, then we can re-construct the entire sub-selects.
    if ($values instanceof Closure) {
        return $this->whereInSub($column, $values, $boolean, $not);
    }

    //......
}

/**
 * Add an external sub-select to the query.
 *
 * @param  string   $column
 * @param  \Illuminate\Database\Query\Builder|static  $query
 * @param  string   $boolean
 * @param  bool     $not
 * @return $this
 */
protected function whereInExistingQuery($column, $query, $boolean, $not)
{
    $type = $not ? 'NotInSub' : 'InSub';

    $this->wheres[] = compact('type', 'column', 'query', 'boolean');

    $this->addBinding($query->getBindings(), 'where');

    return $this;
}

/**
 * Add a where in with a sub-select to the query.
 *
 * @param  string   $column
 * @param  \Closure $callback
 * @param  string   $boolean
 * @param  bool     $not
 * @return $this
 */
protected function whereInSub($column, Closure $callback, $boolean, $not)
{
    $type = $not ? 'NotInSub' : 'InSub';

    // To create the exists sub-select, we will actually create a query and call the
    // provided callback with the query so the developer may set any of the query
    // conditions they want for the in clause, then we'll put it in this array.
    call_user_func($callback, $query = $this->forSubQuery());

    $this->wheres[] = compact('type', 'column', 'query', 'boolean');

    $this->addBinding($query->getBindings(), 'where');

    return $this;
}
```

`whereInExistingQuery`和`whereInSub`都是用来构建NotInSub和InSub类型的where语句，不同的地方在于
`whereInExistingQuery`直接把已有的查询构造器保存起来，而`whereInSub`要重新实例一个，经过`$callback`
处理完以后再保存。

`Grammer`类中的对应的处理方法为

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile a where in sub-select clause.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $where
 * @return string
 */
protected function whereInSub(Builder $query, $where)
{
    return $this->wrap($where['column']).' in ('.$this->compileSelect($where['query']).')';
}

/**
 * Compile a where not in sub-select clause.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $where
 * @return string
 */
protected function whereNotInSub(Builder $query, $where)
{
    return $this->wrap($where['column']).' not in ('.$this->compileSelect($where['query']).')';
}
```

## Exists和NotExists

Exists和NotExists类型的where语句是用来构建类似于`WHERE EXISTS (SELECT id FROM a)`，对应的查询构造器方法是
`whereExists`和`whereNotExists`，用法如下。

```php
whereExists(function($query) {
    $query->from('a')->select(['id']);
})
```

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Add an exists clause to the query.
 *
 * @param  \Closure $callback
 * @param  string   $boolean
 * @param  bool     $not
 * @return $this
 */
public function whereExists(Closure $callback, $boolean = 'and', $not = false)
{
    $query = $this->forSubQuery();

    // Similar to the sub-select clause, we will create a new query instance so
    // the developer may cleanly specify the entire exists query and we will
    // compile the whole thing in the grammar and insert it into the SQL.
    call_user_func($callback, $query);

    return $this->addWhereExistsQuery($query, $boolean, $not);
}

/**
 * Add an exists clause to the query.
 *
 * @param  \Illuminate\Database\Query\Builder $query
 * @param  string  $boolean
 * @param  bool  $not
 * @return $this
 */
public function addWhereExistsQuery(self $query, $boolean = 'and', $not = false)
{
    $type = $not ? 'NotExists' : 'Exists';

    $this->wheres[] = compact('type', 'operator', 'query', 'boolean');

    $this->addBinding($query->getBindings(), 'where');

    return $this;
}
```

对应`Grammer`类中的转换方法为

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile a where exists clause.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $where
 * @return string
 */
protected function whereExists(Builder $query, $where)
{
    return 'exists ('.$this->compileSelect($where['query']).')';
}

/**
 * Compile a where exists clause.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $where
 * @return string
 */
protected function whereNotExists(Builder $query, $where)
{
    return 'not exists ('.$this->compileSelect($where['query']).')';
}
```

这个比较简单。

## Null和NotNull

Null和NotNull类型是用来构建类似于`WHERE a IS NULL`的where语句，对应的查询构造器方法为`whereNull`和`whereNotNull`，
用法如下

```php
whereNull('a');
```

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Add a "where null" clause to the query.
 *
 * @param  string  $column
 * @param  string  $boolean
 * @param  bool    $not
 * @return $this
 */
public function whereNull($column, $boolean = 'and', $not = false)
{
    $type = $not ? 'NotNull' : 'Null';

    $this->wheres[] = compact('type', 'column', 'boolean');

    return $this;
}

/**
 * Add an "or where null" clause to the query.
 *
 * @param  string  $column
 * @return \Illuminate\Database\Query\Builder|static
 */
public function orWhereNull($column)
{
    return $this->whereNull($column, 'or');
}
```

对应的`Grammer`类中的转换方法也很简单

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile a "where null" clause.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $where
 * @return string
 */
protected function whereNull(Builder $query, $where)
{
    return $this->wrap($where['column']).' is null';
}

/**
 * Compile a "where not null" clause.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $where
 * @return string
 */
protected function whereNotNull(Builder $query, $where)
{
    return $this->wrap($where['column']).' is not null';
}
```

## Between

Between类型是用来构建类似于`WHERE a BETWEEN 10 AND 100`，对应的查询构造器方法为`whereBetween`和`whereNotBetween`，
用法如下

```php
whereBetween('a', [10, 100])
```

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Add a where between statement to the query.
 *
 * @param  string  $column
 * @param  array   $values
 * @param  string  $boolean
 * @param  bool  $not
 * @return $this
 */
public function whereBetween($column, array $values, $boolean = 'and', $not = false)
{
    $type = 'between';

    $this->wheres[] = compact('column', 'type', 'boolean', 'not');

    $this->addBinding($values, 'where');

    return $this;
}

/**
 * Add an or where between statement to the query.
 *
 * @param  string  $column
 * @param  array   $values
 * @return \Illuminate\Database\Query\Builder|static
 */
public function orWhereBetween($column, array $values)
{
    return $this->whereBetween($column, $values, 'or');
}
```

`Grammer`类中的转换方法为`whereBetween`，

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile a "between" where clause.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $where
 * @return string
 */
protected function whereBetween(Builder $query, $where)
{
    $between = $where['not'] ? 'not between' : 'between';

    return $this->wrap($where['column']).' '.$between.' ? and ?';
}
```

## Time, Date, Day, Month, Year

Time类型是用来构建类似于`WHERE TIME(a) = '19:00:00'`的SQL语句，对应查询构造器中的方法为`whereTime`。

```php
whereTime('a', '=', '19:00:00');
```

其他Day，Month，Year这些也类似

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Add a "where time" statement to the query.
 *
 * @param  string  $column
 * @param  string   $operator
 * @param  mixed   $value
 * @param  string   $boolean
 * @return \Illuminate\Database\Query\Builder|static
 */
public function whereTime($column, $operator, $value = null, $boolean = 'and')
{
    list($value, $operator) = $this->prepareValueAndOperator(
        $value, $operator, func_num_args() == 2
    );

    return $this->addDateBasedWhere('Time', $column, $operator, $value, $boolean);
}

/**
 * Add a "where date" statement to the query.
 *
 * @param  string  $column
 * @param  string  $operator
 * @param  mixed  $value
 * @param  string  $boolean
 * @return \Illuminate\Database\Query\Builder|static
 */
public function whereDate($column, $operator, $value = null, $boolean = 'and')
{
    list($value, $operator) = $this->prepareValueAndOperator(
        $value, $operator, func_num_args() == 2
    );

    return $this->addDateBasedWhere('Date', $column, $operator, $value, $boolean);
}

/**
 * Add a "where day" statement to the query.
 *
 * @param  string  $column
 * @param  string  $operator
 * @param  mixed  $value
 * @param  string  $boolean
 * @return \Illuminate\Database\Query\Builder|static
 */
public function whereDay($column, $operator, $value = null, $boolean = 'and')
{
    list($value, $operator) = $this->prepareValueAndOperator(
        $value, $operator, func_num_args() == 2
    );

    return $this->addDateBasedWhere('Day', $column, $operator, $value, $boolean);
}

/**
 * Add a "where month" statement to the query.
 *
 * @param  string  $column
 * @param  string  $operator
 * @param  mixed  $value
 * @param  string  $boolean
 * @return \Illuminate\Database\Query\Builder|static
 */
public function whereMonth($column, $operator, $value = null, $boolean = 'and')
{
    list($value, $operator) = $this->prepareValueAndOperator(
        $value, $operator, func_num_args() == 2
    );

    return $this->addDateBasedWhere('Month', $column, $operator, $value, $boolean);
}

/**
 * Add a "where year" statement to the query.
 *
 * @param  string  $column
 * @param  string  $operator
 * @param  mixed  $value
 * @param  string  $boolean
 * @return \Illuminate\Database\Query\Builder|static
 */
public function whereYear($column, $operator, $value = null, $boolean = 'and')
{
    list($value, $operator) = $this->prepareValueAndOperator(
        $value, $operator, func_num_args() == 2
    );

    return $this->addDateBasedWhere('Year', $column, $operator, $value, $boolean);
}

/**
 * Add a date based (year, month, day, time) statement to the query.
 *
 * @param  string  $type
 * @param  string  $column
 * @param  string  $operator
 * @param  mixed  $value
 * @param  string  $boolean
 * @return $this
 */
protected function addDateBasedWhere($type, $column, $operator, $value, $boolean = 'and')
{
    $this->wheres[] = compact('column', 'type', 'boolean', 'operator', 'value');

    if (! $value instanceof Expression) {
        $this->addBinding($value, 'where');
    }

    return $this;
}
```

它们的逻辑都一样，下面是`Grammer`类中用于生成对应SQL的方法。


```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile a "where date" clause.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $where
 * @return string
 */
protected function whereDate(Builder $query, $where)
{
    return $this->dateBasedWhere('date', $query, $where);
}

/**
 * Compile a "where time" clause.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $where
 * @return string
 */
protected function whereTime(Builder $query, $where)
{
    return $this->dateBasedWhere('time', $query, $where);
}

/**
 * Compile a "where day" clause.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $where
 * @return string
 */
protected function whereDay(Builder $query, $where)
{
    return $this->dateBasedWhere('day', $query, $where);
}

/**
 * Compile a "where month" clause.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $where
 * @return string
 */
protected function whereMonth(Builder $query, $where)
{
    return $this->dateBasedWhere('month', $query, $where);
}

/**
 * Compile a "where year" clause.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $where
 * @return string
 */
protected function whereYear(Builder $query, $where)
{
    return $this->dateBasedWhere('year', $query, $where);
}

/**
 * Compile a date based where clause.
 *
 * @param  string  $type
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $where
 * @return string
 */
protected function dateBasedWhere($type, Builder $query, $where)
{
    $value = $this->parameter($where['value']);

    return $type.'('.$this->wrap($where['column']).') '.$where['operator'].' '.$value;
}
```

转换的逻辑都在`dateBasedWhere`方法中。

## Column

Column类型用于生成类似于`WHERE a = b`的SQL语句，对应的查询构造器方法为`whereColumn`，用法如下

```php
whereColumn('a', '=', 'b');
```

查询构造器实现如下

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Add a "where" clause comparing two columns to the query.
 *
 * @param  string|array  $first
 * @param  string|null  $operator
 * @param  string|null  $second
 * @param  string|null  $boolean
 * @return \Illuminate\Database\Query\Builder|static
 */
public function whereColumn($first, $operator = null, $second = null, $boolean = 'and')
{
    // If the column is an array, we will assume it is an array of key-value pairs
    // and can add them each as a where clause. We will maintain the boolean we
    // received when the method was called and pass it into the nested where.
    if (is_array($first)) {
        return $this->addArrayOfWheres($first, $boolean, 'whereColumn');
    }

    // If the given operator is not found in the list of valid operators we will
    // assume that the developer is just short-cutting the '=' operators and
    // we will set the operators to '=' and set the values appropriately.
    if ($this->invalidOperator($operator)) {
        list($second, $operator) = [$operator, '='];
    }

    // Finally, we will add this where clause into this array of clauses that we
    // are building for the query. All of them will be compiled via a grammar
    // once the query is about to be executed and run against the database.
    $type = 'Column';

    $this->wheres[] = compact(
        'type', 'first', 'operator', 'second', 'boolean'
    );

    return $this;
}
```

逻辑也很简单，将参数处理一遍以后就加入到`$this->wheres`数组了。

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile a where clause comparing two columns..
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $where
 * @return string
 */
protected function whereColumn(Builder $query, $where)
{
    return $this->wrap($where['first']).' '.$where['operator'].' '.$this->wrap($where['second']);
}
```

对应的`Grammer`类实现也很简单。

## RowValues

RowValues类型用来生成类似于`WHERE (a, b) = (1, 2)`的SQL语句，在查询构造器中对应的方法为`whereColumn`，它的用法如下

```php
whereColumn(['a', 'b'], '=', [1, 2]);
```

对应的实现为

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Adds a where condition using row values.
 *
 * @param  array   $columns
 * @param  string  $operator
 * @param  array   $values
 * @param  string  $boolean
 * @return $this
 */
public function whereRowValues($columns, $operator, $values, $boolean = 'and')
{
    if (count($columns) != count($values)) {
        throw new InvalidArgumentException('The number of columns must match the number of values');
    }

    $type = 'RowValues';

    $this->wheres[] = compact('type', 'columns', 'operator', 'values', 'boolean');

    $this->addBinding($values);

    return $this;
}
```

对应`Grammer`类的实现为

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile a where row values condition.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $where
 * @return string
 */
protected function whereRowValues(Builder $query, $where)
{
    $values = $this->parameterize($where['values']);

    return '('.implode(', ', $where['columns']).') '.$where['operator'].' ('.$values.')';
}
```

## Raw

这是最简单的实现了

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Add a raw where clause to the query.
 *
 * @param  string  $sql
 * @param  mixed   $bindings
 * @param  string  $boolean
 * @return $this
 */
public function whereRaw($sql, $bindings = [], $boolean = 'and')
{
    $this->wheres[] = ['type' => 'raw', 'sql' => $sql, 'boolean' => $boolean];

    $this->addBinding((array) $bindings, 'where');

    return $this;
}
```

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile a raw where clause.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $where
 * @return string
 */
protected function whereRaw(Builder $query, $where)
{
    return $where['sql'];
}
```