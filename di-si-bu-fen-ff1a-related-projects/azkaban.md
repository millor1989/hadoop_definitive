### [Azkaban](https://azkaban.readthedocs.io/en/latest/)

轻量级的工作流框架，分布式的工作流管理器。

Azkaban 由两 Web 和 Executor 两部分组成，Web 负责管理和调度工作流，Executor 则是工作流的具体执行者。一个 Web 可以对应多个 Executor.

#### 1、工作流

Azkaban 2.0 开始使用 Azkaban Flow 2.0，虽然目前还支持 Flow 1.0，但是未来的将会不提供 Flow 1.0 的支持。

使用 Flow 2.0 需要创建一个 `flow-name.project` 文件，并在该文件中声明工作流 Flow 2.0：

```yaml
azkaban-flow-version: 2.0
```

然后，创建工作流文件 `flow-name.flow`，文件使用 YAML 语法，一个简单的命令执行 job 例子：

```yaml
nodes:
  - name: jobA
    type: command
    config:
      command: echo "This is an echoed text."
```

最后，将两个文件打包为 `zip` 压缩包，通过 Web 上传到对应的 Project 工程中。之后，就可以进行调度或执行。 

#### 2、指定 Executor

在一个 Web 对应多个 Executor 的模式下，flow 执行时使用那个 Executor 是由 Web 来随机指定的。

如果要强制使用某个 Executor 执行 flow，那么需要在调度或执行 flow 是指定参数 `useExecutor:<executor-id>`。

#### 3、工作流全局参数

如果需要为工作流定义全局的参数，需要增加一个 `flow.properties` 文件：

```properties
date=
```

对于需要用到改参数的地方，使用 `${date}` 的方式引用：

```properties
command=sh /data/azkaban/taskFile/jdd_dmp/motoUidBasicStatis/spark-submit.sh ${date}
```

如果 `flow.properties` 中没有给参数具体赋值，可以在执行该工作流是指定该参数。

#### 4、job 失败自动重试

如果需要 job 失败时自动重试，可以在 job 的配置中配置 `retries` 和 `retry.backoff`：

```yaml
nodes:
  - name: job
    type: command
    config:
      command: sh job.sh
      # 重试次数
      retries: 9999
      # 失败重试的间隔（毫秒）：20秒钟
      retry.backoff: 20000
```

#### 5、一个 job 执行多个命令

使用 `command.<num>` 的形式配置多个 command，则 job 会执行多个命令：

```yaml
nodes:
  - name: jobA
    type: command
    config:
      command: echo "This is an echoed text."
      command.1: sh job1.sh
      command.2: sh job2.sh
```

其中 `command` 是**必须要有**的。

应用场景举例：一个 job 要清洗按天、按周、按月三个时间范围的数据，可以使用多命令 job。

每个命令之间互不影响，命令的执行按照配置命令序号串行执行，只要有一个命令失败，这个 job 就会被认为是 `FAILED`。

#### 6、Azkaban 判断 Shell 执行是否成功

Azkaban 是通过 shell 文件的返回值（即 shell 文件中最后一条 shell 命令的返回值）来判断 job 执行是否成功的。即，如果 shell 文件返回 `0`，Azkaban 就认为 job 是成功的，否则就是失败的。

```shell
exit 255
```

使用 shell 的 `exit` 命令，可以指定 shell 文件的返回值，某些场合可能有用——比如，只需要某个 job 执行但不关心某个 job 是否成功时。

#### 7、Conditional Workflow

有条件的工作流，通过判断条件是否成立，决定工作流是否执行。

**需要 Flow 2.0 的支持、定义在工作流的 YAML 文件中才可以**，Flow 1.0 是无法实现的。

- 支持的操作符有：`==, !=,  >, >=, <, <=, &&, ||, !`

- 支持 job 运行时参数

  用户需要通过 `${jobName:param}` 的形式，将参数值写入到 flow 的运行时环境变量 `$JOB_OUTPUT_PROP_FILE`（对应于一个 Azkaban flow 运行时创建的临时文件，不是 Linux 的环境变量，flow 执行结束文件会被删除） 中：

  ```shell
  echo '{"param1":1}' > $JOB_OUTPUT_PROP_FILE
  ```

- 支持 job 状态宏（macro）

  - **all_done** 对应 job 状态： `FAILED, KILLED, SUCCEEDED, SKIPPED, FAILED_SUCCEEDED, CANCELLED`
  - **all_success / one_success** 对应 job 状态： `SUCCEEDED, SKIPPED, FAILED_SUCCEEDED`
  - **all_failed / one_failed** 对应 job 状态： `FAILED, KILLED, CANCELLED`
  - **一个条件中不允许出现多个 job 状态宏**

一个条件 flow（embedded flow）示例：

```yaml
---
config:
  user.to.proxy: hdfs
nodes:
  - name: end
    type: flow
    dependsOn:
      - job1
      - job2
      - job3
    condition: all_done && ${job1:param1} == 1 && ${job2:param} == 2
    nodes:
      - name: jobB
        type: noop
        dependsOn:
          - jobA

      - name: jobA
        type: command
        config:
          command: echo "hello"

  - name: start
    type: command
    config:
      command: echo "start"

  - name: job1
    type: command
    config:
      command: sh /justATest/1.sh
    dependsOn:
      - start

  - name: job2
    type: command
    config:
      command: sh /justATest/2.sh
    dependsOn:
      - start

  - name: job3
    type: command
    config:
      command: /justATest/3.sh
    dependsOn:
      - start
```

其中 job 配置 `type` 值  `noop` （no operation）表示是一个没有任何操作的 job。

三个 shell 文件内容如下：

```
#1.sh
echo '{"param1":1}' > $JOB_OUTPUT_PROP_FILE

#2.sh
echo '{"param":2}' > $JOB_OUTPUT_PROP_FILE

#3.sh
exit 255
```

尽管 job3 失败，内嵌工作流 `flow:end` 的 `condition` 条件 `all_done && ${job1:param1} == 1 && ${job2:param} == 2` 判断为 `true`，`flow:end` 仍然会执行。

`condition` 不仅可以用于 `flow`，也可以用于 `job`。

