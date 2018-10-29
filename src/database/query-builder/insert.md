# insert

查询构造器提供`insert`用于构建`INSERT`语句，在调用`compileInsert`构造SQL语句之前，需要对参数做一些操作，例如
`insert`支持插入单条和多条记录，但是最终都是转换成以多条的形式插入，另外就是在插入多条记录的时候，要保证每条记录
的顺序是一样的，因为最终转换成的SQL语句是类似于`INSERT INTO tb (c1, c2) VALUES (?, ?), (?, ?)`，顺序不一样
就会导致插入的数据错乱。

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Insert a new record into the database.
 *
 * @param  array  $values
 * @return bool
 */
public function insert(array $values)
{
    // Since every insert gets treated like a batch insert, we will make sure the
    // bindings are structured in a way that is convenient when building these
    // inserts statements by verifying these elements are actually an array.
    if (empty($values)) {
        return true;
    }

    if (! is_array(reset($values))) {
        $values = [$values];
    }

    // Here, we will sort the insert keys for every record so that each insert is
    // in the same order for the record. We need to make sure this is the case
    // so there are not any errors or problems when inserting these records.
    else {
        foreach ($values as $key => $value) {
            ksort($value);

            $values[$key] = $value;
        }
    }

    // Finally, we will run this query against the database connection and return
    // the results. We will need to also flatten these bindings before running
    // the query so they are all in one huge, flattened array for execution.
    return $this->connection->insert(
        $this->grammar->compileInsert($this, $values),
        $this->cleanBindings(Arr::flatten($values, 1))
    );
}
```



```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile an insert statement into SQL.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $values
 * @return string
 */
public function compileInsert(Builder $query, array $values)
{
    // Essentially we will force every insert to be treated as a batch insert which
    // simply makes creating the SQL easier for us since we can utilize the same
    // basic routine regardless of an amount of records given to us to insert.
    $table = $this->wrapTable($query->from);

    if (! is_array(reset($values))) {
        $values = [$values];
    }

    $columns = $this->columnize(array_keys(reset($values)));

    // We need to build a list of parameter place-holders of values that are bound
    // to the query. Each insert should have the exact same amount of parameter
    // bindings so we will loop through the record and parameterize them all.
    $parameters = collect($values)->map(function ($record) {
        return '('.$this->parameterize($record).')';
    })->implode(', ');

    return "insert into $table ($columns) values $parameters";
}
```