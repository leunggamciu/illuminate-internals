# 实时Facade

实时Facade免去了要写一个Facade类的过程，只需要在导入的名称前面加上`Facades`就可以了，假如你在服务容器中有一个名为
`App\Contracts\Publisher`的绑定，那么你只需要导入`Facades\App\Contracts\Publisher`，这样，`Publisher`就
成了一个Facade。

在`AliasLoader`的`load`方法里有这样一段逻辑

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

关注的重点是第一个分支，进入这个分支的条件就是，导入的名称是以`Facades`开头的。

```php
//src/Illuminate/Foundation/AliasLoader.php

/**
 * Load a real-time facade for the given alias.
 *
 * @param  string  $alias
 * @return void
 */
protected function loadFacade($alias)
{
    require $this->ensureFacadeExists($alias);
}

/**
 * Ensure that the given alias has an existing real-time facade class.
 *
 * @param  string  $alias
 * @return string
 */
protected function ensureFacadeExists($alias)
{
    if (file_exists($path = storage_path('framework/cache/facade-'.sha1($alias).'.php'))) {
        return $path;
    }

    file_put_contents($path, $this->formatFacadeStub(
        $alias, file_get_contents(__DIR__.'/stubs/facade.stub')
    ));

    return $path;
}

/**
 * Format the facade stub with the proper namespace and class.
 *
 * @param  string  $alias
 * @param  string  $stub
 * @return string
 */
protected function formatFacadeStub($alias, $stub)
{
    $replacements = [
        str_replace('/', '\\', dirname(str_replace('\\', '/', $alias))),
        class_basename($alias),
        substr($alias, strlen(static::$facadeNamespace)),
    ];

    return str_replace(
        ['DummyNamespace', 'DummyClass', 'DummyTarget'], $replacements, $stub
    );
}
```

从上面三个方法可以看出来，实时Facade的实现就是通过分析导入的Facade的名称，将它转化成Facade类定义文件，然后缓存起来。