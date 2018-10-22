# Facade调用

所有的Facade都必须继承`Illuminate\Support\Facades\Facade`，这个类中最核心的方法就是`__callStatic`。

```php
//src/Illuminate/Supports/Facade.php

/**
 * Handle dynamic, static calls to the object.
 *
 * @param  string  $method
 * @param  array   $args
 * @return mixed
 *
 * @throws \RuntimeException
 */
public static function __callStatic($method, $args)
{
    $instance = static::getFacadeRoot();

    if (! $instance) {
        throw new RuntimeException('A facade root has not been set.');
    }

    return $instance->$method(...$args);
}
```

它将静态方法的调用都委托到`$instance`上，实际上`$instance`就是服务容器中的服务实例。

```php
//src/Illuminate/Supports/Facade.php

/**
 * Get the root object behind the facade.
 *
 * @return mixed
 */
public static function getFacadeRoot()
{
    return static::resolveFacadeInstance(static::getFacadeAccessor());
}

/**
 * Get the registered name of the component.
 *
 * @return string
 *
 * @throws \RuntimeException
 */
protected static function getFacadeAccessor()
{
    throw new RuntimeException('Facade does not implement getFacadeAccessor method.');
}

/**
 * Resolve the facade root instance from the container.
 *
 * @param  string|object  $name
 * @return mixed
 */
protected static function resolveFacadeInstance($name)
{
    if (is_object($name)) {
        return $name;
    }

    if (isset(static::$resolvedInstance[$name])) {
        return static::$resolvedInstance[$name];
    }

    return static::$resolvedInstance[$name] = static::$app[$name];
}
```

这里最核心的方法就是`resolveFacadeInstance`，实际上它就是通过`static::$app[$name]`从服务容器中获取服务实例。