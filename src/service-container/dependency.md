# 依赖解决

依赖解决是服务容器中最重要的一个功能之一。容器通过反射得到要实例化的类的依赖以后，调用`resolveDependencies`方法来实现
依赖解决。

```php
//src/Illuminate/Container/Container.php

/**
 * Resolve all of the dependencies from the ReflectionParameters.
 *
 * @param  array  $dependencies
 * @return array
 */
protected function resolveDependencies(array $dependencies)
{
    $results = [];

    foreach ($dependencies as $dependency) {
        // If this dependency has a override for this particular build we will use
        // that instead as the value. Otherwise, we will continue with this run
        // of resolutions and let reflection attempt to determine the result.
        if ($this->hasParameterOverride($dependency)) {
            $results[] = $this->getParameterOverride($dependency);

            continue;
        }

        // If the class is null, it means the dependency is a string or some other
        // primitive type which we can not resolve since it is not a class and
        // we will just bomb out with an error since we have no-where to go.
        $results[] = is_null($dependency->getClass())
                        ? $this->resolvePrimitive($dependency)
                        : $this->resolveClass($dependency);
    }

    return $results;
}
```

`$dependencies`是一个数组，它包含了要实例化的类的构造函数中所有参数，它们的类型是`\ReflectionParameter`。
通过`\ReflectionParameter`，我们可以知道这个参数的类型是primitive还是class。如果它是primitive的话，那么就
调用`resolvePrimitive`，否则就调用`resolveClass`。

```php
//src/Illuminate/Container/Container.php

/**
 * Resolve a non-class hinted primitive dependency.
 *
 * @param  \ReflectionParameter  $parameter
 * @return mixed
 *
 * @throws \Illuminate\Contracts\Container\BindingResolutionException
 */
protected function resolvePrimitive(ReflectionParameter $parameter)
{
    if (! is_null($concrete = $this->getContextualConcrete('$'.$parameter->name))) {
        return $concrete instanceof Closure ? $concrete($this) : $concrete;
    }

    if ($parameter->isDefaultValueAvailable()) {
        return $parameter->getDefaultValue();
    }

    $this->unresolvablePrimitive($parameter);
}

/**
 * Resolve a class based dependency from the container.
 *
 * @param  \ReflectionParameter  $parameter
 * @return mixed
 *
 * @throws \Illuminate\Contracts\Container\BindingResolutionException
 */
protected function resolveClass(ReflectionParameter $parameter)
{
    try {
        return $this->make($parameter->getClass()->name);
    }

    // If we can not resolve the class instance, we will check to see if the value
    // is optional, and if it is we will return the optional parameter value as
    // the value of the dependency, similarly to how we do this with scalars.
    catch (BindingResolutionException $e) {
        if ($parameter->isOptional()) {
            return $parameter->getDefaultValue();
        }

        throw $e;
    }
}
```

对于primitive类型的值来说，如果没有默认值的情况下，容器是没法构建它的值。而对于class类型来说，容器直接以class类型的名称，通过
间接解析，将实例从容器中解析出来作为参数值。
