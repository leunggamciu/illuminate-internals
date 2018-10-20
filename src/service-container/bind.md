# 服务绑定

## 绑定解析函数

`bind`方法是用来绑定服务的主要方法，它有三个参数：`$abstract`，`$concrete`和`$shared`，其中`$abstract`
是`string`，它代表的是服务在容器中的唯一名称，它可以是任意的字符串，也可以是接口或者类名称（字符串形式）；`$concret`
有三种类型的值：`\Closure`，`string`以及`null`，当`$concrete`为`\Closure`时，实际上它就是一个服务解析函数，当
其为`string`时，它是具体类的名称，而当其为`null`时，实际上就相当于`bind($abstract, $abstract)`；最后一个参数
`$shared`参数用来表明是否每次解析出来的`$concret`值都是同一个，这对于单例服务来说很重要。

```php
//src/Illuminate/Container/Container.php

/**
 * Register a binding with the container.
 *
 * @param  string  $abstract
 * @param  \Closure|string|null  $concrete
 * @param  bool  $shared
 * @return void
 */
public function bind($abstract, $concrete = null, $shared = false)
{
    // If no concrete type was given, we will simply set the concrete type to the
    // abstract type. After that, the concrete type to be registered as shared
    // without being forced to state their classes in both of the parameters.
    $this->dropStaleInstances($abstract);

    if (is_null($concrete)) {
        $concrete = $abstract;
    }

    // If the factory is not a Closure, it means it is just a class name which is
    // bound into this container to the abstract type and we will just wrap it
    // up inside its own Closure to give us more convenience when extending.
    if (! $concrete instanceof Closure) {
        $concrete = $this->getClosure($abstract, $concrete);
    }

    $this->bindings[$abstract] = compact('concrete', 'shared');

    // If the abstract type was already resolved in this container we'll fire the
    // rebound listener so that any objects which have already gotten resolved
    // can have their copy of the object updated via the listener callbacks.
    if ($this->resolved($abstract)) {
        $this->rebound($abstract);
    }
}
```

`$this->dropStaleInstances`用于丢弃之前解析出来的共享结果。当服务是共享的时候，容器在第一次将服务解析出来以后，
它就将该服务保存起来，后续拿到的都是该结果，所以，当重新绑定一次时，我们需要丢弃之前解析出来的结果，否则新的绑定就不生效。

当`$concrete`是一个类名称的时候，它会被转化成一个解析函数。之后解析函数和`$shared`参数会以数组形式存放到`$this->bindings`
数组中，这个数组存放着所有容器中的服务解析函数。最后，如果`$abstract`已经解析过，那么这一次绑定就是重新绑定，那么就触发重新绑定
事件。

## 绑定单例解析函数

`singleton`方法用于绑定一个单例解析函数，它只是`$this->bind($abstract, $concrete, true)`的简写。

```php
//src/Illuminate/Container/Container.php

/**
 * Register a shared binding in the container.
 *
 * @param  string  $abstract
 * @param  \Closure|string|null  $concrete
 * @return void
 */
public function singleton($abstract, $concrete = null)
{
    $this->bind($abstract, $concrete, true);
}
```

## 绑定实例

我们可以通过`instance`方法直接将一个已经初始化了的服务绑定到容器中


```php
//src/Illuminate/Container/Container.php

/**
 * Register an existing instance as shared in the container.
 *
 * @param  string  $abstract
 * @param  mixed   $instance
 * @return mixed
 */
public function instance($abstract, $instance)
{
    $this->removeAbstractAlias($abstract);

    $isBound = $this->bound($abstract);

    unset($this->aliases[$abstract]);

    // We'll check to determine if this type has been bound before, and if it has
    // we will fire the rebound callbacks registered with the container and it
    // can be updated with consuming classes that have gotten resolved here.
    $this->instances[$abstract] = $instance;

    if ($isBound) {
        $this->rebound($abstract);
    }

    return $instance;
}
```


## 服务别名

每个服务在容器中都有一个唯一的名称，除此以外，我们还能为该名称定义多个别名。`alias`就是用于定义服务别名的方法


```php
//src/Illuminate/Container/Container.php

/**
 * Alias a type to a different name.
 *
 * @param  string  $abstract
 * @param  string  $alias
 * @return void
 */
public function alias($abstract, $alias)
{
    $this->aliases[$alias] = $abstract;

    $this->abstractAliases[$abstract][] = $alias;
}
```

该方法只是简单地建立原名称与别名之间的双向关系。`$this->aliases`数组存放的是别名对原名称的映射，也就是该别名属于哪个
原名，`$this->abstractAliases`数组存放的则是原名对别名的映射，也就是该原名拥有什么别名。