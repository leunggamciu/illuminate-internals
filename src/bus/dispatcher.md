# 指令分发器

在开始分析指令分发器之前，我们先来看一下指令分发器是怎样绑定到服务容器的

```php
//src/Illuminate/Bus/BusServiceProvider.php

/**
 * Register the service provider.
 *
 * @return void
 */
public function register()
{
    $this->app->singleton(Dispatcher::class, function ($app) {
        return new Dispatcher($app, function ($connection = null) use ($app) {
            return $app[QueueFactoryContract::class]->connection($connection);
        });
    });

    $this->app->alias(
        Dispatcher::class, DispatcherContract::class
    );

    $this->app->alias(
        Dispatcher::class, QueueingDispatcherContract::class
    );
}
```

注意`\Illuminate\Bus\Dispatcher`的第二个参数，它是一个用来获取默认队列的函数，因为异步指令的分发就是通过队列实现的。

下面是分发器完整的构造方法

```php
//src/Illuminate/Bus/Dispatcher.php

/**
 * Create a new command dispatcher instance.
 *
 * @param  \Illuminate\Contracts\Container\Container  $container
 * @param  \Closure|null  $queueResolver
 * @return void
 */
public function __construct(Container $container, Closure $queueResolver = null)
{
    $this->container = $container;
    $this->queueResolver = $queueResolver;
    $this->pipeline = new Pipeline($container);
}
```

暂时先忽略`$pipeline`属性。

下面来看一下分发指令的逻辑

```php
//src/Illuminate/Bus/Dispatcher.php

/**
 * Dispatch a command to its appropriate handler.
 *
 * @param  mixed  $command
 * @return mixed
 */
public function dispatch($command)
{
    if ($this->queueResolver && $this->commandShouldBeQueued($command)) {
        return $this->dispatchToQueue($command);
    }

    return $this->dispatchNow($command);
}
```

逻辑很简单，首先`$queueResolver`在服务提供者的注册方法处已经设置了，所以具体指令是要分发到队列还是立马执行就要看`commandShouldBeQueued`

```php
//src/Illuminate/Bus/Dispatcher.php

/**
 * Determine if the given command should be queued.
 *
 * @param  mixed  $command
 * @return bool
 */
protected function commandShouldBeQueued($command)
{
    return $command instanceof ShouldQueue;
}
```

它就是判断指令本身到底有没有实现`\Illuminate\Contracts\Queue\ShouldQueue`。

我们首先看指令立马执行的情况，也就是同步指令

```php
//src/Illuminate/Bus/Dispatcher.php

/**
 * Dispatch a command to its appropriate handler in the current process.
 *
 * @param  mixed  $command
 * @param  mixed  $handler
 * @return mixed
 */
public function dispatchNow($command, $handler = null)
{
    if ($handler || $handler = $this->getCommandHandler($command)) {
        $callback = function ($command) use ($handler) {
            return $handler->handle($command);
        };
    } else {
        $callback = function ($command) {
            return $this->container->call([$command, 'handle']);
        };
    }

    return $this->pipeline->send($command)->through($this->pipes)->then($callback);
}
```

在这里要说明一下的是，同步指令有自执行(self handling)和使用独立处理器来执行两种，自执行的指令就是指令本身带有
`handle`方法的指令，而使用独立处理器来执行的话，那么就需要通过分发器的`map`方法来建立指令和对应指令处理器之间
的关系。独立指令处理器就是一个拥有`handle`方法，并接受对应指令类作为参数。

在分发同步指令时，还应用了管道这个组件，这样就能像为同步指令添加中间件支持。分发器的`pipeThrough`就是用来指定
中间件的

```php
//src/Illuminate/Bus/Dispatcher.php

/**
 * Set the pipes through which commands should be piped before dispatching.
 *
 * @param  array  $pipes
 * @return $this
 */
public function pipeThrough(array $pipes)
{
    $this->pipes = $pipes;

    return $this;
}
```

接下来再看将指令分发到队列的情况

```php
//src/Illuminate/Bus/Dispatcher.php

/**
 * Dispatch a command to its appropriate handler behind a queue.
 *
 * @param  mixed  $command
 * @return mixed
 *
 * @throws \RuntimeException
 */
public function dispatchToQueue($command)
{
    $connection = $command->connection ?? null;

    $queue = call_user_func($this->queueResolver, $connection);

    if (! $queue instanceof Queue) {
        throw new RuntimeException('Queue resolver did not return a Queue implementation.');
    }

    if (method_exists($command, 'queue')) {
        return $command->queue($queue, $command);
    }

    return $this->pushCommandToQueue($queue, $command);
}
```

首先通过`$queueResolver`获取队列，如果指令中存在`queue`方法的话，那么就调用指令的`queue`方法，这相当于我们在指令的
定义中就能指定该指令推入到哪个队列以及是否延迟执行。

如果指令中没有`queue`方法的话，那么就直接调用`pushCommandToQueue`

```php
//src/Illuminate/Bus/Dispatcher.php

/**
 * Push the command onto the given queue instance.
 *
 * @param  \Illuminate\Contracts\Queue\Queue  $queue
 * @param  mixed  $command
 * @return mixed
 */
protected function pushCommandToQueue($queue, $command)
{
    if (isset($command->queue, $command->delay)) {
        return $queue->laterOn($command->queue, $command->delay, $command);
    }

    if (isset($command->queue)) {
        return $queue->pushOn($command->queue, $command);
    }

    if (isset($command->delay)) {
        return $queue->later($command->delay, $command);
    }

    return $queue->push($command);
}
```

它主要的作用就是根据指令中的属性，来决定要把指令推入到哪个队列以及是否延迟执行。也就是说，除了能在`queue`方法中指定以外，还可以指定
`$queue`，`$delay`这两个公共属性来达到同样的目的，当然，`queue`方法要优先。