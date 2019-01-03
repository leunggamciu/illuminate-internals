# 删除

## 普通删除

Eloquent为模型提供多种删除方式，例如在已知模型的情况下，可以调用模型的`delete`方法；或者已知主键的情况下，调用模型的`destroy`方法；
又或者通过模型查询构造器来删除。

首先是模型本身提供的`delete`方法

```php
//src/Illuminate/Database/Eloquent/Model.php

/**
 * Delete the model from the database.
 *
 * @return bool|null
 *
 * @throws \Exception
 */
public function delete()
{
    if (is_null($this->getKeyName())) {
        throw new Exception('No primary key defined on model.');
    }

    // If the model doesn't exist, there is nothing to delete so we'll just return
    // immediately and not do anything else. Otherwise, we will continue with a
    // deletion process on the model, firing the proper events, and so forth.
    if (! $this->exists) {
        return;
    }

    if ($this->fireModelEvent('deleting') === false) {
        return false;
    }

    // Here, we'll touch the owning models, verifying these timestamps get updated
    // for the models. This will allow any caching to get broken on the parents
    // by the timestamp. Then we will go ahead and delete the model instance.
    $this->touchOwners();

    $this->performDeleteOnModel();

    // Once the model has been deleted, we will fire off the deleted event so that
    // the developers may hook into post-delete operations. We will then return
    // a boolean true as the delete is presumably successful on the database.
    $this->fireModelEvent('deleted', false);

    return true;
}
```

重点就是`touchOwners`和`performDeleteOnModel`两个调用，我们主要关注后者

```php
//src/Illuminate/Database/Eloquent/Model.php

/**
 * Perform the actual delete query on this model instance.
 *
 * @return void
 */
protected function performDeleteOnModel()
{
    $this->setKeysForSaveQuery($this->newQueryWithoutScopes())->delete();

    $this->exists = false;
}
```

其中`$this->setKeysForSaveQuery($this->newQueryWithoutScopes())`这一段就是用于构建一个模型查询构造器，并加上
where约束

```php
//src/Illuminate/Database/Eloquent/Model.php

/**
 * Set the keys for a save update query.
 *
 * @param  \Illuminate\Database\Eloquent\Builder  $query
 * @return \Illuminate\Database\Eloquent\Builder
 */
protected function setKeysForSaveQuery(Builder $query)
{
    $query->where($this->getKeyName(), '=', $this->getKeyForSaveQuery());

    return $query;
}
```

最后执行模型查询构造器的`delete`方法，完成删除。

然后是可以根据主键删除模型的`destory`方法

```php
//src/Illuminate/Database/Eloquent/Model.php

/**
 * Destroy the models for the given IDs.
 *
 * @param  array|int  $ids
 * @return int
 */
public static function destroy($ids)
{
    // We'll initialize a count here so we will return the total number of deletes
    // for the operation. The developers can then check this number as a boolean
    // type value or get this total count of records deleted for logging, etc.
    $count = 0;

    $ids = is_array($ids) ? $ids : func_get_args();

    // We will actually pull the models from the database table and call delete on
    // each of them individually so that their events get fired properly with a
    // correct set of attributes in case the developers wants to check these.
    $key = ($instance = new static)->getKeyName();

    foreach ($instance->whereIn($key, $ids)->get() as $model) {
        if ($model->delete()) {
            $count++;
        }
    }

    return $count;
}
```

它的实现很简单，就是根据主键找到所有模型，然后调用模型的`delete`方法。

而直接通过模型查询构造器来删除的过程就不再叙述了，它的过程和查询的逻辑差不多。

## 软删除

Eloquent支持通过修改记录中的`deleted_at`列，实现软删除。在使用软删除前，我们需要先把`\Illuminate\Database\Eloquent\SoftDeletes`这个
特征导入到模型中。

我们重点关注`bootSoftDeletes`和`performDeleteOnModel`这两个方法

```php
//src/Illuminate/Database/Eloquent/SoftDeletes.php

/**
 * Boot the soft deleting trait for a model.
 *
 * @return void
 */
public static function bootSoftDeletes()
{
    static::addGlobalScope(new SoftDeletingScope);
}
```

`bootSoftDeletes`是一个在模型启动时候运行的方法，它的原理先不说，总之它是这个特征里面最先运行的方法。它为模型添加了一个全局查询范围。

```php
//src/Illuminate/Database/Eloquent/SoftDeletes.php

/**
 * Perform the actual delete query on this model instance.
 *
 * @return mixed
 */
protected function performDeleteOnModel()
{
    if ($this->forceDeleting) {
        $this->exists = false;

        return $this->newQueryWithoutScopes()->where($this->getKeyName(), $this->getKey())->forceDelete();
    }

    return $this->runSoftDelete();
}
```

而`performDeleteOnModel`则覆盖了模型本身的`performDeleteOnModel`，在非强制删除的情况下，它调用`runSoftDelete`，来具体执行
软删除

```php
//src/Illuminate/Database/Eloquent/SoftDeletes.php

/**
 * Perform the actual delete query on this model instance.
 *
 * @return void
 */
protected function runSoftDelete()
{
    $query = $this->newQueryWithoutScopes()->where($this->getKeyName(), $this->getKey());

    $time = $this->freshTimestamp();

    $columns = [$this->getDeletedAtColumn() => $this->fromDateTime($time)];

    $this->{$this->getDeletedAtColumn()} = $time;

    if ($this->timestamps && ! is_null($this->getUpdatedAtColumn())) {
        $this->{$this->getUpdatedAtColumn()} = $time;

        $columns[$this->getUpdatedAtColumn()] = $this->fromDateTime($time);
    }

    $query->update($columns);
}
```

`runSoftDelete`就是使用当前的时间来更新模型中的用来标记软删除的字段。