# delete

查询构造器提供`delete`方法用于删除数据库记录

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Delete a record from the database.
 *
 * @param  mixed  $id
 * @return int
 */
public function delete($id = null)
{
    // If an ID is passed to the method, we will set the where clause to check the
    // ID to let developers to simply and quickly remove a single row from this
    // database without manually specifying the "where" clauses on the query.
    if (! is_null($id)) {
        $this->where($this->from.'.id', '=', $id);
    }

    return $this->connection->delete(
        $this->grammar->compileDelete($this), $this->cleanBindings(
            $this->grammar->prepareBindingsForDelete($this->bindings)
        )
    );
}
```

`delete`接受一个可选的`$id`参数用于指定要删除的行，这样就不需要通过`where`来指定。

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile a delete statement into SQL.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @return string
 */
public function compileDelete(Builder $query)
{
    $wheres = is_array($query->wheres) ? $this->compileWheres($query) : '';

    return trim("delete from {$this->wrapTable($query->from)} $wheres");
}

/**
 * Prepare the bindings for a delete statement.
 *
 * @param  array  $bindings
 * @return array
 */
public function prepareBindingsForDelete(array $bindings)
{
    return Arr::flatten($bindings);
}
```