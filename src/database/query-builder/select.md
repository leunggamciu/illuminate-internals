# select

查询构造器提供`select`方法用于指定要操作的列

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Set the columns to be selected.
 *
 * @param  array|mixed  $columns
 * @return $this
 */
public function select($columns = ['*'])
{
    $this->columns = is_array($columns) ? $columns : func_get_args();

    return $this;
}
```

它将所有列以数组的形式存放到`$this->columns`中。`Grammer`中使用`compileColumn`将这些列转换成SQL语句

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile the "select *" portion of the query.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $columns
 * @return string|null
 */
protected function compileColumns(Builder $query, $columns)
{
    // If the query is actually performing an aggregating select, we will let that
    // compiler handle the building of the select clauses, as it will need some
    // more syntax that is best handled by that function to keep things neat.
    if (! is_null($query->aggregate)) {
        return;
    }

    $select = $query->distinct ? 'select distinct ' : 'select ';

    return $select.$this->columnize($columns);
}
```

这里比较复杂的就是`columnize`方法。当我们指定列的时候，可以只指定列的名称，例如`select(['foo'])`，可以对列做重命名，
例如`select(['foo as f'])`，还可以在列的前面指定表名，例如`select(['t.foo as f'])`。这些SQL语句中包含三种不同
类型的标识符：表名，列名，列的别名，我们需要把这些标识符quote起来，避免它们使用了特殊字符或者数据库中的保留字。

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Convert an array of column names into a delimited string.
 *
 * @param  array   $columns
 * @return string
 */
public function columnize(array $columns)
{
    return implode(', ', array_map([$this, 'wrap'], $columns));
}
```

`columnize`将`$columns`数组中的每个元素用`$this->wrap`处理完以后，再使用`, `连接起来。

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Wrap a value in keyword identifiers.
 *
 * @param  \Illuminate\Database\Query\Expression|string  $value
 * @param  bool    $prefixAlias
 * @return string
 */
public function wrap($value, $prefixAlias = false)
{
    if ($this->isExpression($value)) {
        return $this->getValue($value);
    }

    // If the value being wrapped has a column alias we will need to separate out
    // the pieces so we can wrap each of the segments of the expression on its
    // own, and then join these both back together using the "as" connector.
    if (strpos(strtolower($value), ' as ') !== false) {
        return $this->wrapAliasedValue($value, $prefixAlias);
    }

    return $this->wrapSegments(explode('.', $value));
}
```

这里先不考虑第一个条件分支。`wrap`先处理别名，它调用`$this->wrapAliasedValue`，将列名和别名通过`preg_split`分割开
以后分别作quote，最后再用` as `连接起来。

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Wrap a value that has an alias.
 *
 * @param  string  $value
 * @param  bool  $prefixAlias
 * @return string
 */
protected function wrapAliasedValue($value, $prefixAlias = false)
{
    $segments = preg_split('/\s+as\s+/i', $value);

    // If we are wrapping a table we need to prefix the alias with the table prefix
    // as well in order to generate proper syntax. If this is a column of course
    // no prefix is necessary. The condition will be true when from wrapTable.
    if ($prefixAlias) {
        $segments[1] = $this->tablePrefix.$segments[1];
    }

    return $this->wrap(
        $segments[0]).' as '.$this->wrapValue($segments[1]
    );
}
```

其中`$segments[0]`在这里就是列名，`$segments[1]`是别名，首先来看处理别名的情况

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Wrap a single string in keyword identifiers.
 *
 * @param  string  $value
 * @return string
 */
protected function wrapValue($value)
{
    if ($value !== '*') {
        return '"'.str_replace('"', '""', $value).'"';
    }

    return $value;
}
```

它直接在标识符的两边用`"`quote起来，还把标识符中的出现的`"`字符作转义。在MySQL中，quote使用`` ` ``，而不是`"`。所以在
`MySQLGrammer`类中，对`wrapValue`作了重写

```php
//src/Illuminate/Database/Query/Grammers/MySQLGrammer.php

/**
 * Wrap a single string in keyword identifiers.
 *
 * @param  string  $value
 * @return string
 */
protected function wrapValue($value)
{
    if ($value === '*') {
        return $value;
    }

    // If the given value is a JSON selector we will wrap it differently than a
    // traditional value. We will need to split this path and wrap each part
    // wrapped, etc. Otherwise, we will simply wrap the value as a string.
    if ($this->isJsonSelector($value)) {
        return $this->wrapJsonSelector($value);
    }

    return '`'.str_replace('`', '``', $value).'`';
}
```

然后再来看处理列名，处理列名依然用了原来的`wrap`方法，不过这一次是不带别名，它将表名和列名分割成一个数组，
然后调用`wrapSegments`方法

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Wrap the given value segments.
 *
 * @param  array  $segments
 * @return string
 */
protected function wrapSegments($segments)
{
    return collect($segments)->map(function ($segment, $key) use ($segments) {
        return $key == 0 && count($segments) > 1
                        ? $this->wrapTable($segment)
                        : $this->wrapValue($segment);
    })->implode('.');
}
```

`wrapSegments`将表名用`$this->wrapTable`处理完，再将列名用`$this->wrapValue`处理完以后，用`.`重新连接起来。

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Wrap a table in keyword identifiers.
 *
 * @param  \Illuminate\Database\Query\Expression|string  $table
 * @return string
 */
public function wrapTable($table)
{
    if (! $this->isExpression($table)) {
        return $this->wrap($this->tablePrefix.$table, true);
    }

    return $this->getValue($table);
}
```

`wrapTable`主要的作用就是为表名加上统一的表前缀后，再作quote。

`select(['t.foo as f'])`最终生成出来的SQL语句就是`SELECT "t"."foo" as "f"`。