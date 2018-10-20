# 服务解析

服务解析的目的就是要根据给定的名称，将该名称对应的服务（对象）构建出来。服务解析可以分为直接解析和间接解析

## 直接解析

首先是直接解析，使用直接解析来实例化一个类和手动通过`new`来实例化一个类的效果是一样的，它比`new`方便的地方就在于
它能自动解决依赖。


```php
//src/Illuminate/Container/Container.php

/**
 * Instantiate a concrete instance of the given type.
 *
 * @param  string  $concrete
 * @return mixed
 *
 * @throws \Illuminate\Contracts\Container\BindingResolutionException
 */
public function build($concrete)
{
    // If the concrete type is actually a Closure, we will just execute it and
    // hand back the results of the functions, which allows functions to be
    // used as resolvers for more fine-tuned resolution of these objects.
    if ($concrete instanceof Closure) {
        return $concrete($this, $this->getLastParameterOverride());
    }

    $reflector = new ReflectionClass($concrete);

    // If the type is not instantiable, the developer is attempting to resolve
    // an abstract type such as an Interface of Abstract Class and there is
    // no binding registered for the abstractions so we need to bail out.
    if (! $reflector->isInstantiable()) {
        return $this->notInstantiable($concrete);
    }

    $this->buildStack[] = $concrete;

    $constructor = $reflector->getConstructor();

    // If there are no constructors, that means there are no dependencies then
    // we can just resolve the instances of the objects right away, without
    // resolving any other types or dependencies out of these containers.
    if (is_null($constructor)) {
        array_pop($this->buildStack);

        return new $concrete;
    }

    $dependencies = $constructor->getParameters();

    // Once we have all the constructor's parameters we can create each of the
    // dependency instances and then use the reflection instances to make a
    // new instance of this class, injecting the created dependencies in.
    $instances = $this->resolveDependencies(
        $dependencies
    );

    array_pop($this->buildStack);

    return $reflector->newInstanceArgs($instances);
}
```

直接解析接受的是具体的类名称或者解析函数，如果是解析函数的话，那么直接调用解析函数来完成解析过程，而如果是具体类的名称的话，
那么就要用到反射，通过反射机制找到要构建类的依赖（也就是构造函数的参数），接着使用`resolveDependencies`方法来解决这些依赖。

## 间接解析

间接解析是通过容器中的服务标识，将该标识对应的服务解析出来，间接解析比直接解析多了一步就是要根据服务标识，在容器中找到具体的服务解析函数
或者具体类。


```php
//src/Illuminate/Container/Container.php

/**
 * Resolve the given type from the container.
 *
 * @param  string  $abstract
 * @param  array  $parameters
 * @return mixed
 */
public function make($abstract, array $parameters = [])
{
    return $this->resolve($abstract, $parameters);
}
```

其中`$abstract`是服务的名称，它可以是一个之前通过服务绑定绑定到容器的服务名称，也可以是一个可实例化的类名称，如果是后者，那么`make`的作用和`build`是一样的。

它的具体实现是`resolve`方法：

```php
//src/Illuminate/Container/Container.php

/**
 * Resolve the given type from the container.
 *
 * @param  string  $abstract
 * @param  array  $parameters
 * @return mixed
 */
protected function resolve($abstract, $parameters = [])
{
    $abstract = $this->getAlias($abstract);

    $needsContextualBuild = ! empty($parameters) || ! is_null(
        $this->getContextualConcrete($abstract)
    );

    // If an instance of the type is currently being managed as a singleton we'll
    // just return an existing instance instead of instantiating new instances
    // so the developer can keep using the same objects instance every time.
    if (isset($this->instances[$abstract]) && ! $needsContextualBuild) {
        return $this->instances[$abstract];
    }

    $this->with[] = $parameters;

    $concrete = $this->getConcrete($abstract);

    // We're ready to instantiate an instance of the concrete type registered for
    // the binding. This will instantiate the types, as well as resolve any of
    // its "nested" dependencies recursively until all have gotten resolved.
    if ($this->isBuildable($concrete, $abstract)) {
        $object = $this->build($concrete);
    } else {
        $object = $this->make($concrete);
    }

    // If we defined any extenders for this type, we'll need to spin through them
    // and apply them to the object being built. This allows for the extension
    // of services, such as changing configuration or decorating the object.
    foreach ($this->getExtenders($abstract) as $extender) {
        $object = $extender($object, $this);
    }

    // If the requested type is registered as a singleton we'll want to cache off
    // the instances in "memory" so we can return it later without creating an
    // entirely new instance of an object on each subsequent request for it.
    if ($this->isShared($abstract) && ! $needsContextualBuild) {
        $this->instances[$abstract] = $object;
    }

    $this->fireResolvingCallbacks($abstract, $object);

    // Before returning, we will also set the resolved flag to "true" and pop off
    // the parameter overrides for this build. After those two things are done
    // we will be ready to return back the fully constructed class instance.
    $this->resolved[$abstract] = true;

    array_pop($this->with);

    return $object;
}
```

如果忽略context build，那么整个方法的逻辑也很简单，首先通过`getAlias`找到`$abstract`在容器中的最终名称，因为`$abstract`有可能是一个别名。然后看该名称是不是绑定到一个实例上，如果是的话，那么就直接返回这个实例。如果不考虑context build的情况，那么这时候的`$concrete`
值有可能是一个类名称或者是一个解析函数，满足这两者任意一个，逻辑就进入到了`build`方法。

当对象构建完毕以后，接下来就是依次调用extender来对构建好了的对象作扩展，然后再根据绑定时候的设置，如果设置为共享对象，那么这个对象就会被加入到`$this->instances`数组中，后续的`make`就只需要返回之前创建好了的对象就好，不需要再重新`make`。

最后就是触发事件以及修改状态了表明容器中这个名称已经被解析过了。