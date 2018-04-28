# 视图

视图是用户最终可见的页面的抽象，视图有它的名称，对应的模板路径，渲染模板所用到的渲染引擎，以及渲染要用到的数据。视图
有可能不是独立存在的，一个视图可能从其他视图继承过来，也可能include其他视图，也就是说，一个视图有可能由多个视图构成。
我把第一个渲染的视图称为入口视图，入口视图所依赖的其他视图称为非入口视图，入口视图与非入口视图是相对而言的。视图所依赖
的`factory`可以理解成一个状态管理的对象，通过它我们可以知道当前视图与入口视图之间的关系，入口视图是否渲染完毕等等。

视图中最主要的方法就是`render`，它返回渲染完成后的视图。

```php
//src/Illuminate/View/View.php

/**
 * Get the string contents of the view.
 *
 * @param  callable|null  $callback
 * @return string
 *
 * @throws \Throwable
 */
public function render(callable $callback = null)
{
    try {
        $contents = $this->renderContents();

        $response = isset($callback) ? call_user_func($callback, $this, $contents) : null;

        // Once we have the contents of the view, we will flush the sections if we are
        // done rendering all views so that there is nothing left hanging over when
        // another view gets rendered in the future by the application developer.
        $this->factory->flushStateIfDoneRendering();

        return ! is_null($response) ? $response : $contents;
    } catch (Exception $e) {
        $this->factory->flushState();

        throw $e;
    } catch (Throwable $e) {
        $this->factory->flushState();

        throw $e;
    }
}
```

它调用`renderContents`来渲染视图，如果渲染完毕(入口视图渲染完毕)，那么就将所有状态重置。


```php
//src/Illuminate/View/View.php

/**
 * Get the contents of the view instance.
 *
 * @return string
 */
protected function renderContents()
{
    // We will keep track of the amount of views being rendered so we can flush
    // the section after the complete rendering operation is done. This will
    // clear out the sections for any separate views that may be rendered.
    $this->factory->incrementRender();

    $this->factory->callComposer($this);

    $contents = $this->getContents();

    // Once we've finished rendering the view, we'll decrement the render count
    // so that each sections get flushed out next time a view is created and
    // no old sections are staying around in the memory of an environment.
    $this->factory->decrementRender();

    return $contents;
}
```

`renderContents`主要是调用`getContents`来获取渲染后生成的字符串。`factory`维护着一个计数器，用来记录当前视图
相对于入口视图来说，处于哪一级，这个计数器在稍后会用到。它还触发了`composing:view名称`事件。

`getContents`只是简单地调用对应的渲染引擎，完成渲染工作。

```php
//src/Illuminate/View/View.php

/**
 * Get the evaluated contents of the view.
 *
 * @return string
 */
protected function getContents()
{
    return $this->engine->get($this->path, $this->gatherData());
}
```