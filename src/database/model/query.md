# 查询

在Eloquent中，通过模型，我们可以构造一个与该模型对应的模型查询构造器，然后使用这个模型查询构造器来构造SQL语句，另外，模型
查询构造器还提供了一些操作方法，像`get`, `create`, `update`, `delete`，这些方法执行构造好的SQL语句，并返回对应的结果。

我们可以使用形如`User::where('active', 1)->get()`来获取所有active值为1的User模型。可是，在`\Illuminate\Database\Eloquent\Model`
中，并没有提供`where`静态方法，实际上，模型中一些静态调用是通过`__callStatic`这个魔术方法来实现

```php
//src/Illuminate/Database/Eloquent/Model.php

/**
 * Handle dynamic static method calls into the method.
 *
 * @param  string  $method
 * @param  array  $parameters
 * @return mixed
 */
public static function __callStatic($method, $parameters)
{
    return (new static)->$method(...$parameters);
}
```

而`__callStatic`的实现就是实例化一个当前实例，然后调用该实例的方法。然而，模型也没有提供`where`实例方法，最终它是通过`__call`
这个魔术方法实现。

```php
//src/Illuminate/Database/Eloquent/Model.php

/**
 * Handle dynamic method calls into the model.
 *
 * @param  string  $method
 * @param  array  $parameters
 * @return mixed
 */
public function __call($method, $parameters)
{
    if (in_array($method, ['increment', 'decrement'])) {
        return $this->$method(...$parameters);
    }

    return $this->newQuery()->$method(...$parameters);
}
```

因此，之前的那个语句可以改写成`(new User)->where('active', 1)->get()`。

```php
//src/Illuminate/Database/Eloquent/Model.php

/**
 * Get a new query builder for the model's table.
 *
 * @return \Illuminate\Database\Eloquent\Builder
 */
public function newQuery()
{
    return $this->registerGlobalScopes($this->newQueryWithoutScopes());
}

/**
 * Get a new query builder that doesn't have any global scopes.
 
 * @return \Illuminate\Database\Eloquent\Builder|static
 */
public function newQueryWithoutScopes()
{
    $builder = $this->newEloquentBuilder($this->newBaseQueryBuilder());

    // Once we have the query builders, we will set the model instances so the
    // builder can easily access any information it may need from the model
    // while it is constructing and executing various queries against it.
    return $builder->setModel($this)
                ->with($this->with)
                ->withCount($this->withCount);
}
```

`newQuery`返回的是模型查询构造器，要注意的是，这个模型查询构造器是注册了全局查询范围的，还有就是要注意模型查询构造器中的
`setModel`方法

```php
//src/Illuminate/Database/Eloquent/Builder.php

/**
 * Set a model instance for the model being queried.
 *
 * @param  \Illuminate\Database\Eloquent\Model  $model
 * @return $this
 */
public function setModel(Model $model)
{
    $this->model = $model;

    $this->query->from($model->getTable());

    return $this;
}
```

在这里，它为最终生成的SQL语句指定要操作的数据表了。

```php
//src/Illuminate/Database/Eloquent/Builder.php

/**
 * Execute the query as a "select" statement.
 *
 * @param  array  $columns
 * @return \Illuminate\Database\Eloquent\Collection|static[]
 */
public function get($columns = ['*'])
{
    $builder = $this->applyScopes();

    // If we actually found models we will also eager load any relationships that
    // have been specified as needing to be eager loaded, which will solve the
    // n+1 query issue for the developers to avoid running a lot of queries.
    if (count($models = $builder->getModels($columns)) > 0) {
        $models = $builder->eagerLoadRelations($models);
    }

    return $builder->getModel()->newCollection($models);
}

/**
 * Get the hydrated models without eager loading.
 *
 * @param  array  $columns
 * @return \Illuminate\Database\Eloquent\Model[]
 */
public function getModels($columns = ['*'])
{
    return $this->model->hydrate(
        $this->query->get($columns)->all()
    )->all();
}
```

获取回来的数据只是一个个的键值数组，因此，接下来的过程就是把这些数组转换成模型，这个过程称为"hydrate"

这里看起来使用的是模型的`hydrate`方法，实际上用的是模型查询构造器的

```php
//src/Illuminate/Database/Eloquent/Builder.php

/**
 * Create a collection of models from plain arrays.
 *
 * @param  array  $items
 * @return \Illuminate\Database\Eloquent\Collection
 */
public function hydrate(array $items)
{
    $instance = $this->newModelInstance();

    return $instance->newCollection(array_map(function ($item) use ($instance) {
        return $instance->newFromBuilder($item);
    }, $items));
}
```