# having

having语句有两种构造方法，一个是`having`，另外一个就是`havingRaw`，分别对应两种不同类型的having语句: Basic和Raw。

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Add a "having" clause to the query.
 *
 * @param  string  $column
 * @param  string|null  $operator
 * @param  string|null  $value
 * @param  string  $boolean
 * @return $this
 */
public function having($column, $operator = null, $value = null, $boolean = 'and')
{
    $type = 'Basic';

    // Here we will make some assumptions about the operator. If only 2 values are
    // passed to the method, we will assume that the operator is an equals sign
    // and keep going. Otherwise, we'll require the operator to be passed in.
    list($value, $operator) = $this->prepareValueAndOperator(
        $value, $operator, func_num_args() == 2
    );

    // If the given operator is not found in the list of valid operators we will
    // assume that the developer is just short-cutting the '=' operators and
    // we will set the operators to '=' and set the values appropriately.
    if ($this->invalidOperator($operator)) {
        list($value, $operator) = [$operator, '='];
    }

    $this->havings[] = compact('type', 'column', 'operator', 'value', 'boolean');

    if (! $value instanceof Expression) {
        $this->addBinding($value, 'having');
    }

    return $this;
}

/**
 * Add a raw having clause to the query.
 *
 * @param  string  $sql
 * @param  array   $bindings
 * @param  string  $boolean
 * @return $this
 */
public function havingRaw($sql, array $bindings = [], $boolean = 'and')
{
    $type = 'Raw';

    $this->havings[] = compact('type', 'sql', 'boolean');

    $this->addBinding($bindings, 'having');

    return $this;
}
```

`Grammer`中的处理方法也比较简单，主要是注意一下`wrap`和`parameter`的作用，前者用来处理列名，例如加quote，表名加统一前缀，
后者将值转换成占位符。

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile a single having clause.
 *
 * @param  array   $having
 * @return string
 */
protected function compileHaving(array $having)
{
    // If the having clause is "raw", we can just return the clause straight away
    // without doing any more processing on it. Otherwise, we will compile the
    // clause into SQL based on the components that make it up from builder.
    if ($having['type'] === 'Raw') {
        return $having['boolean'].' '.$having['sql'];
    }

    return $this->compileBasicHaving($having);
}

/**
 * Compile a basic having clause.
 *
 * @param  array   $having
 * @return string
 */
protected function compileBasicHaving($having)
{
    $column = $this->wrap($having['column']);

    $parameter = $this->parameter($having['value']);

    return $having['boolean'].' '.$column.' '.$having['operator'].' '.$parameter;
}
```