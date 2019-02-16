# eloquent模型

在任务类中，当我们注入eloquent模型或者模型集后，这个任务实例在序列化时候，会把这两者转换成
`Illuminate\Contracts\Database\ModelIdentifier`实例，然后在worker端再把这个实例转换为对应的
eloquent模型或者模型集。

这样做主要的原因是eloquent模型中的一些属性有可能包含`Closure`类型的数据，譬如`$with`这个属性，它是用来指定
预载入关系的，和通过`with`方法指定预载入不同的是，凡是通过该模型构建的查询都会预载入`$with`中指定的关系。
拥有这种类型属性的实例是没有办法被`serialize`函数序列化的，也就是说，eloquent模型是不能被`serialize`的。
因此，在任务实例被`serialize`之前，将所有eloquent属性转换为一个能被`serialize`的实例，在`unserialize`的
时候再把它转换回来。

实现这一特性最主要的就是`Illuminate\Queue\SerializesModels`和`Illuminate\Queue\SerializesAndRestoreModelIdentifiers`
这两个特征。在artisan生成的任务类中已经导入了SerializesModels特征。

```php
//src/Illuminate/Queue/SerializesModels.php

trait SerializesModels
{
    use SerializesAndRestoresModelIdentifiers;

    /**
     * Prepare the instance for serialization.
     *
     * @return array
     */
    public function __sleep()
    {
        $properties = (new ReflectionClass($this))->getProperties();

        foreach ($properties as $property) {
            $property->setValue($this, $this->getSerializedPropertyValue(
                $this->getPropertyValue($property)
            ));
        }

        return array_values(array_filter(array_map(function ($p) {
            return $p->isStatic() ? null : $p->getName();
        }, $properties)));
    }

    /**
     * Restore the model after serialization.
     *
     * @return void
     */
    public function __wakeup()
    {
        foreach ((new ReflectionClass($this))->getProperties() as $property) {
            if ($property->isStatic()) {
                continue;
            }

            $property->setValue($this, $this->getRestoredPropertyValue(
                $this->getPropertyValue($property)
            ));
        }
    }

    /**
     * Get the property value for the given property.
     *
     * @param  \ReflectionProperty  $property
     * @return mixed
     */
    protected function getPropertyValue(ReflectionProperty $property)
    {
        $property->setAccessible(true);

        return $property->getValue($this);
    }
}
```

它主要就是实现了`__sleep`和`__wakeup`这两个在序列化和反序列化中用到的魔术方法。两者都利用了反射，重新设置一遍
实例中所有属性值。值得注意的是，这个特性只适用于实例属性，不适用于静态属性。

```php
//src/Illuminate/Queue/SerializesAndRestoresModelIdentifiers.php

trait SerializesAndRestoresModelIdentifiers
{
    /**
     * Get the property value prepared for serialization.
     *
     * @param  mixed  $value
     * @return mixed
     */
    protected function getSerializedPropertyValue($value)
    {
        if ($value instanceof QueueableCollection) {
            return new ModelIdentifier(
                $value->getQueueableClass(),
                $value->getQueueableIds(),
                $value->getQueueableRelations(),
                $value->getQueueableConnection()
            );
        }

        if ($value instanceof QueueableEntity) {
            return new ModelIdentifier(
                get_class($value),
                $value->getQueueableId(),
                $value->getQueueableRelations(),
                $value->getQueueableConnection()
            );
        }

        return $value;
    }

    /**
     * Get the restored property value after deserialization.
     *
     * @param  mixed  $value
     * @return mixed
     */
    protected function getRestoredPropertyValue($value)
    {
        if (! $value instanceof ModelIdentifier) {
            return $value;
        }

        return is_array($value->id)
                ? $this->restoreCollection($value)
                : $this->restoreModel($value);
    }

    /**
     * Restore a queueable collection instance.
     *
     * @param  \Illuminate\Contracts\Database\ModelIdentifier  $value
     * @return \Illuminate\Database\Eloquent\Collection
     */
    protected function restoreCollection($value)
    {
        if (! $value->class || count($value->id) === 0) {
            return new EloquentCollection;
        }

        return $this->getQueryForModelRestoration(
            (new $value->class)->setConnection($value->connection), $value->id
        )->useWritePdo()->get();
    }

    /**
     * Restore the model from the model identifier instance.
     *
     * @param  \Illuminate\Contracts\Database\ModelIdentifier  $value
     * @return \Illuminate\Database\Eloquent\Model
     */
    public function restoreModel($value)
    {
        return $this->getQueryForModelRestoration(
            (new $value->class)->setConnection($value->connection), $value->id
        )->useWritePdo()->firstOrFail()->load($value->relations ?? []);
    }

    /**
     * Get the query for model restoration.
     *
     * @param  \Illuminate\Database\Eloquent\Model  $model
     * @param  array|int  $ids
     * @return \Illuminate\Database\Eloquent\Builder
     */
    protected function getQueryForModelRestoration($model, $ids)
    {
        return $model->newQueryForRestoration($ids);
    }
}
```