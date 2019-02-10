# 队列

这里的队列指的是实现了`\Illuminate\Contracts\Queue\Queue`接口的类，它主要的作用就是在对应的
消息中间件服务中实现队列的功能

```php
//src/Illuminate/Contracts/Queue/Queue.php

interface Queue
{
    /**
     * Get the size of the queue.
     *
     * @param  string  $queue
     * @return int
     */
    public function size($queue = null);

    /**
     * Push a new job onto the queue.
     *
     * @param  string|object  $job
     * @param  mixed   $data
     * @param  string  $queue
     * @return mixed
     */
    public function push($job, $data = '', $queue = null);

    /**
     * Push a new job onto the queue.
     *
     * @param  string  $queue
     * @param  string|object  $job
     * @param  mixed   $data
     * @return mixed
     */
    public function pushOn($queue, $job, $data = '');

    /**
     * Push a raw payload onto the queue.
     *
     * @param  string  $payload
     * @param  string  $queue
     * @param  array   $options
     * @return mixed
     */
    public function pushRaw($payload, $queue = null, array $options = []);

    /**
     * Push a new job onto the queue after a delay.
     *
     * @param  \DateTimeInterface|\DateInterval|int  $delay
     * @param  string|object  $job
     * @param  mixed   $data
     * @param  string  $queue
     * @return mixed
     */
    public function later($delay, $job, $data = '', $queue = null);

    /**
     * Push a new job onto the queue after a delay.
     *
     * @param  string  $queue
     * @param  \DateTimeInterface|\DateInterval|int  $delay
     * @param  string|object  $job
     * @param  mixed   $data
     * @return mixed
     */
    public function laterOn($queue, $delay, $job, $data = '');

    /**
     * Push an array of jobs onto the queue.
     *
     * @param  array   $jobs
     * @param  mixed   $data
     * @param  string  $queue
     * @return mixed
     */
    public function bulk($jobs, $data = '', $queue = null);

    /**
     * Pop the next job off of the queue.
     *
     * @param  string  $queue
     * @return \Illuminate\Contracts\Queue\Job|null
     */
    public function pop($queue = null);

    /**
     * Get the connection name for the queue.
     *
     * @return string
     */
    public function getConnectionName();

    /**
     * Set the connection name for the queue.
     *
     * @param  string  $name
     * @return $this
     */
    public function setConnectionName($name);
}
```

我们主要关注`push`，`pop`这两个方法，它们是队列中最根本的两个方法。`push`负责把任务透过连接，推入到对应的消息中间件中，
而`pop`则相反，从消息中间件中获取任务。

```php
//src/Illuminate/Queue/DatabaseQueue.php

/**
 * Push a new job onto the queue.
 *
 * @param  string  $job
 * @param  mixed   $data
 * @param  string  $queue
 * @return mixed
 */
public function push($job, $data = '', $queue = null)
{
    return $this->pushToDatabase($queue, $this->createPayload($job, $data));
}

/**
 * Push a raw payload to the database with a given delay.
 *
 * @param  string|null  $queue
 * @param  string  $payload
 * @param  \DateTimeInterface|\DateInterval|int  $delay
 * @param  int  $attempts
 * @return mixed
 */
protected function pushToDatabase($queue, $payload, $delay = 0, $attempts = 0)
{
    return $this->database->table($this->table)->insertGetId($this->buildDatabaseRecord(
        $this->getQueue($queue), $payload, $this->availableAt($delay), $attempts
    ));
}
```

在`DatabaseQueue`中，`push`就是往数据库中写入一条记录，具体的字段可以看`buildDatabaseRecord`的实现

```php
//src/Illuminate/Queue/DatabaseQueue.php

/**
 * Create an array to insert for the given job.
 *
 * @param  string|null  $queue
 * @param  string  $payload
 * @param  int  $availableAt
 * @param  int  $attempts
 * @return array
 */
protected function buildDatabaseRecord($queue, $payload, $availableAt, $attempts = 0)
{
    return [
        'queue' => $queue,
        'attempts' => $attempts,
        'reserved_at' => null,
        'available_at' => $availableAt,
        'created_at' => $this->currentTime(),
        'payload' => $payload,
    ];
}
```

其中`queue`代表的是队列名称，在一个连接中，支持多个队列，这个名称就是用来区分同一个连接的不同队列。`attempts`用来
记录该任务是第几次执行，Laravel的队列支持将执行失败的任务再重新投入到队列中，一直到它超过设置的最大执行次数或者执行
成功。`reserved_at`表示的是从队列中获取到任务的具体时间，每个任务都有最大的执行时长，当这个任务超时以后，它就能被
再次获取。`available_at`表示任务什么时候能被获取，任务支持延迟执行，实际上就是设置`availabled_at`为当前时间加上
要延迟的秒数，在获取任务时候，只有`available_at`小于等于当前时间的任务才会被获取。`payload`就是具体的任务内容。

下面来看看`createPayload`的实现

```php
//src/Illuminate/Queue/Queue.php

/**
 * Create a payload string from the given job and data.
 *
 * @param  string  $job
 * @param  mixed   $data
 * @return string
 *
 * @throws \Illuminate\Queue\InvalidPayloadException
 */
protected function createPayload($job, $data = '')
{
    $payload = json_encode($this->createPayloadArray($job, $data));

    if (JSON_ERROR_NONE !== json_last_error()) {
        throw new InvalidPayloadException(
            'Unable to JSON encode payload. Error code: '.json_last_error()
        );
    }

    return $payload;
}
```

`payload`实际上就是一段JSON字符串，接下来再看它有什么内容

```php
//src/Illuminate/Queue/Queue.php

/**
 * Create a payload array from the given job and data.
 *
 * @param  string  $job
 * @param  mixed   $data
 * @return array
 */
protected function createPayloadArray($job, $data = '')
{
    return is_object($job)
                ? $this->createObjectPayload($job)
                : $this->createStringPayload($job, $data);
}

/**
 * Create a payload for an object-based queue handler.
 *
 * @param  mixed  $job
 * @return array
 */
protected function createObjectPayload($job)
{
    return [
        'displayName' => $this->getDisplayName($job),
        'job' => 'Illuminate\Queue\CallQueuedHandler@call',
        'maxTries' => $job->tries ?? null,
        'timeout' => $job->timeout ?? null,
        'timeoutAt' => $this->getJobExpiration($job),
        'data' => [
            'commandName' => get_class($job),
            'command' => serialize(clone $job),
        ],
    ];
}

/**
 * Create a typical, string based queue payload array.
 *
 * @param  string  $job
 * @param  mixed  $data
 * @return array
 */
protected function createStringPayload($job, $data)
{
    return [
        'displayName' => is_string($job) ? explode('@', $job)[0] : null,
        'job' => $job, 'maxTries' => null,
        'timeout' => null, 'data' => $data,
    ];
}
```

这里的`$job`参数支持两种类型，一种是对象，也就是通过构造一个job实例，另外一种就是指定job的类名。后面一种比较少用，
和前者不同的是，它还需要有一个`fire`的公共方法，而且不支持在job中指定最长可执行时间以及最多执行次数。


接下来看用来从队列中获取任务的`pop`方法

```php
//src/Illuminate/Queue/DatabaseQueue.php

/**
 * Pop the next job off of the queue.
 *
 * @param  string  $queue
 * @return \Illuminate\Contracts\Queue\Job|null
 * @throws \Exception|\Throwable
 */
public function pop($queue = null)
{
    $queue = $this->getQueue($queue);

    return $this->database->transaction(function () use ($queue) {
        if ($job = $this->getNextAvailableJob($queue)) {
            return $this->marshalJob($queue, $job);
        }

        return null;
    });
}
```

它从数据库中获取一条记录，然后转换为一个`\Illuminate\Contracts\Queue\Job`实例。

```php
//src/Illuminate/Queue/DatabaseQueue.php

/**
 * Get the next available job for the queue.
 *
 * @param  string|null  $queue
 * @return \Illuminate\Queue\Jobs\DatabaseJobRecord|null
 */
protected function getNextAvailableJob($queue)
{
    $job = $this->database->table($this->table)
                ->lockForUpdate()
                ->where('queue', $this->getQueue($queue))
                ->where(function ($query) {
                    $this->isAvailable($query);
                    $this->isReservedButExpired($query);
                })
                ->orderBy('id', 'asc')
                ->first();

    return $job ? new DatabaseJobRecord((object) $job) : null;
}
```

这里有两个要注意的点，首先它应用了写锁，这是因为后续它要更新对应记录的状态，这些状态影响记录能否被获取，如果不应用写锁的话，
那么同一个任务就有可能被多个worker获取到，造成重复执行的问题。另外一点就是获取记录的条件

```php
//src/Illuminate/Queue/DatabaseQueue.php

/**
 * Modify the query to check for available jobs.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @return void
 */
protected function isAvailable($query)
{
    $query->where(function ($query) {
        $query->whereNull('reserved_at')
              ->where('available_at', '<=', $this->currentTime());
    });
}

/**
 * Modify the query to check for jobs that are reserved but have expired.
 *
 * @param  \Illuminate\Database\Query\Builder  $query
 * @return void
 */
protected function isReservedButExpired($query)
{
    $expiration = Carbon::now()->subSeconds($this->retryAfter)->getTimestamp();

    $query->orWhere(function ($query) use ($expiration) {
        $query->where('reserved_at', '<=', $expiration);
    });
}
```

`isAvailable`和`isReservedButExpired`是或的关系。首先是`isAvailable`，这是任务被第一次获取时的情况，而
`isReservedButExpired`就是任务已经被获取过，但是执行超时了，它的条件就是当前时间不能小于上一次获取任务的时间
加上retry_after后的时间，也就是在job在上一次被获取之后的retry_after秒内不能被再次获取。

获取到的记录还需要经过`marshalJob`方法的处理

```php
//src/Illuminate/Queue/DatabaseQueue.php

/**
 * Marshal the reserved job into a DatabaseJob instance.
 *
 * @param  string  $queue
 * @param  \Illuminate\Queue\Jobs\DatabaseJobRecord  $job
 * @return \Illuminate\Queue\Jobs\DatabaseJob
 */
protected function marshalJob($queue, $job)
{
    $job = $this->markJobAsReserved($job);

    return new DatabaseJob(
        $this->container, $this, $job, $this->connectionName, $queue
    );
}

/**
 * Mark the given job ID as reserved.
 *
 * @param  \Illuminate\Queue\Jobs\DatabaseJobRecord  $job
 * @return \Illuminate\Queue\Jobs\DatabaseJobRecord
 */
protected function markJobAsReserved($job)
{
    $this->database->table($this->table)->where('id', $job->id)->update([
        'reserved_at' => $job->touch(),
        'attempts' => $job->increment(),
    ]);

    return $job;
}
```

`marshalJob`除了返回构造的`\Illuminate\Queue\Jobs\DatabaseJob`以外，它还会更新当前记录，主要
就是记录本次获取任务的时间戳以及为`attempts`记录加一。