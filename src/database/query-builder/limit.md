# limit

这个太难了，讲不下去~

```php
//src/Illuminate/Database/Query/Builder.php

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
```

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile the "limit" portions of the query.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  int  $limit
 * @return string
 */
protected function compileLimit(Builder $query, $limit)
{
    return 'limit '.(int) $limit;
}
```