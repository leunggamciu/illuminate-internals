# 视图工厂

一、使用`file`方法根据模板路径来渲染

```php
//src/Illuminate/View/Factory.php

/**
 * Get the evaluated view contents for the given view.
 *
 * @param  string  $path
 * @param  array   $data
 * @param  array   $mergeData
 * @return \Illuminate\Contracts\View\View
 */
public function file($path, $data = [], $mergeData = [])
{
    $data = array_merge($mergeData, $this->parseData($data));

    return tap($this->viewInstance($path, $path, $data), function ($view) {
        $this->callCreator($view);
    });
}
```

首选使用`parseData`来将接收到的数据转化成可用于渲染的数据，目前是把实现了`Arrayable`接口的数据转化成数组，实现如下：


```php
//src/Illuminate/View/Factory.php

/**
 * Parse the given data into a raw array.
 *
 * @param  mixed  $data
 * @return array
 */
protected function parseData($data)
{
    return $data instanceof Arrayable ? $data->toArray() : $data;
}
```

接下来使用`viewInstance`方法，创建一个`View`实例，紧接着触发`creating: view名称`事件。

二、使用`make`方法根据模板名称来渲染

与`file`不同的是，`make`方法只需要提供视图的名称，它就会在候选目录中找到对应的模板来渲染。


```php
//src/Illuminate/View/Factory.php

/**
 * Get the evaluated view contents for the given view.
 *
 * @param  string  $view
 * @param  array   $data
 * @param  array   $mergeData
 * @return \Illuminate\Contracts\View\View
 */
public function make($view, $data = [], $mergeData = [])
{
    $path = $this->finder->find(
        $view = $this->normalizeName($view)
    );

    // Next, we will create the view instance and call the view creator for the view
    // which can set any data, etc. Then we will return the view instance back to
    // the caller for rendering or performing other view manipulations on this.
    $data = array_merge($mergeData, $this->parseData($data));

    return tap($this->viewInstance($view, $path, $data), function ($view) {
        $this->callCreator($view);
    });
}
```

比`file`方法多出了下面这段用于查找模板路径的逻辑：

```php
$path = $this->finder->find(
    $view = $this->normalizeName($view)
);
```

由于View名称是用`.`来分割目录的，所以`normalizeName`的作用就是把`.`转化成路径分隔符`/`。

三、使用`first`渲染第一个找到的view

```php
//src/Illuminate/View/Factory.php

/**
 * Get the first view that actually exists from the given list.
 *
 * @param  array  $views
 * @param  array   $data
 * @param  array   $mergeData
 * @return \Illuminate\Contracts\View\View
 */
public function first(array $views, $data = [], $mergeData = [])
{
    $view = collect($views)->first(function ($view) {
        return $this->exists($view);
    });

    if (! $view) {
        throw new InvalidArgumentException('None of the views in the given array exist.');
    }

    return $this->make($view, $data, $mergeData);
}
```

其中`$view`参数就是一组待渲染的View名称，`first`方法的作用就是从候选目录中找这些view，渲染第一个找到的view。这在
模板覆盖的场景中可以用到。

四、使用`renderWhen`实现条件渲染

`renderWhen`的实现很简单，不满足条件直接返回空字符串。

```php
//src/Illuminate/View/Factory.php

/**
 * Get the rendered content of the view based on a given condition.
 *
 * @param  bool  $condition
 * @param  string  $view
 * @param  array   $data
 * @param  array   $mergeData
 * @return string
 */
public function renderWhen($condition, $view, $data = [], $mergeData = [])
{
    if (! $condition) {
        return '';
    }

    return $this->make($view, $this->parseData($data), $mergeData)->render();
}
```

后面Blade中的`@includeIf`指令就使用了这个方法。

五、使用`renderEach`实现不同数据渲染同一份模板，并拼接所有结果


```php
//src/Illuminate/View/Factory.php

/**
 * Get the rendered contents of a partial from a loop.
 *
 * @param  string  $view
 * @param  array   $data
 * @param  string  $iterator
 * @param  string  $empty
 * @return string
 */
public function renderEach($view, $data, $iterator, $empty = 'raw|')
{
    $result = '';

    // If is actually data in the array, we will loop through the data and append
    // an instance of the partial view to the final result HTML passing in the
    // iterated value of this data array, allowing the views to access them.
    if (count($data) > 0) {
        foreach ($data as $key => $value) {
            $result .= $this->make(
                $view, ['key' => $key, $iterator => $value]
            )->render();
        }
    }

    // If there is no data in the array, we will render the contents of the empty
    // view. Alternatively, the "empty view" could be a raw string that begins
    // with "raw|" for convenience and to let this know that it is a string.
    else {
        $result = Str::startsWith($empty, 'raw|')
                    ? substr($empty, 4)
                    : $this->make($empty)->render();
    }

    return $result;
}
```

后面Blade指令中的`@each`指令就使用了这个方法。

这里比较有意思的地方是第四个参数，如果在`$data`为空的时候不需要渲染一个特定的视图，而只给出一段普通的字符串，那么可以传递
`raw|$data为空时候显示的字符串`，否则`$empty`参数就是模板名称。

六、使用`exists`方法判断对应名称的模板在候选目录中是否存在

```php
//src/Illuminate/View/Factory.php

/**
 * Determine if a given view exists.
 *
 * @param  string  $view
 * @return bool
 */
public function exists($view)
{
    try {
        $this->finder->find($view);
    } catch (InvalidArgumentException $e) {
        return false;
    }

    return true;
}
```

这里没有什么可讲的了，具体实现可以参考[view-finder](./view-finder.md)。

七、使用`getEngineFromPath`根据模板路径获取对应渲染引擎

```php
//src/Illuminate/View/Factory.php

/**
 * Get the appropriate view engine for the given path.
 *
 * @param  string  $path
 * @return \Illuminate\Contracts\View\Engine
 *
 * @throws \InvalidArgumentException
 */
public function getEngineFromPath($path)
{
    if (! $extension = $this->getExtension($path)) {
        throw new InvalidArgumentException("Unrecognized extension in file: {$path}");
    }

    $engine = $this->extensions[$extension];

    return $this->engines->resolve($engine);
}
```

主要是根据模板名称的后缀，找到对应的渲染引擎的名称，然后使用[engine-resolver](./engine-resolver.md)来根据引擎名称找到对应的渲染引擎实例。

八、使用`share`方法实现视图间数据共享


```php
//src/Illuminate/View/Factory.php

/**
 * Add a piece of shared data to the environment.
 *
 * @param  array|string  $key
 * @param  mixed  $value
 * @return mixed
 */
public function share($key, $value = null)
{
    $keys = is_array($key) ? $key : [$key => $value];

    foreach ($keys as $key => $value) {
        $this->shared[$key] = $value;
    }

    return $value;
}
```

`share`方法主要是把数据放到`$this->shared`中，View在渲染的时候会通过factory的`getShared`方法找到所有共享数据，与普通数据组装以后
再做渲染。


九、候选目录与命名空间


```php
//src/Illuminate/View/Factory.php

/**
 * Add a location to the array of view locations.
 *
 * @param  string  $location
 * @return void
 */
public function addLocation($location)
{
    $this->finder->addLocation($location);
}

/**
 * Add a new namespace to the loader.
 *
 * @param  string  $namespace
 * @param  string|array  $hints
 * @return $this
 */
public function addNamespace($namespace, $hints)
{
    $this->finder->addNamespace($namespace, $hints);

    return $this;
}

/**
 * Prepend a new namespace to the loader.
 *
 * @param  string  $namespace
 * @param  string|array  $hints
 * @return $this
 */
public function prependNamespace($namespace, $hints)
{
    $this->finder->prependNamespace($namespace, $hints);

    return $this;
}

/**
 * Replace the namespace hints for the given namespace.
 *
 * @param  string  $namespace
 * @param  string|array  $hints
 * @return $this
 */
public function replaceNamespace($namespace, $hints)
{
    $this->finder->replaceNamespace($namespace, $hints);

    return $this;
}
```

具体实现可以参考[view-finder](./view-finder.md)

十、注册自定义渲染引擎


```php
//src/Illuminate/View/Factory.php

/**
 * Register a valid view extension and its engine.
 *
 * @param  string    $extension
 * @param  string    $engine
 * @param  \Closure  $resolver
 * @return void
 */
public function addExtension($extension, $engine, $resolver = null)
{
    $this->finder->addExtension($extension);

    if (isset($resolver)) {
        $this->engines->register($engine, $resolver);
    }

    unset($this->extensions[$extension]);

    $this->extensions = array_merge([$extension => $engine], $this->extensions);
}
```

其中`$extension`是自定义渲染引擎要渲染的模板文件的后缀， `$engine`为引擎名称，`$resolver`是一个根据引擎名称
返回对应引擎实例的函数。

十一、状态管理

```php
//src/Illuminate/View/Factory.php

/**
 * Increment the rendering counter.
 *
 * @return void
 */
public function incrementRender()
{
    $this->renderCount++;
}

/**
 * Decrement the rendering counter.
 *
 * @return void
 */
public function decrementRender()
{
    $this->renderCount--;
}

/**
 * Check if there are no active render operations.
 *
 * @return bool
 */
public function doneRendering()
{
    return $this->renderCount == 0;
}

/**
 * Flush all of the factory state like sections and stacks.
 *
 * @return void
 */
public function flushState()
{
    $this->renderCount = 0;

    $this->flushSections();
    $this->flushStacks();
}

/**
 * Flush all of the section contents if done rendering.
 *
 * @return void
 */
public function flushStateIfDoneRendering()
{
    if ($this->doneRendering()) {
        $this->flushState();
    }
}
```

在渲染一个入口视图的过程中，Factory本身保存很多状态，这些状态应该只属于本次渲染过程，所以它需要维护一套机制，在渲染过程结束以后，
重置这些状态，这样就不会影响到下一个入口视图的渲染。

系统在渲染一个视图之前，会调用`incrementRender`，渲染结束以后调用`decrementRender`，直到这个计数器为0，那么就表示入口视图已经
渲染好了，这也是`doneRendering`要做的事情。

`flushState`就是用来重置状态的。