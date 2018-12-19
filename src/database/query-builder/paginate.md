# paginate

查询构造器提供`paginate`方法用于对查询结果分页

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Paginate the given query into a simple paginator.
 *
 * @param  int  $perPage
 * @param  array  $columns
 * @param  string  $pageName
 * @param  int|null  $page
 * @return \Illuminate\Contracts\Pagination\LengthAwarePaginator
 */
public function paginate($perPage = 15, $columns = ['*'], $pageName = 'page', $page = null)
{
    $page = $page ?: Paginator::resolveCurrentPage($pageName);

    $total = $this->getCountForPagination($columns);

    $results = $total ? $this->forPage($page, $perPage)->get($columns) : collect();

    return $this->paginator($results, $total, $perPage, $page, [
        'path' => Paginator::resolveCurrentPath(),
        'pageName' => $pageName,
    ]);
}
```

首先它调用`Paginator::resolveCurrentPage`来获得当前请求的页数，实际上它内部的实现就是从容器中拿到本次请求的request对象，
然后获取它里面page字段值。接着调用`getCountForPagination`统计本次查询结果的总行数，最后就是根据总行数，每页数量和当前请求第几页
来算出应该从哪里开始取值以及取多少值。

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Set the limit and offset for a given page.
 *
 * @param  int  $page
 * @param  int  $perPage
 * @return \Illuminate\Database\Query\Builder|static
 */
public function forPage($page, $perPage = 15)
{
    return $this->skip(($page - 1) * $perPage)->take($perPage);
}

/**
 * Alias to set the "offset" value of the query.
 *
 * @param  int  $value
 * @return \Illuminate\Database\Query\Builder|static
 */
public function skip($value)
{
    return $this->offset($value);
}

/**
 * Alias to set the "limit" value of the query.
 *
 * @param  int  $value
 * @return \Illuminate\Database\Query\Builder|static
 */
public function take($value)
{
    return $this->limit($value);
}
```

`forPage`实际上就是使用了`offset`和`limit`方法来设置当前查询构造器。