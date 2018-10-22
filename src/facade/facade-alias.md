# 别名

除了通过如`Illuminate\Supports\Facades\DB`这样的名称来使用DB Facade以外，还可以直接使用`DB`，背后的实现实际上就是
通过`class_alias`将`DB`类作为`Illuminate\Supports\Facades\DB`类的别名。

设置别名的操作是在框架启动阶段完成的。它的核心逻辑位于`src/Illuminate/Foundation/AliasLoader.php`，在框架启动的时候，
会调用`AliasLoader::getInstance()->register()`来注册别名，这段逻辑位于`src/Illuminate/Foundation/Bootstrap/RegisterFacades`。

`getInstance`实际上就是一个获取单例的方法

```php
//src/Illuminate/Foundation/AliasLoader.php

/**
 * Get or create the singleton alias loader instance.
 *
 * @param  array  $aliases
 * @return \Illuminate\Foundation\AliasLoader
 */
public static function getInstance(array $aliases = [])
{
    if (is_null(static::$instance)) {
        return static::$instance = new static($aliases);
    }

    $aliases = array_merge(static::$instance->getAliases(), $aliases);

    static::$instance->setAliases($aliases);

    return static::$instance;
}
```

获取到`AliasLoader`的实例以后，就调用它的`register`方法

```php
//src/Illuminate/Foundation/AliasLoader.php

/**
 * Register the loader on the auto-loader stack.
 *
 * @return void
 */
public function register()
{
    if (! $this->registered) {
        $this->prependToLoaderStack();

        $this->registered = true;
    }
}

/**
 * Prepend the load method to the auto-loader stack.
 *
 * @return void
 */
protected function prependToLoaderStack()
{
    spl_autoload_register([$this, 'load'], true, true);
}
```

逻辑很简单，就是注册一个`spl_autoload_register`的回调，这是PHP中用于做自动加载的函数。


```php
//src/Illuminate/Foundation/AliasLoader.php

/**
 * Load a class alias if it is registered.
 *
 * @param  string  $alias
 * @return bool|null
 */
public function load($alias)
{
    if (static::$facadeNamespace && strpos($alias, static::$facadeNamespace) === 0) {
        $this->loadFacade($alias);

        return true;
    }

    if (isset($this->aliases[$alias])) {
        return class_alias($this->aliases[$alias], $alias);
    }
}
```

这里先不考虑实时Facade的情况，那么`load`方法只是简单地用`class_alias`为一个类设置别名。