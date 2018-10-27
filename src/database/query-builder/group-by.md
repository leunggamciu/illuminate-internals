# group by

查询构造器提供`groupBy`方法用于构造SQL中的group by语句，它的实现如下

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Add a "group by" clause to the query.
 *
 * @param  array  ...$groups
 * @return $this
 */
public function groupBy(...$groups)
{
    foreach ($groups as $group) {
        $this->groups = array_merge(
            (array) $this->groups,
            Arr::wrap($group)
        );
    }

    return $this;
}
```

它将所有group by的列名存到`$this->groups`中，然后由`Grammer`类中的`compileGroups`方法来转换成SQL语句。

```php
//src/Illuminate/Database/Query/Grammers/Grammer.php

/**
 * Compile the "group by" portions of the query.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $groups
 * @return string
 */
protected function compileGroups(Builder $query, $groups)
{
    return 'group by '.$this->columnize($groups);
}
```

`columnize`的作用在前面已经将过，主要是对标识符作quote，为表名加上统一前缀，然后将处理好的列用逗号连接起来。