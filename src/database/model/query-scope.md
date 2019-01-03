# 查询范围

查询范围用来指定特定的查询条件，它分为局部查询范围和全局查询范围

## 局部查询范围

局部查询范围需要在操作模型的时候手动指定，例如`User::active()->get()`，对应的查询范围定义如下

```php
class User extends Model
{
    public function scopeActive($builder)
    {
        return $builder->where('active', 1);
    }
}
```

局部查询范围的调用最终是落到模型查询构造器的`__call`魔术方法中

```php
//src/Illuminate/Database/Eloquent/Builder.php

/**
 * Dynamically handle calls into the query instance.
 *
 * @param  string  $method
 * @param  array  $parameters
 * @return mixed
 */
public function __call($method, $parameters)
{
    if ($method === 'macro') {
        $this->localMacros[$parameters[0]] = $parameters[1];

        return;
    }

    if (isset($this->localMacros[$method])) {
        array_unshift($parameters, $this);

        return $this->localMacros[$method](...$parameters);
    }

    if (isset(static::$macros[$method])) {
        if (static::$macros[$method] instanceof Closure) {
            return call_user_func_array(static::$macros[$method]->bindTo($this, static::class), $parameters);
        }

        return call_user_func_array(static::$macros[$method], $parameters);
    }

    if (method_exists($this->model, $scope = 'scope'.ucfirst($method))) {
        return $this->callScope([$this->model, $scope], $parameters);
    }

    if (in_array($method, $this->passthru)) {
        return $this->toBase()->{$method}(...$parameters);
    }

    $this->query->{$method}(...$parameters);

    return $this;
}
```

其中有一段逻辑就是用来判断模型里面是不是存在形如`scopeXxxxx`的方法

```php
if (method_exists($this->model, $scope = 'scope'.ucfirst($method))) {
    return $this->callScope([$this->model, $scope], $parameters);
}
```

有的话就直接调用`callScope`


```php
//src/Illuminate/Database/Eloquent/Builder.php

/**
 * Apply the given scope on the current builder instance.
 *
 * @param  callable  $scope
 * @param  array  $parameters
 * @return mixed
 */
protected function callScope(callable $scope, $parameters = [])
{
    array_unshift($parameters, $this);

    $query = $this->getQuery();

    // We will keep track of how many wheres are on the query before running the
    // scope so that we can properly group the added scope constraints in the
    // query as their own isolated nested where statement and avoid issues.
    $originalWhereCount = is_null($query->wheres)
                ? 0 : count($query->wheres);

    $result = $scope(...array_values($parameters)) ?? $this;

    if (count((array) $query->wheres) > $originalWhereCount) {
        $this->addNewWheresWithinGroup($query, $originalWhereCount);
    }

    return $result;
}
```

`callScope`的作用就是调用查询范围方法，不过在这之前，它需要记录在未应用查询范围时查询构造器中where项的数量，
如果查询范围中使用了`where`，那么`addNewWheresWithinGroup`就会尝试给查询范围中和非查询范围中的where语句分别加上括号。

```php
//src/Illuminate/Database/Eloquent/Builder.php

/**
 * Nest where conditions by slicing them at the given where count.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  int  $originalWhereCount
 * @return void
 */
protected function addNewWheresWithinGroup(QueryBuilder $query, $originalWhereCount)
{
    // Here, we totally remove all of the where clauses since we are going to
    // rebuild them as nested queries by slicing the groups of wheres into
    // their own sections. This is to prevent any confusing logic order.
    $allWheres = $query->wheres;

    $query->wheres = [];

    $this->groupWhereSliceForScope(
        $query, array_slice($allWheres, 0, $originalWhereCount)
    );

    $this->groupWhereSliceForScope(
        $query, array_slice($allWheres, $originalWhereCount)
    );
}

/**
 * Slice where conditions at the given offset and add them to the query as a nested condition.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @param  array  $whereSlice
 * @return void
 */
protected function groupWhereSliceForScope(QueryBuilder $query, $whereSlice)
{
    $whereBooleans = collect($whereSlice)->pluck('boolean');

    // Here we'll check if the given subset of where clauses contains any "or"
    // booleans and in this case create a nested where expression. That way
    // we don't add any unnecessary nesting thus keeping the query clean.
    if ($whereBooleans->contains('or')) {
        $query->wheres[] = $this->createNestedWhere(
            $whereSlice, $whereBooleans->first()
        );
    } else {
        $query->wheres = array_merge($query->wheres, $whereSlice);
    }
}
```

从`groupWhereSliceForScope`可以看出，它并不是总为where语句加括号，只有在where语句中包含有or逻辑时才加。

## 全局查询范围

当应用全局查询范围时，所有通过模型的URD操作都会被自动添加查询范围。我们可以使用模型中的`addGlobalScope`方法为模型添加
全局查询范围。

```php
//src/Illuminate/Database/Eloquent/Concerns/HasGlobalScopes.php

/**
 * Register a new global scope on the model.
 *
 * @param  \Illuminate\Database\Eloquent\Scope|\Closure|string  $scope
 * @param  \Closure|null  $implementation
 * @return mixed
 *
 * @throws \InvalidArgumentException
 */
public static function addGlobalScope($scope, Closure $implementation = null)
{
    if (is_string($scope) && ! is_null($implementation)) {
        return static::$globalScopes[static::class][$scope] = $implementation;
    } elseif ($scope instanceof Closure) {
        return static::$globalScopes[static::class][spl_object_hash($scope)] = $scope;
    } elseif ($scope instanceof Scope) {
        return static::$globalScopes[static::class][get_class($scope)] = $scope;
    }

    throw new InvalidArgumentException('Global scope must be an instance of Closure or Scope.');
}
```

全局查询范围可以是一个`callable`对象，`\Closure`实例或者`\Illuminate\Database\Eloquent\Scope`实例。

所有通过模型对数据库进行的操作都是通过模型查询构造器，通过模型中的`newQuery`方法，我们可以获取模型对应的查询构造器。

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
 * Register the global scopes for this builder instance.
 *
 * @param  \Illuminate\Database\Eloquent\Builder  $builder
 * @return \Illuminate\Database\Eloquent\Builder
 */
public function registerGlobalScopes($builder)
{
    foreach ($this->getGlobalScopes() as $identifier => $scope) {
        $builder->withGlobalScope($identifier, $scope);
    }

    return $builder;
}
```

它会把模型中的全局查询范围注册到模型查询构造器中


```php
//src/Illuminate/Database/Eloquent/Builder.php

/**
 * Register a new global scope.
 *
 * @param  string  $identifier
 * @param  \Illuminate\Database\Eloquent\Scope|\Closure  $scope
 * @return $this
 */
public function withGlobalScope($identifier, $scope)
{
    $this->scopes[$identifier] = $scope;

    if (method_exists($scope, 'extend')) {
        $scope->extend($this);
    }

    return $this;
}
```

### 查询中的全局查询范围

查询操作一般是使用模型查询构造器中的`get`方法

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
```

它在第一行就首先调用`applyScope`，将模型查询构造器中的全局查询范围应用到数据库查询构造器中。

```php
//src/Illuminate/Database/Eloquent/Builder.php

/**
 * Apply the scopes to the Eloquent builder instance and return it.
 *
 * @return \Illuminate\Database\Eloquent\Builder|static
 */
public function applyScopes()
{
    if (! $this->scopes) {
        return $this;
    }

    $builder = clone $this;

    foreach ($this->scopes as $identifier => $scope) {
        if (! isset($builder->scopes[$identifier])) {
            continue;
        }

        $builder->callScope(function (Builder $builder) use ($scope) {
            // If the scope is a Closure we will just go ahead and call the scope with the
            // builder instance. The "callScope" method will properly group the clauses
            // that are added to this query so "where" clauses maintain proper logic.
            if ($scope instanceof Closure) {
                $scope($builder);
            }

            // If the scope is a scope object, we will call the apply method on this scope
            // passing in the builder and the model instance. After we run all of these
            // scopes we will return back the builder instance to the outside caller.
            if ($scope instanceof Scope) {
                $scope->apply($builder, $this->getModel());
            }
        });
    }

    return $builder;
}
```

### 更新与删除中的全局查询范围

所有通过模型进行的更新与删除操作最终都是通过模型查询构造器中的`update`和`delete`方法。

```php
//src/Illuminate/Database/Eloquent/Builder.php

/**
 * Update a record in the database.
 *
 * @param  array  $values
 * @return int
 */
public function update(array $values)
{
    return $this->toBase()->update($this->addUpdatedAtColumn($values));
}

/**
 * Delete a record from the database.
 *
 * @return mixed
 */
public function delete()
{
    if (isset($this->onDelete)) {
        return call_user_func($this->onDelete, $this);
    }

    return $this->toBase()->delete();
}
```

它们都应用了`toBase`方法，`toBase`是用来获取数据库查询构造器，然后为它应用全局查询范围。

```php
//src/Illuminate/Database/Eloquent/Builder.php

/**
 * Get a base query builder instance.
 *
 * @return \Illuminate\Database\Query\Builder
 */
public function toBase()
{
    return $this->applyScopes()->getQuery();
}
```
