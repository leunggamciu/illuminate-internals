# offset

`offset`用于指定从第几行开始取数据，在MySQL中，`LIMIT 1, 2`相当于`LIMIT 2 OFFSET 1`。

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Set the "offset" value of the query.
 *
 * @param  int  $value
 * @return $this
 */
public function offset($value)
{
    $property = $this->unions ? 'unionOffset' : 'offset';

    $this->$property = max(0, $value);

    return $this;
}
```

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile the "offset" portions of the query.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  int  $offset
 * @return string
 */
protected function compileOffset(Builder $query, $offset)
{
    return 'offset '.(int) $offset;
}
```