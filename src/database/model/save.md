# 新建与更新模型

`save`是模型提供的用于新建或者更新模型的方法。`save`方法内部用`$this->exists`这个属性来判断是执行更新操作还是
新建操作，所有通过模型查询构造器获取到的模型，它的`$this->exists`值都为`true`，表明该模型是数据库中已经存在的
记录，同样，刚插入到数据库的模型的`$this->exists`值也为`true`，这样就能在刚插入到数据库的模型上执行更新操作。

```php
//src/Illuminate/Database/Eloquent/Model.php

/**
 * Save the model to the database.
 *
 * @param  array  $options
 * @return bool
 */
public function save(array $options = [])
{
    $query = $this->newQueryWithoutScopes();

    // If the "saving" event returns false we'll bail out of the save and return
    // false, indicating that the save failed. This provides a chance for any
    // listeners to cancel save operations if validations fail or whatever.
    if ($this->fireModelEvent('saving') === false) {
        return false;
    }

    // If the model already exists in the database we can just update our record
    // that is already in this database using the current IDs in this "where"
    // clause to only update this model. Otherwise, we'll just insert them.
    if ($this->exists) {
        $saved = $this->isDirty() ?
                    $this->performUpdate($query) : true;
    }

    // If the model is brand new, we'll insert it into our database and set the
    // ID attribute on the model to the value of the newly inserted row's ID
    // which is typically an auto-increment value managed by the database.
    else {
        $saved = $this->performInsert($query);

        if (! $this->getConnectionName() &&
            $connection = $query->getConnection()) {
            $this->setConnection($connection->getName());
        }
    }

    // If the model is successfully saved, we need to do a few more things once
    // that is done. We will call the "saved" method here to run any actions
    // we need to happen after a model gets successfully saved right here.
    if ($saved) {
        $this->finishSave($options);
    }

    return $saved;
}
```

在执行更新操作的时候，并不是任何时候都向数据库发送更新语句，只有在模型的记录和它获取回来时的记录不同的时候，才真正更新。
`save`方法在内部调用了`isDirty`来判断模型的就来是否被修改了。

```php
//src/Illuminate/Database/Eloquent/Concerns/HasAttributes.php

/**
 * Determine if the model or given attribute(s) have been modified.
 *
 * @param  array|string|null  $attributes
 * @return bool
 */
public function isDirty($attributes = null)
{
    return $this->hasChanges(
        $this->getDirty(), is_array($attributes) ? $attributes : func_get_args()
    );
}
```

基本的原理就是将要修改的字段和原来的字段作比较，如果两者不相等，那就意味着这个字段dirty了，那就需要执行更新了。

当更新或者插入操作完成以后，它就会把新的字段值同步到original中

```php
//src/Illuminate/Database/Eloquent/Model.php

/**
 * Perform any actions that are necessary after the model is saved.
 *
 * @param  array  $options
 * @return void
 */
protected function finishSave(array $options)
{
    $this->fireModelEvent('saved', false);

    if ($this->isDirty() && ($options['touch'] ?? true)) {
        $this->touchOwners();
    }

    $this->syncOriginal();
}
```

`syncOriginal`只是将当前的所有字段复制到`$this->original`中

```php
//src/Illuminate/Database/Eloquent/Concerns/HasAttributes.php

/**
 * Sync the original attributes with the current.
 *
 * @return $this
 */
public function syncOriginal()
{
    $this->original = $this->attributes;

    return $this;
}
```