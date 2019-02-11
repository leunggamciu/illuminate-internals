# 自分发指令

一般情况下，我们分发指令可以使用`dispatch`助手函数，或者从服务容器中拿到`\Illuminate\Bus\Dispatcher`服务，用它
来分发。例如

```php
dispatch(new SendEmailCommand)
```

或者

```php
app(Dispatcher::class)->dispatch(new SendEmailCommand);
```

Laravel提供了一个特征，引入这个特征以后，分发指令就可以这样

```php
SendEmailCommand::dispatch();
```

这个特征就是`\Illuminate\Foundation\Bus\Dispatchable`

```php
//src/Illuminate/Foundation/Bus/Dispatchable.php

trait Dispatchable
{
    /**
     * Dispatch the job with the given arguments.
     *
     * @return \Illuminate\Foundation\Bus\PendingDispatch
     */
    public static function dispatch()
    {
        return new PendingDispatch(new static(...func_get_args()));
    }

    /**
     * Dispatch a command to its appropriate handler in the current process.
     *
     * @return mixed
     */
    public static function dispatchNow()
    {
        return app(Dispatcher::class)->dispatchNow(new static(...func_get_args()));
    }

    /**
     * Set the jobs that should run if this job is successful.
     *
     * @param  array  $chain
     * @return \Illuminate\Foundation\Bus\PendingChain
     */
    public static function withChain($chain)
    {
        return new PendingChain(get_called_class(), $chain);
    }
}
```

实际上就是为指令类引入了三个静态方法

首先来看`dispatch`，它直接返回一个`\Illuminate\Foundation\Bus\PendingDispatch`实例，这是因为需要支持在分发
的时候指定队列连接，队列名称等信息，这个实例就是用来指定这些信息的。

```php
//src/Illuminate/Foundation/Bus/PendingDispatch.php

class PendingDispatch
{
    /**
     * The job.
     *
     * @var mixed
     */
    protected $job;

    /**
     * Create a new pending job dispatch.
     *
     * @param  mixed  $job
     * @return void
     */
    public function __construct($job)
    {
        $this->job = $job;
    }

    /**
     * Set the desired connection for the job.
     *
     * @param  string|null  $connection
     * @return $this
     */
    public function onConnection($connection)
    {
        $this->job->onConnection($connection);

        return $this;
    }

    /**
     * Set the desired queue for the job.
     *
     * @param  string|null  $queue
     * @return $this
     */
    public function onQueue($queue)
    {
        $this->job->onQueue($queue);

        return $this;
    }

    /**
     * Set the desired connection for the chain.
     *
     * @param  string|null  $connection
     * @return $this
     */
    public function allOnConnection($connection)
    {
        $this->job->allOnConnection($connection);

        return $this;
    }

    /**
     * Set the desired queue for the chain.
     *
     * @param  string|null  $queue
     * @return $this
     */
    public function allOnQueue($queue)
    {
        $this->job->allOnQueue($queue);

        return $this;
    }

    /**
     * Set the desired delay for the job.
     *
     * @param  \DateTime|int|null  $delay
     * @return $this
     */
    public function delay($delay)
    {
        $this->job->delay($delay);

        return $this;
    }

    /**
     * Set the jobs that should run if this job is successful.
     *
     * @param  array  $chain
     * @return $this
     */
    public function chain($chain)
    {
        $this->job->chain($chain);

        return $this;
    }

    /**
     * Handle the object's destruction.
     *
     * @return void
     */
    public function __destruct()
    {
        app(Dispatcher::class)->dispatch($this->job);
    }
}
```

这个特征的重点就在析构方法中，它把实际的分发逻辑隐藏在这里，否则就需要再显式提供一个dispatch方法，那么自分发的调用就变成

```php
SendEmailCommand::dispatch()->onQueue('email')->dispatch();
```

`dispatchNow`是用于分发同步指令，所以不需要指定队列信息，因此可以直接分发，

而`withChain`则是为自分发提供指定指令链的接口。它返回`\Illuminate\Foundation\Bus\PendingChain`实例，然后就可以通过
这个实例来指定指令链，这个实例还提供一个`dispatch`方法，它返回的是`\Illuminate\Foundation\Bus\PendingDispatch`，
通过这两步操作，就能实现

```php
SendEmailCommand::withChain([new FooCommand])->onQueue('email');
```

```php
//src/Illuminate/Foundation/Bus/PendingChain.php

class PendingChain
{
    /**
     * The class name of the job being dispatched.
     *
     * @var string
     */
    public $class;

    /**
     * The jobs to be chained.
     *
     * @var array
     */
    public $chain;

    /**
     * Create a new PendingChain instance.
     *
     * @param  string  $class
     * @param  array  $chain
     * @return void
     */
    public function __construct($class, $chain)
    {
        $this->class = $class;
        $this->chain = $chain;
    }

    /**
     * Dispatch the job with the given arguments.
     *
     * @return \Illuminate\Foundation\Bus\PendingDispatch
     */
    public function dispatch()
    {
        return (new PendingDispatch(
            new $this->class(...func_get_args())
        ))->chain($this->chain);
    }
}
```

当然，我们也可以不用这种方法来指定指令链，因为`PendingDispatch`已经提供了一个`chain`方法，所以，上述的调用可以改写成

```php
SendEmailCommand::dispatch()->chain([new FooCommand])->onQueue('email');
```