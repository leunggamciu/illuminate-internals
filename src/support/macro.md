# Macro

macro是一种为类/实例动态添加方法的技术。在Laravel的核心中，很多地方都使用了macro，譬如在模型的软删除中，当一个模型引入
了`\Illuminate\Database\Eloquent\SoftDeletes`这个特征后，它就会给模型查询构造器添加一些软删除相关的方法，像用于
从软删除中恢复模型的`restore`，囊括软删除模型的`withTrashed`，排除软删除模型的`withoutTrashed`，只考虑软删除模型的
`onlyTrashed`。

它的实现很简单，主要就是使用了`__call`和`__callStatic`这两个魔术方法。它本身是一个特征，所以当一个类实现了自己的`__call`
或者`__callStatic`时，当引入这个特征时，就要考虑冲突的问题。

```php
//src/Illuminate/Support/Traits/Macroable.php

/**
 * Register a custom macro.
 *
 * @param  string $name
 * @param  object|callable  $macro
 *
 * @return void
 */
public static function macro($name, $macro)
{
    static::$macros[$name] = $macro;
}

/**
 * Checks if macro is registered.
 *
 * @param  string  $name
 * @return bool
 */
public static function hasMacro($name)
{
    return isset(static::$macros[$name]);
}
```

`macro`和`hasMacro`都很简单，就是用来注册macro以及判断macro是否存在

```php
//src/Illuminate/Support/Traits/Macroable.php

/**
 * Dynamically handle calls to the class.
 *
 * @param  string  $method
 * @param  array   $parameters
 * @return mixed
 *
 * @throws \BadMethodCallException
 */
public static function __callStatic($method, $parameters)
{
    if (! static::hasMacro($method)) {
        throw new BadMethodCallException(sprintf(
            'Method %s::%s does not exist.', static::class, $method
        ));
    }

    if (static::$macros[$method] instanceof Closure) {
        return call_user_func_array(Closure::bind(static::$macros[$method], null, static::class), $parameters);
    }

    return call_user_func_array(static::$macros[$method], $parameters);
}
```

`__callStatic`用来处理静态的macro调用，这里要注意的是，当一个macro是`\Closure`实例时，把当前类的作用域绑定到这个实例中，否则会出现一种
情况就是在这个macro方法中，不能像普通的类静态方法一样，访问类的私有或者受保护属性和方法。

```php
class Foo
{
    use Macroable;

    private static function bar()
    {

    }
}

Foo::macro('baz', function() {
    return static::bar();
});

//如果没有绑定作用域，那么就会出错
Foo::baz();
```

```php
//src/Illuminate/Support/Traits/Macroable.php

/**
 * Dynamically handle calls to the class.
 *
 * @param  string  $method
 * @param  array   $parameters
 * @return mixed
 *
 * @throws \BadMethodCallException
 */
public function __call($method, $parameters)
{
    if (! static::hasMacro($method)) {
        throw new BadMethodCallException(sprintf(
            'Method %s::%s does not exist.', static::class, $method
        ));
    }

    $macro = static::$macros[$method];

    if ($macro instanceof Closure) {
        return call_user_func_array($macro->bindTo($this, static::class), $parameters);
    }

    return call_user_func_array($macro, $parameters);
}
```

`__call`和`__callStatic`类似，但是它除了要绑定类的作用域，还要当前实例绑定到`\Closure`的`this`，这样，在closure中
使用的`$this`指向的就是当前实例。

除此以外，它还支持从其他类实例中混入它们的公共或者受保护方法

```php
class Foo
{
    use Macroable;

}

class Bar
{
    public function baz()
    {
        return function() {

        };
    }
}

Foo::mixin(new Bar);

Foo::baz();
```

要注意的是混入方法的返回值要是`callable`类型的。

```php
//src/Illuminate/Support/Traits/Macroable.php

/**
 * Mix another object into the class.
 *
 * @param  object  $mixin
 * @return void
 * @throws \ReflectionException
 */
public static function mixin($mixin)
{
    $methods = (new ReflectionClass($mixin))->getMethods(
        ReflectionMethod::IS_PUBLIC | ReflectionMethod::IS_PROTECTED
    );

    foreach ($methods as $method) {
        $method->setAccessible(true);

        static::macro($method->name, $method->invoke($mixin));
    }
}
```

它利用了反射机制，把要混入的实例中的公共和受保护方法都拿出来，执行一遍，然后注册到macro中。