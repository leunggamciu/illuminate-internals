# 管道

在分析它的实现原理之前，先来看看它的用法

```php
$result = (new Pipeline(new \Illuminate\Container\Container))
            ->send('foo')
            ->through([$m1, $m2])
            ->then(function ($piped) {
                return $piped;
            });
```

它的用法很简单，从字面上理解就是发送`foo`，在`foo`流过`$m1`和`$m2`以后，执行then里面定义的回调。

如果这么看的话，那么貌似它的实现也不复杂，只需要以`foo`为参数调用`m1`，将得到的结果作为参数调用`m2`，再将得到的结果作为参数调用`destination`，
最后调用`destination`。这是前置中间件

后置中间件的实现和前置的类似，只需要将调用`destination`后得到的值作为参数调用后置中间件`m3`，再将结果作为参数调用`m4`，依次类推，最后得出的
结果就是经过后置中间件处理过的结果。

但是这种方式实现的中间件使用起来有一些缺点，例如没法控制后面中间件的执行。

## 洋葱圈模型

![洋葱圈模型](https://camo.githubusercontent.com/d80cf3b511ef4898bcde9a464de491fa15a50d06/68747470733a2f2f7261772e6769746875622e636f6d2f66656e676d6b322f6b6f612d67756964652f6d61737465722f6f6e696f6e2e706e67)



如果不看`through`方法，那么

```php
(new Pipeline)->send('foo')->then(function($piped) {
    return $piped;
})
```

和

```php
$pipe = function($piped) {
    return $piped;
};
$pipe('foo');
```

的作用是一样的。在Laravel中，我们把`pipe`这个函数称为`destination`，它是洋葱圈最里面的那一圈。假如现在我们要在这一圈的基础上再套一圈
那么我们首先要保证套上的这一圈也是一个函数，而且和`destination`有着一样的签名。所以就有

```php

//$enhancedPipe1的作用就是修饰$pipe函数，所以它接收的参数是$pipe，返回的值是和$pipe具有相同签名的函数
$enhancedPiped1 = function($pipe) {
    //下面返回的函数就是洋葱的第二圈
    return function($piped) use ($pipe) {
        //该中间件的业务逻辑都写在这里
        //$pipe是洋葱的第一圈，也就是destination

        //如果这里写逻辑，那么这个中间件就是一个前置中间件

        //这里是先执行里面一层的中间件
        //如果忽略这一样，那么里面一层的中间件就不会执行
        $result = $pipe($piped);

        //如果这里写逻辑，那么这个中间件就是一个后置中间件
    }
};
```

套上一层以后，调用就变成

```php
$enhancedPipe = $enhancedPipe1($pipe);
$enhancedPipe('foo');
```

如果再套一层，那么就变成

```php

$enhancedPipe2 = function($enhancedPipe1) {
    return function($piped) use ($enhancedPipe1) {
        //洋葱的第三圈
    };
};
```

调用就变成

```php
$enhancedPipe = $enhancedPipe2($enhancedPipe1($pipe));
$enhancedPipe('foo');
```

按照这个逻辑，我们可以定义任意数量的中间件。

可是，如果按照这种方式来调用的话，那么`$enhancedPipe1`和`$enhancedPipe2`的调用顺序就固定了。
如果我想要修改调用顺序，像`$enhancedPipe1($enhancedPipe2($pipe))`，就要修改整个调用过程，当数量多起来以后，就变得十分难维护。

仔细观察以下， 实际上`$enhancedPipe1`和`$enhancedPipe2`函数逻辑是一样的，不同的只是返回函数里面的业务逻辑，我们把这些业务逻辑单独
抽出来，用数组存放起来。

```php
//$m1和$m2都是callable的
$logics = [$m1, $m2];
```

我们已经有`$pipe`这个函数了，接下来就是要构建`$enhancedPipe`，这里非常适用的一个函数就是`array_reduce`，将`$pipe`作为初始值
结合`$logics`，最终归纳成一个`$enhancedPipe`函数。

```php

$enhancedPipe = array_reduce($logics, function($carry, $m) {
    //最终要归纳成和$pipe具有相同签名的函数
    return function($piped) use ($carry, $m) {
        
        return $m($piped, $carry);
    };
}, $pipe);
$enhancedPipe('foo');
```

## Laravel实现

```php
//src/Illuminate/Pipeline/Pipeline.php

/**
 * Set the object being sent through the pipeline.
 *
 * @param  mixed  $passable
 * @return $this
 */
public function send($passable)
{
    $this->passable = $passable;

    return $this;
}

/**
 * Set the array of pipes.
 *
 * @param  array|mixed  $pipes
 * @return $this
 */
public function through($pipes)
{
    $this->pipes = is_array($pipes) ? $pipes : func_get_args();

    return $this;
}

/**
 * Run the pipeline with a final destination callback.
 *
 * @param  \Closure  $destination
 * @return mixed
 */
public function then(Closure $destination)
{
    $pipeline = array_reduce(
        array_reverse($this->pipes), $this->carry(), $this->prepareDestination($destination)
    );

    return $pipeline($this->passable);
}
```

`send`是用于定义要传入到`destination`函数的参数，`through`就是定义所有要执行的中间件，最后由`then`负责构建整个洋葱圈。

```php
//src/Illuminate/Pipeline/Pipeline.php

/**
 * Get the final piece of the Closure onion.
 *
 * @param  \Closure  $destination
 * @return \Closure
 */
protected function prepareDestination(Closure $destination)
{
    return function ($passable) use ($destination) {
        return $destination($passable);
    };
}
```

`prepareDestination`在这里的作用有点多余，实际上`$destination`本身就是洋葱圈最里面的那一层，所以`then`完全可以改写成

```php
$pipeline = array_reduce(
    array_reverse($this->pipes), $this->carry(), $destination
);

return $pipeline($this->passable);
```

重点是`carry`方法

```php
//src/Illuminate/Pipeline/Pipeline.php

/**
 * Get a Closure that represents a slice of the application onion.
 *
 * @return \Closure
 */
protected function carry()
{
    return function ($stack, $pipe) {
        return function ($passable) use ($stack, $pipe) {
            if (is_callable($pipe)) {
                // If the pipe is an instance of a Closure, we will just call it directly but
                // otherwise we'll resolve the pipes out of the container and call it with
                // the appropriate method and arguments, returning the results back out.
                return $pipe($passable, $stack);
            } elseif (! is_object($pipe)) {
                list($name, $parameters) = $this->parsePipeString($pipe);

                // If the pipe is a string we will parse the string and resolve the class out
                // of the dependency injection container. We can then build a callable and
                // execute the pipe function giving in the parameters that are required.
                $pipe = $this->getContainer()->make($name);

                $parameters = array_merge([$passable, $stack], $parameters);
            } else {
                // If the pipe is already an object we'll just make a callable and pass it to
                // the pipe as-is. There is no need to do any extra parsing and formatting
                // since the object we're given was already a fully instantiated object.
                $parameters = [$passable, $stack];
            }

            return method_exists($pipe, $this->method)
                            ? $pipe->{$this->method}(...$parameters)
                            : $pipe(...$parameters);
        };
    };
}
```

这里直接看第二个`return`，和前面实现不同的是，这里支持多种中间件调用方式，中间件可以是一个可调用的函数，类等等。