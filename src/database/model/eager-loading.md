# 关系预加载

关系预加载可以有效防止N+1查询的问题，在使用模型查询构造器时，通过`with`方法，指定要预加载的关系

```php
$books = Book::with('author')->get();
foreach ($books as $book) {
    $book->author->name;
}
```

`with`首先将要预加载的关系都存放起来

```php
//src/Illuminate/Database/Eloquent/Builder.php

/**
 * Set the relationships that should be eager loaded.
 *
 * @param  mixed  $relations
 * @return $this
 */
public function with($relations)
{
    $eagerLoad = $this->parseWithRelations(is_string($relations) ? func_get_args() : $relations);

    $this->eagerLoad = array_merge($this->eagerLoad, $eagerLoad);

    return $this;
}
```

`parseWithRelations`将传递进来的要预加载的关系名称组织成一个统一的数据结构

```php
//src/Illuminate/Database/Eloquent/Builder.php

/**
 * Parse a list of relations into individuals.
 *
 * @param  array  $relations
 * @return array
 */
protected function parseWithRelations(array $relations)
{
    $results = [];

    foreach ($relations as $name => $constraints) {
        // If the "relation" value is actually a numeric key, we can assume that no
        // constraints have been specified for the eager load and we'll just put
        // an empty Closure with the loader so that we can treat all the same.
        if (is_numeric($name)) {
            $name = $constraints;

            list($name, $constraints) = Str::contains($name, ':')
                        ? $this->createSelectWithConstraint($name)
                        : [$name, function () {
                            //
                        }];
        }

        // We need to separate out any nested includes. Which allows the developers
        // to load deep relationships using "dots" without stating each level of
        // the relationship with its own key in the array of eager load names.
        $results = $this->addNestedWiths($name, $results);

        $results[$name] = $constraints;
    }

    return $results;
}
```

这里首先忽略为预加载的关系指定约束以及嵌套预加载这两种情况，一般情况下，它把预加载关系的名称转换成

```php
['author' => function() {}]
```

这种形式的数组，然后存放在模型查询构造器中的`eagerLoad`属性中。

在获取结果后，模型查询构造器会判断，如果结果不为空，那么它就会做进一步的加载，将`eagerLoad`中指定的预加载关系
都加载进来。

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

`eagerLoadRelations`会迭代`eagerLoad`属性，依次将每个预加载关系加载进来。这里，我们依然先忽略嵌套
加载的情况，一般情况下，它就是使用`eagerLoadRelation`，将关系逐个加载进来。


```php
//src/Illuminate/Database/Eloquent/Builder.php

/**
 * Eager load the relationships for the models.
 *
 * @param  array  $models
 * @return array
 */
public function eagerLoadRelations(array $models)
{
    foreach ($this->eagerLoad as $name => $constraints) {
        // For nested eager loads we'll skip loading them here and they will be set as an
        // eager load on the query to retrieve the relation so that they will be eager
        // loaded on that query, because that is where they get hydrated as models.
        if (strpos($name, '.') === false) {
            $models = $this->eagerLoadRelation($models, $name, $constraints);
        }
    }

    return $models;
}
```

`eagerLoadRelation`中就需要用到`\Illuminate\Database\Eloquent\Relations\Relation`中的方法。

```php
//src/Illuminate/Database/Eloquent/Builder.php

/**
 * Eagerly load the relationship on a set of models.
 *
 * @param  array  $models
 * @param  string  $name
 * @param  \Closure  $constraints
 * @return array
 */
protected function eagerLoadRelation(array $models, $name, Closure $constraints)
{
    // First we will "back up" the existing where conditions on the query so we can
    // add our eager constraints. Then we will merge the wheres that were on the
    // query back to it in order that any where conditions might be specified.
    $relation = $this->getRelation($name);

    $relation->addEagerConstraints($models);

    $constraints($relation);

    // Once we have the results, we just match those back up to their parent models
    // using the relationship instance. Then we just return the finished arrays
    // of models which have been eagerly hydrated and are readied for return.
    return $relation->match(
        $relation->initRelation($models, $name),
        $relation->getEager(), $name
    );
}
```

从`eagerLoadRelation`中可以看到，它主要用到了`addEagerConstraints`，`match`，`initRelation`以及`getEager`。

下面来看看这几个方法的具体职责。首先是`addEagerConstraints`，它主要就是为即将要运行的预加载SQL语句添加约束

```php
//src/Illuminate/Database/Eloquent/Relations/BelongsTo.php

/**
 * Set the constraints for an eager load of the relation.
 *
 * @param  array  $models
 * @return void
 */
public function addEagerConstraints(array $models)
{
    // We'll grab the primary key name of the related models since it could be set to
    // a non-standard name and not "id". We will then construct the constraint for
    // our eagerly loading query so it returns the proper models from execution.
    $key = $this->related->getTable().'.'.$this->ownerKey;

    $this->query->whereIn($key, $this->getEagerModelKeys($models));
}
```

上面是`\Illuminate\Database\Eloquent\Relations\BelongsTo`关系的`addEagerConstraints`实现，它主要就是使用`IN`语句
来筛选符合对应外键值的数据。`getEagerModelKeys`就是在结果集中找到所有外键值。

接下来是`getEager`，它实际上就是运行预加载SQL语句，获取预加载结果。

```php
//src/Illuminate/Database/Eloquent/Relations/Relation.php

/**
 * Get the relationship for eager loading.
 *
 * @return \Illuminate\Database\Eloquent\Collection
 */
public function getEager()
{
    return $this->get();
}
```

`initRelation`则是用于初始化原来模型中的关系，这是因为Laravel支持默认关系模型，也就是当对应的关系不存在的时候，为这个关系指定一个默认
的模型，它在这里一次性为这些模型指定默认的关系模型。而那些对应关系模型存在的模型，就需要在`match`方法中作一次匹配，然后把默认模型覆盖掉。

```php
//src/Illuminate/Database/Eloquent/Relations/BelongsTo.php

/**
 * Match the eagerly loaded results to their parents.
 *
 * @param  array   $models
 * @param  \Illuminate\Database\Eloquent\Collection  $results
 * @param  string  $relation
 * @return array
 */
public function match(array $models, Collection $results, $relation)
{
    $foreign = $this->foreignKey;

    $owner = $this->ownerKey;

    // First we will get to build a dictionary of the child models by their primary
    // key of the relationship, then we can easily match the children back onto
    // the parents using that dictionary and the primary key of the children.
    $dictionary = [];

    foreach ($results as $result) {
        $dictionary[$result->getAttribute($owner)] = $result;
    }

    // Once we have the dictionary constructed, we can loop through all the parents
    // and match back onto their children using these keys of the dictionary and
    // the primary key of the children to map them onto the correct instances.
    foreach ($models as $model) {
        if (isset($dictionary[$model->{$foreign}])) {
            $model->setRelation($relation, $dictionary[$model->{$foreign}]);
        }
    }

    return $models;
}
```

由于之前模型对应的关系模型都是默认值，在`match`方法中就需要将一些默认值覆盖掉，换成实际的关系模型。实现很简单，就是根据各个键值作匹配。

以上就是普通预加载执行的大概流程。

## 带约束的预加载

我们可以为预加载的关系指定约束

```php
$books = Book::with(['author' => function($builder) { $builder->where('active', 1) }])->get();
```

普通预加载最终是转换成一个带有空约束的预加载。

## 嵌套预加载

预加载支持嵌套使用，例如

```php
$books = Book::with('author.profile')->get();
```

这样不但把`Book`模型的`author`关系预加载进来，还把`Author`的`profile`关系也预加载进来。

我们还能为嵌套的预加载指定约束

```php
$books = Book::with(['author.profile' => function($builder) { $builder->where('bar', 'foo') }])->get();
```

要注意的是，上面的语句所指定的约束是在`profile`上的，如果同时还要在`author`上指定约束的话，那么可以

```php
$books = Book::with([
    'author' => function($builder) {
        $builder->where('foo', 'bar');
    },
    'author.profile' => function($builder) {
        $builder->where('bar', 'foo');
    }
])->get();
```

或者是

```php
$books = Book::with([
    'author' => function($builder) {
        $builder->with(['profile' => function($builder) {
            $builder->where('bar', 'foo');
        }])->where('foo', 'bar');
    }
])->get();
```

当多几个嵌套层级的话，后面一种表达方式就不太友好。

实际上，对于像

```php
$books = Book::with('author.profile')->get();
```

这种方式的嵌套预加载，Laravel都会把它转换成

```php
$books = Book::with([
    'author' => function() {},
    'author.profile' => function() {}
])->get();
```

我们可以看一下`parseWithRelations`中调用的`addNestedWiths`实现

```php
//src/Illuminate/Database/Eloquent/Builder.php

/**
 * Parse the nested relationships in a relation.
 *
 * @param  string  $name
 * @param  array  $results
 * @return array
 */
protected function addNestedWiths($name, $results)
{
    $progress = [];

    // If the relation has already been set on the result array, we will not set it
    // again, since that would override any constraints that were already placed
    // on the relationships. We will only set the ones that are not specified.
    foreach (explode('.', $name) as $segment) {
        $progress[] = $segment;

        if (! isset($results[$last = implode('.', $progress)])) {
            $results[$last] = function () {
                //
            };
        }
    }

    return $results;
}
```

它把`foo.bar.baz`这种形式的嵌套预加载都转换成

```php
[
    'foo' => function() {},
    'foo.bar' => function() {},
    'foo.bar.baz' => function() {}
]
```

然后放到`eagerLoad`属性中。接着看一下用于获取预加载关系的`getRelation`实现

```php
//src/Illuminate/Database/Eloquent/Builder.php

/**
 * Get the relation instance for the given relation name.
 *
 * @param  string  $name
 * @return \Illuminate\Database\Eloquent\Relations\Relation
 */
public function getRelation($name)
{
    // We want to run a relationship query without any constrains so that we will
    // not have to remove these where clauses manually which gets really hacky
    // and error prone. We don't want constraints because we add eager ones.
    $relation = Relation::noConstraints(function () use ($name) {
        try {
            return $this->getModel()->{$name}();
        } catch (BadMethodCallException $e) {
            throw RelationNotFoundException::make($this->getModel(), $name);
        }
    });

    $nested = $this->relationsNestedUnder($name);

    // If there are nested relationships set on the query, we will put those onto
    // the query instances so that they can be handled after this relationship
    // is loaded. In this way they will all trickle down as they are loaded.
    if (count($nested) > 0) {
        $relation->getQuery()->with($nested);
    }

    return $relation;
}
```

首先通过模型的关系方法获得`\Illuminate\Database\Eloquent\Relations\Relation`实例，但是和一般的获取方法不同的是，
它是通过`Relation::noConstraints`，这是因为`\Illuminate\Database\Eloquent\Relations\Relation`在构造函数
中会使用`addConstraints`来添加约束，这是为了做普通加载，可是目前是做自动加载，因此就不能使用普通加载的那些约束。

接下来就要看这个预加载的关系下有多少个嵌套加载

```php
//src/Illuminate/Database/Eloquent/Builder.php

/**
 * Get the deeply nested relations for a given top-level relation.
 *
 * @param  string  $relation
 * @return array
 */
protected function relationsNestedUnder($relation)
{
    $nested = [];

    // We are basically looking for any relationships that are nested deeper than
    // the given top-level relationship. We will just check for any relations
    // that start with the given top relations and adds them to our arrays.
    foreach ($this->eagerLoad as $name => $constraints) {
        if ($this->isNestedUnder($relation, $name)) {
            $nested[substr($name, strlen($relation.'.'))] = $constraints;
        }
    }

    return $nested;
}

/**
 * Determine if the relationship is nested.
 *
 * @param  string  $relation
 * @param  string  $name
 * @return bool
 */
protected function isNestedUnder($relation, $name)
{
    return Str::contains($name, '.') && Str::startsWith($name, $relation.'.');
}
```

然后使用下一级关系的`with`方法，让下一级关系来做预加载。

## 懒惰式预加载

懒惰式预加载支持动态选择预加载的关系

```php
$books = Book::all();
if ($someCondition) {
    $books->load(['author', 'publisher']);
}
```

`\Illuminate\Database\Eloquent\Collection`提供`load`方法，用于为集合中的所有模型预加载对应关系

```php
//src/Illuminate/Database/Eloquent/Collection.php

/**
 * Load a set of relationships onto the collection.
 *
 * @param  mixed  $relations
 * @return $this
 */
public function load($relations)
{
    if ($this->isNotEmpty()) {
        if (is_string($relations)) {
            $relations = func_get_args();
        }

        $query = $this->first()->newQueryWithoutRelationships()->with($relations);

        $this->items = $query->eagerLoadRelations($this->items);
    }

    return $this;
}
```

它从模型集合中获取一个模型，调用该模型对应模型查询构造器中的`with`方法，指定要预加载的关系，最后通过模型查询构造器的
`eagerLoadRelations`，将这些关系预加载进来。它的实现原理和一般的关系预加载一样。在一般的关系预加载中，通过动态构造`with`
的参数，也能实现动态预加载。