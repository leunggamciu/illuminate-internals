# Worker

worker是一个单独运行的进程


```php
//src/Illuminate/Queue/Worker.php

/**
 * Listen to the given queue in a loop.
 *
 * @param  string  $connectionName
 * @param  string  $queue
 * @param  \Illuminate\Queue\WorkerOptions  $options
 * @return void
 */
public function daemon($connectionName, $queue, WorkerOptions $options)
{
    if ($this->supportsAsyncSignals()) {
        $this->listenForSignals();
    }

    $lastRestart = $this->getTimestampOfLastQueueRestart();

    while (true) {
        // Before reserving any jobs, we will make sure this queue is not paused and
        // if it is we will just pause this worker for a given amount of time and
        // make sure we do not need to kill this worker process off completely.
        if (! $this->daemonShouldRun($options, $connectionName, $queue)) {
            $this->pauseWorker($options, $lastRestart);

            continue;
        }

        // First, we will attempt to get the next job off of the queue. We will also
        // register the timeout handler and reset the alarm for this job so it is
        // not stuck in a frozen state forever. Then, we can fire off this job.
        $job = $this->getNextJob(
            $this->manager->connection($connectionName), $queue
        );

        if ($this->supportsAsyncSignals()) {
            $this->registerTimeoutHandler($job, $options);
        }

        // If the daemon should run (not in maintenance mode, etc.), then we can run
        // fire off this job for processing. Otherwise, we will need to sleep the
        // worker so no more jobs are processed until they should be processed.
        if ($job) {
            $this->runJob($job, $connectionName, $options);
        } else {
            $this->sleep($options->sleep);
        }

        // Finally, we will check to see if we have exceeded our memory limits or if
        // the queue should restart based on other indications. If so, we'll stop
        // this worker and let whatever is "monitoring" it restart the process.
        $this->stopIfNecessary($options, $lastRestart);
    }
}
```

`daemon`方法是`queue:work`命令的入口。虽然它叫daemon，实际上它并不是一个daemon进程

如果有pcntl扩展的情况下，那么它首先安装一些信号处理函数

```php
//src/Illuminate/Queue/Worker.php

/**
 * Enable async signals for the process.
 *
 * @return void
 */
protected function listenForSignals()
{
    pcntl_async_signals(true);

    pcntl_signal(SIGTERM, function () {
        $this->shouldQuit = true;
    });

    pcntl_signal(SIGUSR2, function () {
        $this->paused = true;
    });

    pcntl_signal(SIGCONT, function () {
        $this->paused = false;
    });
}
```

它安装了SIGTERM信号处理函数，这样就能安全地使用Ctrl+C来终止worker，而SIGUSR2则用于暂停worker的运行，
SIGCONT用于恢复worker的运行。

接下来获取worker上一次重启的时间戳，这个时间戳是用来判断worker是不是应该停止，在queue:restart命令中，
实际上它就是把一个时间戳写入缓存中，所以在进入任务处理循环前拿到这个时间戳，在任务处理循环中再获取一次，
对比两次的值就知道用户是否运行了queue:restart命令，如果有，那么worker就进入停止流程。所以，实际上
queue:restart并不是重启，它只是一个安全停止的命令。

接下来进入任务处理循环

它首先判断这个worker应不应该进入下面的获取任务-处理任务的流程，这是因为在Laravel中，默认情况下，处于
维护模式中的应用，worker是不处理任务的，又或者收到了SIGUSR2信号，worker需要暂停，这时，它就不需要进入
下面的流程，只需要睡眠一段时间，然后进入下一次循环。

```php
//src/Illuminate/Queue/Worker.php

/**
 * Determine if the daemon should process on this iteration.
 *
 * @param  \Illuminate\Queue\WorkerOptions  $options
 * @param  string  $connectionName
 * @param  string  $queue
 * @return bool
 */
protected function daemonShouldRun(WorkerOptions $options, $connectionName, $queue)
{
    return ! (($this->manager->isDownForMaintenance() && ! $options->force) ||
        $this->paused ||
        $this->events->until(new Events\Looping($connectionName, $queue)) === false);
}
```

worker在启动的时候还可以指定force选项，这样，即使在维护模式下，worker也会处理任务。

```php
//src/Illuminate/Queue/Worker.php

/**
 * Pause the worker for the current loop.
 *
 * @param  \Illuminate\Queue\WorkerOptions  $options
 * @param  int  $lastRestart
 * @return void
 */
protected function pauseWorker(WorkerOptions $options, $lastRestart)
{
    $this->sleep($options->sleep > 0 ? $options->sleep : 1);

    $this->stopIfNecessary($options, $lastRestart);
}

/**
 * Stop the process if necessary.
 *
 * @param  \Illuminate\Queue\WorkerOptions  $options
 * @param  int  $lastRestart
 */
protected function stopIfNecessary(WorkerOptions $options, $lastRestart)
{
    if ($this->shouldQuit) {
        $this->kill();
    }

    if ($this->memoryExceeded($options->memory)) {
        $this->stop(12);
    } elseif ($this->queueShouldRestart($lastRestart)) {
        $this->stop();
    }
}
```

worker的暂停实际上就是睡眠一段时间，这里在它醒来以后，还要判断要不要停止worker，因为有可能在它睡眠的过程中，用户
发出了queue:restart命令，如果这时候不处理，那么就要等到进入下一次循环前再处理了。

接下来就从队列中获取任务

```php
//src/Illuminate/Queue/Worker.php

/**
 * Get the next job from the queue connection.
 *
 * @param  \Illuminate\Contracts\Queue\Queue  $connection
 * @param  string  $queue
 * @return \Illuminate\Contracts\Queue\Job|null
 */
protected function getNextJob($connection, $queue)
{
    try {
        foreach (explode(',', $queue) as $queue) {
            if (! is_null($job = $connection->pop($queue))) {
                return $job;
            }
        }
    } catch (Exception $e) {
        $this->exceptions->report($e);

        $this->stopWorkerIfLostConnection($e);
    } catch (Throwable $e) {
        $this->exceptions->report($e = new FatalThrowableError($e));

        $this->stopWorkerIfLostConnection($e);
    }
}
```

它从给定的connection中，依次从每个queue中获取任务，只要获取到一个任务，就马上返回

然后就是安装一个定时器处理函数以及设置一个定时器，这是为了防止任务执行超过设定的时间

```php
//src/Illuminate/Queue/Worker.php

/**
 * Register the worker timeout handler.
 *
 * @param  \Illuminate\Contracts\Queue\Job|null  $job
 * @param  \Illuminate\Queue\WorkerOptions  $options
 * @return void
 */
protected function registerTimeoutHandler($job, WorkerOptions $options)
{
    // We will register a signal handler for the alarm signal so that we can kill this
    // process if it is running too long because it has frozen. This uses the async
    // signals supported in recent versions of PHP to accomplish it conveniently.
    pcntl_signal(SIGALRM, function () {
        $this->kill(1);
    });

    pcntl_alarm(
        max($this->timeoutForJob($job, $options), 0)
    );
}
```

接下来如果获取到任务的话，就开始处理任务，否则就睡眠一段时间后，再判断worker是不是应该停下来。

```php
//src/Illuminate/Queue/Worker.php

/**
 * Process the given job.
 *
 * @param  \Illuminate\Contracts\Queue\Job  $job
 * @param  string  $connectionName
 * @param  \Illuminate\Queue\WorkerOptions  $options
 * @return void
 */
protected function runJob($job, $connectionName, WorkerOptions $options)
{
    try {
        return $this->process($connectionName, $job, $options);
    } catch (Exception $e) {
        $this->exceptions->report($e);

        $this->stopWorkerIfLostConnection($e);
    } catch (Throwable $e) {
        $this->exceptions->report($e = new FatalThrowableError($e));

        $this->stopWorkerIfLostConnection($e);
    }
}

/**
 * Process the given job from the queue.
 *
 * @param  string  $connectionName
 * @param  \Illuminate\Contracts\Queue\Job  $job
 * @param  \Illuminate\Queue\WorkerOptions  $options
 * @return void
 *
 * @throws \Throwable
 */
public function process($connectionName, $job, WorkerOptions $options)
{
    try {
        // First we will raise the before job event and determine if the job has already ran
        // over its maximum attempt limits, which could primarily happen when this job is
        // continually timing out and not actually throwing any exceptions from itself.
        $this->raiseBeforeJobEvent($connectionName, $job);

        $this->markJobAsFailedIfAlreadyExceedsMaxAttempts(
            $connectionName, $job, (int) $options->maxTries
        );

        // Here we will fire off the job and let it process. We will catch any exceptions so
        // they can be reported to the developers logs, etc. Once the job is finished the
        // proper events will be fired to let any listeners know this job has finished.
        $job->fire();

        $this->raiseAfterJobEvent($connectionName, $job);
    } catch (Exception $e) {
        $this->handleJobException($connectionName, $job, $options, $e);
    } catch (Throwable $e) {
        $this->handleJobException(
            $connectionName, $job, $options, new FatalThrowableError($e)
        );
    }
}
```

任务的运行实际上就是调用它的`fire`方法，但是在这之前还要判断拿到的任务是不是已经到达最大运行次数，如果
达到的话，那么它就被标记为失败，然后由`handleJobException`接管。

我们再来看一下任务的`fire`方法实现，所有任务都是继承自`\Illuminate\Queue\Jobs\Job`这个类

```php
//src/Illuminate/Queue/Jobs/Job.php

/**
 * Fire the job.
 *
 * @return void
 */
public function fire()
{
    $payload = $this->payload();

    list($class, $method) = JobName::parse($payload['job']);

    ($this->instance = $this->resolve($class))->{$method}($this, $payload['data']);
}
```

它拿到任务中payload部分的数据，然后拿到payload中的job字段。如果推入任务队列的任务是一个对象，那么这个job
字段对应的值就是"Illuminate\Queue\CallQueuedHandler@call"，如果是任务的类名称，那么它对应的就是任务的类名称。

这里还要注意一点就是`JobName::parse`

```php
//src/Illuminate/Queue/Jobs/JobName.php

/**
 * Parse the given job name into a class / method array.
 *
 * @param  string  $job
 * @return array
 */
public static function parse($job)
{
    return Str::parseCallback($job, 'fire');
}
```

如果你传递的是任务的类名称，那么这个类中就必须包含一个`fire`公共方法。和传递类对象相比，通过传递类名称来达到
异步执行的方法是比较低级(low level)的，它就仅仅调用这个类的`fire`方法，而`Illuminate\Queue\CallQueuedHandler@call`
则实现了更多的特性。


```php
//src/Illuminate/Queue/CallQueuedHandler.php

/**
 * Handle the queued job.
 *
 * @param  \Illuminate\Contracts\Queue\Job  $job
 * @param  array  $data
 * @return void
 */
public function call(Job $job, array $data)
{
    try {
        $command = $this->setJobInstanceIfNecessary(
            $job, unserialize($data['command'])
        );
    } catch (ModelNotFoundException $e) {
        return $this->handleModelNotFound($job, $e);
    }

    $this->dispatcher->dispatchNow(
        $command, $this->resolveHandler($job, $command)
    );

    if (! $job->hasFailed() && ! $job->isReleased()) {
        $this->ensureNextJobInChainIsDispatched($command);
    }

    if (! $job->isDeletedOrReleased()) {
        $job->delete();
    }
}
```

首先是处理eloquent模型丢失的问题，注入到任务中的模型在序列化的时候并不是把整个模型序列化，而只是提取它的主键，在worker
中反序列化以后，再根据这个主键从数据库中把模型拿出来，我们要考虑如果在这个时间区间中，对应的数据库记录被删除了，应该
怎么处理。Laravel提供了两种方法，一种是直接将这个任务删除，另外一种就是把这个任务标记为失败。

另外就是支持处理任务链，如果任务链中有依赖当前执行任务的，那么在当前任务执行成功以后，就开始执行任务链中的下一个任务，实际上
就是dispatch下一个任务。