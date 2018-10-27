# 查询构造器

查询构造器主要是提供用于构建SQL语句的fluent接口，它生成SQL以及对应的各个绑定值，最终通过`Connection`类将语句
发送给数据库，然后将返回的结果经过处理以后返回给上层应用。

一条SQL语句可以划分成多个部分来考虑，查询构造器将通过fluent接口接收到的参数分别存入不同的数据结构中，最后由`Grammer`
根据这些数据结构把SQL拼接出来。

在查询构造器中，它定义了一组内部的属性用于存放SQL语句的各个部分

* `$columns`，用于存放要操作的列信息
* `$distinct`，用于表示是否需要去掉重复的行
* `$from`，用于存放要操作的表
* `$joins`，用于存放join语句
* `$wheres`，用于存放where语句
* `$groups`，用于存放group by语句
* `$havings`，用于存放having语句
* `$orders`，用于存放order by语句
* `$limits`，用于存放limit语句
* `$unions`，用于存放union语句

当查询构造器构造完毕以后，就可以调用构造器提供的运行方法，向数据库发送SQL语句。例如在构造完select语句以后，那就可以调用
`get`方法来运行该语句

```php
//src/Illuminate/Database/Query/Builder.php

/**
 * Execute the query as a "select" statement.
 *
 * @param  array  $columns
 * @return \Illuminate\Support\Collection
 */
public function get($columns = ['*'])
{
    $original = $this->columns;

    if (is_null($original)) {
        $this->columns = $columns;
    }

    $results = $this->processor->processSelect($this, $this->runSelect());

    $this->columns = $original;

    return collect($results);
}

/**
 * Run the query as a "select" statement against the connection.
 *
 * @return array
 */
protected function runSelect()
{
    return $this->connection->select(
        $this->toSql(), $this->getBindings(), ! $this->useWritePdo
    );
}

/**
 * Get the SQL representation of the query.
 *
 * @return string
 */
public function toSql()
{
    return $this->grammar->compileSelect($this);
}
```

大概的流程就是先使用`toSql`方法将内部的数据结构转换成SQL语句，完成以后调用`runSelect`向数据库发送SQL语句，返回的结果集经过`$this->processor`
中的`processSelect`方法处理，最终返回结果。