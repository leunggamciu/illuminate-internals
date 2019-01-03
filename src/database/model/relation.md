# 模型关系

模型与模型之间可以通过数据表的外键建立关系，通过模型关系，我们可以通过一个模型，操作与该模型有关联的其他模型，例如访问关联的模型，
通过模型关系，我们可以像访问模型本身属性一样去访问与该模型有关联的模型。

在模型中，我们通过`getAttribute`来访问模型的属性

```php
//src/Illuminate/Database/Eloquent/Concerns/HasAttributes.php

/**
 * Get an attribute from the model.
 *
 * @param  string  $key
 * @return mixed
 */
public function getAttribute($key)
{
    if (! $key) {
        return;
    }

    // If the attribute exists in the attribute array or has a "get" mutator we will
    // get the attribute's value. Otherwise, we will proceed as if the developers
    // are asking for a relationship's value. This covers both types of values.
    if (array_key_exists($key, $this->attributes) ||
        $this->hasGetMutator($key)) {
        return $this->getAttributeValue($key);
    }

    // Here we will determine if the model base class itself contains this given key
    // since we don't want to treat any of those methods as relationships because
    // they are all intended as helper methods and none of these are relations.
    if (method_exists(self::class, $key)) {
        return;
    }

    return $this->getRelationValue($key);
}
```

当模型的属性不存在是，对应的名称就会被当做是关系，然后使用`getRelationValue`来获取对应的关系模型。

```php
//src/Illuminate/Database/Eloquent/Concerns/HasAttributes.php

/**
 * Get a relationship.
 *
 * @param  string  $key
 * @return mixed
 */
public function getRelationValue($key)
{
    // If the key already exists in the relationships array, it just means the
    // relationship has already been loaded, so we'll just return it out of
    // here because there is no need to query within the relations twice.
    if ($this->relationLoaded($key)) {
        return $this->relations[$key];
    }

    // If the "attribute" exists as a method on the model, we will just assume
    // it is a relationship and will load and return results from the query
    // and hydrate the relationship's value on the "relationships" array.
    if (method_exists($this, $key)) {
        return $this->getRelationshipFromMethod($key);
    }
}

/**
 * Get a relationship value from a method.
 *
 * @param  string  $method
 * @return mixed
 *
 * @throws \LogicException
 */
protected function getRelationshipFromMethod($method)
{
    $relation = $this->$method();

    if (! $relation instanceof Relation) {
        throw new LogicException(sprintf(
            '%s::%s must return a relationship instance.', static::class, $method
        ));
    }

    return tap($relation->getResults(), function ($results) use ($method) {
        $this->setRelation($method, $results);
    });
}
```

`getRelationValue`主要就是负责调用对应关系的方法，得到一个`\Illuminate\Database\Eloquent\Relations\Relation`实例，
在调用它的`getResults`方法获取这个关系对应的模型数据以后，将这些数据缓存起来，这样下一次访问的时候就不需要再访问数据库。

从这里可以看出，所有的关系都是`\Illuminate\Database\Eloquent\Relations\Relation`的实例，它提供一个`getResults`方法。

## 一对一

### 正向关系

通过模型中的`hasOne`方法，我们可以定义父模型拥有一个子模型的关系。

```php
//src/Illuminate/Database/Eloquent/Concerns/HasRelationships.php

/**
 * Define a one-to-one relationship.
 *
 * @param  string  $related
 * @param  string  $foreignKey
 * @param  string  $localKey
 * @return \Illuminate\Database\Eloquent\Relations\HasOne
 */
public function hasOne($related, $foreignKey = null, $localKey = null)
{
    $instance = $this->newRelatedInstance($related);

    $foreignKey = $foreignKey ?: $this->getForeignKey();

    $localKey = $localKey ?: $this->getKeyName();

    return $this->newHasOne($instance->newQuery(), $this, $instance->getTable().'.'.$foreignKey, $localKey);
}

/**
 * Instantiate a new HasOne relationship.
 *
 * @param  \Illuminate\Database\Eloquent\Builder  $query
 * @param  \Illuminate\Database\Eloquent\Model  $parent
 * @param  string  $foreignKey
 * @param  string  $localKey
 * @return \Illuminate\Database\Eloquent\Relations\HasOne
 */
protected function newHasOne(Builder $query, Model $parent, $foreignKey, $localKey)
{
    return new HasOne($query, $parent, $foreignKey, $localKey);
}
```

`hasOne`的作用就是构造以及返回`\Illuminate\Database\Eloquent\Relations\HasOne`实例，这个实例最终继承自
`\Illuminate\Database\Eloquent\Relations\Relation`，

下面看看`\Illuminate\Database\Eloquent\Relations\HasOne`的`getResults`方法是如何实现的


```php
//src/Illuminate/Database/Eloquent/Relations/HasOne.php

/**
 * Get the results of the relationship.
 *
 * @return mixed
 */
public function getResults()
{
    return $this->query->first() ?: $this->getDefaultFor($this->parent);
} 
```

这里只是简单地调用子模型的模型查询构造器中的`first`方法。但是我们怎么知道它拿回来的数据就是跟父模型对应的呢？

这种根据父模型中的值获取对应子模型或者根据子模型中的值获取对应父模型都可以抽象成一个已知模型和一张数据表，我们要做的就是
在查询数据表的时候，结合已知的模型，为查询过程加上对应的约束，从而获取与已知模型对应的数据。

Eloquent把这部分逻辑放到`\Illuminate\Database\Eloquent\Relations\Relation`的构造函数中

```php
//src/Illuminate/Database/Eloquent/Relations/Relation.php

/**
 * Create a new relation instance.
 *
 * @param  \Illuminate\Database\Eloquent\Builder  $query
 * @param  \Illuminate\Database\Eloquent\Model  $parent
 * @return void
 */
public function __construct(Builder $query, Model $parent)
{
    $this->query = $query;
    $this->parent = $parent;
    $this->related = $query->getModel();

    $this->addConstraints();
}

/**
 * Set the base constraints on the relation query.
 *
 * @return void
 */
abstract public function addConstraints();
```

这里的`addConstraints`就是用于为操作数据表的过程加上对应约束。由于不同的关系，添加的约束不同，所以这里是一个抽象方法，下面主要看
`\Illuminate\Database\Eloquent\Relations\HasOne`继承的`\Illuminate\Database\Eloquent\Relations\HasOneOrMany`中的具体实现

```php
//src/Illuminate/Database/Eloquent/Relations/HasOneOrMany.php

/**
 * Set the base constraints on the relation query.
 *
 * @return void
 */
public function addConstraints()
{
    if (static::$constraints) {
        $this->query->where($this->foreignKey, '=', $this->getParentKey());

        $this->query->whereNotNull($this->foreignKey);
    }
}
```

如果是一对一的关系，那么`$this->query`在这里代表的就是子模型的模型查询构造器，我们需要约束的是子模型中的外键值与父模型的主键值相等。

### 反向关系

我们把A模型 has one B模型称为正向关系，B模型 belongs to A模型称为反向关系。在一对一或者一对多中，反向关系由
`\Illuminate\Database\Eloquent\Relations\BelongsTo`负责处理。

同样，它也是一个继承自`\Illuminate\Database\Eloquent\Relations\Relation`，所以我们也是主要关注`addConstraint`以及
`getResults`方法。


```php
//src/Illuminate/Database/Eloquent/Relations/BelongsTo.php

/**
 * Set the base constraints on the relation query.
 *
 * @return void
 */
public function addConstraints()
{
    if (static::$constraints) {
        // For belongs to relationships, which are essentially the inverse of has one
        // or has many relationships, we need to actually query on the primary key
        // of the related models matching on the foreign key that's on a parent.
        $table = $this->related->getTable();

        $this->query->where($table.'.'.$this->ownerKey, '=', $this->child->{$this->foreignKey});
    }
}
```

这里的`$this->query`代表的是父模型的模型查询构造器，这里需要约束的就是父模型的主键值与子模型的外键值相等。

```php
//src/Illuminate/Database/Eloquent/Relations/BelongsTo.php

/**
 * Get the results of the relationship.
 *
 * @return mixed
 */
public function getResults()
{
    return $this->query->first() ?: $this->getDefaultFor($this->parent);
}
```

`getResults`直接执行父模型的查询构造器找到对应的父模型即可。

## 一对多

一对多和一对一实际上大部分的逻辑是一样的，所以这里就不再一一展开。

模型提供的`hasMany`用于定义模型之间一对多的关系，和一对一的同理，它也是返回一个`\Illuminate\Database\Eloquent\Relations\Relation`的
实例

```php
//src/Illuminate/Database/Eloquent/Concerns/HasRelationships.php

/**
 * Define a one-to-many relationship.
 *
 * @param  string  $related
 * @param  string  $foreignKey
 * @param  string  $localKey
 * @return \Illuminate\Database\Eloquent\Relations\HasMany
 */
public function hasMany($related, $foreignKey = null, $localKey = null)
{
    $instance = $this->newRelatedInstance($related);

    $foreignKey = $foreignKey ?: $this->getForeignKey();

    $localKey = $localKey ?: $this->getKeyName();

    return $this->newHasMany(
        $instance->newQuery(), $this, $instance->getTable().'.'.$foreignKey, $localKey
    );
}

/**
 * Instantiate a new HasMany relationship.
 *
 * @param  \Illuminate\Database\Eloquent\Builder  $query
 * @param  \Illuminate\Database\Eloquent\Model  $parent
 * @param  string  $foreignKey
 * @param  string  $localKey
 * @return \Illuminate\Database\Eloquent\Relations\HasMany
 */
protected function newHasMany(Builder $query, Model $parent, $foreignKey, $localKey)
{
    return new HasMany($query, $parent, $foreignKey, $localKey);
}
```

它的`addConstraints`的实现和`\Illuminate\Database\Eloquent\Relations\HasOne`中的一样，
因为两者都是继承自`\Illuminate\Database\Eloquent\Relations\HasOneOrMany`。

```php
//src/Illuminate/Database/Eloquent/Relations/HasOneOrMany.php

/**
 * Set the base constraints on the relation query.
 *
 * @return void
 */
public function addConstraints()
{
    if (static::$constraints) {
        $this->query->where($this->foreignKey, '=', $this->getParentKey());

        $this->query->whereNotNull($this->foreignKey);
    }
}
```

`\Illuminate\Database\Eloquent\Relations\HasMany`中的`getResults`方法很简单

```php
//src/Illuminate/Database/Eloquent/Relations/HasMany.php

/**
 * Get the results of the relationship.
 *
 * @return mixed
 */
public function getResults()
{
    return $this->query->get();
}
```

反向关系和一对一的反向关系一样。

## 多对多

多对多关系要比一对多或者一对一复杂一点

```php
//src/Illuminate/Database/Eloquent/Relations/BelongsToMany.php

/**
 * Set the base constraints on the relation query.
 *
 * @return void
 */
public function addConstraints()
{
    $this->performJoin();

    if (static::$constraints) {
        $this->addWhereConstraints();
    }
}
```

在`addConstraints`中，它首先要做的就是连接关联表和中间表

```php
//src/Illuminate/Database/Eloquent/Relations/BelongsToMany.php

/**
 * Set the join clause for the relation query.
 *
 * @param  \Illuminate\Database\Eloquent\Builder|null  $query
 * @return $this
 */
protected function performJoin($query = null)
{
    $query = $query ?: $this->query;

    // We need to join to the intermediate table on the related model's primary
    // key column with the intermediate table's foreign key for the related
    // model instance. Then we can set the "where" for the parent models.
    $baseTable = $this->related->getTable();

    $key = $baseTable.'.'.$this->relatedKey;

    $query->join($this->table, $key, '=', $this->getQualifiedRelatedPivotKeyName());

    return $this;
}
```

然后添加where约束，目的就是要筛选出与该模型有关的数据。

```php
//src/Illuminate/Database/Eloquent/Relations/BelongsToMany.php

/**
 * Set the where clause for the relation query.
 *
 * @return $this
 */
protected function addWhereConstraints()
{
    $this->query->where(
        $this->getQualifiedForeignPivotKeyName(), '=', $this->parent->{$this->parentKey}
    );

    return $this;
}
```

和一对一或者一对多的关系相比，多对多中`getResults`需要额外处理中间表的数据

```php
//src/Illuminate/Database/Eloquent/Relations/BelongsToMany.php

/**
 * Get the results of the relationship.
 *
 * @return mixed
 */
public function getResults()
{
    return $this->get();
}

/**
 * Execute the query as a "select" statement.
 *
 * @param  array  $columns
 * @return \Illuminate\Database\Eloquent\Collection
 */
public function get($columns = ['*'])
{
    // First we'll add the proper select columns onto the query so it is run with
    // the proper columns. Then, we will get the results and hydrate out pivot
    // models with the result of those columns as a separate model relation.
    $columns = $this->query->getQuery()->columns ? [] : $columns;

    $builder = $this->query->applyScopes();

    $models = $builder->addSelect(
        $this->shouldSelect($columns)
    )->getModels();

    $this->hydratePivotRelation($models);

    // If we actually found models we will also eager load any relationships that
    // have been specified as needing to be eager loaded. This will solve the
    // n + 1 query problem for the developer and also increase performance.
    if (count($models) > 0) {
        $models = $builder->eagerLoadRelations($models);
    }

    return $this->related->newCollection($models);
}
```

和一对一或者一对多不同的是，在多对多中的`getResults`/`get`方法中，需要先为模型查询构造器应用全局查询范围，这是因为
在下面使用的`getModels`方法是直接使用数据库查询构造器。

`shouldSelect`将中间表返回的字段名称都重命名成以`pivot_`开头，后面接原来的字段名。这是为了区分返回来的字段是中间表的字段
还是关联表的字段。

`hydratePivotRelation`将中间表的数据转换成pivot模型，并为关联的模型设置一个新的关系，用来访问这个pivot模型。

```php
//src/Illuminate/Database/Eloquent/Relations/BelongsToMany.php

/**
 * Hydrate the pivot table relationship on the models.
 *
 * @param  array  $models
 * @return void
 */
protected function hydratePivotRelation(array $models)
{
    // To hydrate the pivot relationship, we will just gather the pivot attributes
    // and create a new Pivot model, which is basically a dynamic model that we
    // will set the attributes, table, and connections on it so it will work.
    foreach ($models as $model) {
        $model->setRelation($this->accessor, $this->newExistingPivot(
            $this->migratePivotAttributes($model)
        ));
    }
}
```

`migratePivotAttributes`将属于中间表的数据从`$model`中筛选出来，并去掉原来`$model`中属于中间表的数据。

```php
//src/Illuminate/Database/Eloquent/Relations/BelongsToMany.php

/**
 * Get the pivot attributes from a model.
 *
 * @param  \Illuminate\Database\Eloquent\Model  $model
 * @return array
 */
protected function migratePivotAttributes(Model $model)
{
    $values = [];

    foreach ($model->getAttributes() as $key => $value) {
        // To get the pivots attributes we will just take any of the attributes which
        // begin with "pivot_" and add those to this arrays, as well as unsetting
        // them from the parent's models since they exist in a different table.
        if (strpos($key, 'pivot_') === 0) {
            $values[substr($key, 6)] = $value;

            unset($model->$key);
        }
    }

    return $values;
}
```

## 间接一对多

和多对多一样，间接一对多也需要做连表操作

```php
//src/Illuminate/Database/Eloquent/Relations/HasManyThrough.php

/**
 * Set the base constraints on the relation query.
 *
 * @return void
 */
public function addConstraints()
{
    $localValue = $this->farParent[$this->localKey];

    $this->performJoin();

    if (static::$constraints) {
        $this->query->where($this->getQualifiedFirstKeyName(), '=', $localValue);
    }
}

/**
 * Set the join clause on the query.
 *
 * @param  \Illuminate\Database\Eloquent\Builder|null  $query
 * @return void
 */
protected function performJoin(Builder $query = null)
{
    $query = $query ?: $this->query;

    $farKey = $this->getQualifiedFarKeyName();

    $query->join($this->throughParent->getTable(), $this->getQualifiedParentKeyName(), '=', $farKey);

    if ($this->throughParentSoftDeletes()) {
        $query->whereNull($this->throughParent->getQualifiedDeletedAtColumn());
    }
}
```

所以在`addConstraints`中首先要做的就是连表操作，它将远程表和中间表连接起来。这里要注意的是要过滤掉中间表中已经被软删除的数据。

```php
//src/Illuminate/Database/Eloquent/Relations/HasManyThrough.php

/**
 * Get the results of the relationship.
 *
 * @return mixed
 */
public function getResults()
{
    return $this->get();
}

/**
 * Execute the query as a "select" statement.
 *
 * @param  array  $columns
 * @return \Illuminate\Database\Eloquent\Collection
 */
public function get($columns = ['*'])
{
    // First we'll add the proper select columns onto the query so it is run with
    // the proper columns. Then, we will get the results and hydrate out pivot
    // models with the result of those columns as a separate model relation.
    $columns = $this->query->getQuery()->columns ? [] : $columns;

    $builder = $this->query->applyScopes();

    $models = $builder->addSelect(
        $this->shouldSelect($columns)
    )->getModels();

    // If we actually found models we will also eager load any relationships that
    // have been specified as needing to be eager loaded. This will solve the
    // n + 1 query problem for the developer and also increase performance.
    if (count($models) > 0) {
        $models = $builder->eagerLoadRelations($models);
    }

    return $this->related->newCollection($models);
}
```

`getResults`/`get`的实现和多对多中的差不多，除了中间表指向主表的外键以后，不需要中间表的其他数据，因此`shouldSelect`
就只选择远程表的所有字段，加上中间表中指向主表的外键字段。

```php
//src/Illuminate/Database/Eloquent/Relations/HasManyThrough.php

/**
 * Set the select clause for the relation query.
 *
 * @param  array  $columns
 * @return array
 */
protected function shouldSelect(array $columns = ['*'])
{
    if ($columns == ['*']) {
        $columns = [$this->related->getTable().'.*'];
    }

    return array_merge($columns, [$this->getQualifiedFirstKeyName()]);
}
```

和多对多有一点不一样的是，通过`getModels`拿回来的数据并没有去掉中间表的数据，因此，通过远程模型，我们可以访问中间表中那个
指向主表的外键值。

## 多态关联

### 一对一，一对多

#### 正向关系

多态实际上就是外键指向不同的表，至于具体指向哪个表，由另外一个字段指定。在Laravel中，除了有一对一或者一对多的多态关联以外，也有
多对多的多态关联。和普通的一对一或者一对多相比，多态下的SQL语句只需要添加多一个约束，就是确定外键具体指向哪个表。

具体地，`\Illuminate\Database\Eloquent\Relations\MorphOneOrMany`是继承自`\Illuminate\Database\Eloquent\Relations\HasOneOrMany`，
它比父类就是多了一个条件约束`$this->query->where($this->morphType, $this->morphClass)`。

```php
//src/Illuminate/Database/Eloquent/Relations/MorphOneOrMany.php

/**
 * Set the base constraints on the relation query.
 *
 * @return void
 */
public function addConstraints()
{
    if (static::$constraints) {
        parent::addConstraints();

        $this->query->where($this->morphType, $this->morphClass);
    }
}
```

这是多态一对一或者一对多的正向关系。

#### 反向关系

在反向关系中，我们已知包含多态字段的记录，接着就是根据具体指向的数据表，然后到该数据表中找到对应的记录。至于具体指向哪个数据表，在包含多态字段的
记录中同样也有，我们要做的就是判断哪个字段才是用来记录具体指向的数据表，默认情况下，该字段的名称为`关系方法名_type`，而外键的名称则为`关系方法名_id`。

```php
//src/Illuminate/Database/Eloquent/Concerns/HasRelationships.php

/**
 * Define a polymorphic, inverse one-to-one or many relationship.
 *
 * @param  string  $name
 * @param  string  $type
 * @param  string  $id
 * @param  string  $ownerKey
 * @return \Illuminate\Database\Eloquent\Relations\MorphTo
 */
public function morphTo($name = null, $type = null, $id = null, $ownerKey = null)
{
    // If no name is provided, we will use the backtrace to get the function name
    // since that is most likely the name of the polymorphic interface. We can
    // use that to get both the class and foreign key that will be utilized.
    $name = $name ?: $this->guessBelongsToRelation();

    list($type, $id) = $this->getMorphs(
        Str::snake($name), $type, $id
    );

    // If the type value is null it is probably safe to assume we're eager loading
    // the relationship. In this case we'll just pass in a dummy query where we
    // need to remove any eager loads that may already be defined on a model.
    return empty($class = $this->{$type})
                ? $this->morphEagerTo($name, $type, $id, $ownerKey)
                : $this->morphInstanceTo($class, $name, $type, $id, $ownerKey);
}
```

其中`$class = $this->{$type}`就是找到关联数据表对应的模型名称。

```php
//src/Illuminate/Database/Eloquent/Concerns/HasRelationships.php

/**
 * Define a polymorphic, inverse one-to-one or many relationship.
 *
 * @param  string  $target
 * @param  string  $name
 * @param  string  $type
 * @param  string  $id
 * @param  string  $ownerKey
 * @return \Illuminate\Database\Eloquent\Relations\MorphTo
 */
protected function morphInstanceTo($target, $name, $type, $id, $ownerKey)
{
    $instance = $this->newRelatedInstance(
        static::getActualClassNameForMorph($target)
    );

    return $this->newMorphTo(
        $instance->newQuery(), $this, $id, $ownerKey ?? $instance->getKeyName(), $type, $name
    );
}
```

在`morphInstanceTo`中，`$instance`就是关联模型的实例，通过它，我们就能找到具体的关联模型。

### 多对多

多态的多对多关联与普通的多对多关联很相似，实际上，它也是继承自`\Illuminate\Database\Eloquent\Relations\BelongsToMany`。和普通的多对多
关系相比，多态多对多额外多了一个约束

```php
//src/Illuminate/Database/Eloquent/Relations/MorphToMany.php

/**
 * Set the where clause for the relation query.
 *
 * @return $this
 */
protected function addWhereConstraints()
{
    parent::addWhereConstraints();

    $this->query->where($this->table.'.'.$this->morphType, $this->morphClass);

    return $this;
}
```

`$this->morphType`就是存放具体指向哪个数据表的列名，而`$this->morphClass`就是具体的数据表（模型）的名称。