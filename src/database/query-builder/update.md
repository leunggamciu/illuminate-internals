# update

查询构造器提供`update`方法用于更新数据库记录。

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Update a record in the database.
 *
 * @param  array  $values
 * @return int
 */
public function update(array $values)
{
    $sql = $this->grammar->compileUpdate($this, $values);

    return $this->connection->update($sql, $this->cleanBindings(
        $this->grammar->prepareBindingsForUpdate($this->bindings, $values)
    ));
}
```

先看看`prepareBindingsForUpate`的实现

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Prepare the bindings for an update statement.
 *
 * @param  array  $bindings
 * @param  array  $values
 * @return array
 */
public function prepareBindingsForUpdate(array $bindings, array $values)
{
    $cleanBindings = Arr::except($bindings, ['join', 'select']);

    return array_values(
        array_merge($bindings['join'], $values, Arr::flatten($cleanBindings))
    );
}
```

它主要的作用是将要更新的数据绑定插入到join类型绑定和where类型绑定之间。


```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile an update statement into SQL.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $values
 * @return string
 */
public function compileUpdate(Builder $query, $values)
{
    $table = $this->wrapTable($query->from);

    // Each one of the columns in the update statements needs to be wrapped in the
    // keyword identifiers, also a place-holder needs to be created for each of
    // the values in the list of bindings so we can make the sets statements.
    $columns = collect($values)->map(function ($value, $key) {
        return $this->wrap($key).' = '.$this->parameter($value);
    })->implode(', ');

    // If the query has any "join" clauses, we will setup the joins on the builder
    // and compile them so we can attach them to this update, as update queries
    // can get join statements to attach to other tables when they're needed.
    $joins = '';

    if (isset($query->joins)) {
        $joins = ' '.$this->compileJoins($query, $query->joins);
    }

    // Of course, update queries may also be constrained by where clauses so we'll
    // need to compile the where clauses and attach it to the query so only the
    // intended records are updated by the SQL statements we generate to run.
    $wheres = $this->compileWheres($query);

    return trim("update {$table}{$joins} set $columns $wheres");
}
```

`compileUpdate`的实现很简单，它分别生成`UPDATE`语句的各个部分，然后再将它们拼接在一起。
