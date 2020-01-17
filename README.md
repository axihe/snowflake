# Snowflake

## 前置条件

- **Node10.X** 及以上的版本
    - 代码中使用 `Bigint` 实现，该类型在 **Node10.X** 版本才开始支持，推荐使用 `Node.js 12.14.0`
- 服务器的系统时间不允许回拨

## 安装

```
npm install snow-flake -save
```

## 使用方法


```
var idWorker = require("../index");
let id = idWorker.nextId();//45386714578944n
```


worker_id 是 `0-31`的机器ID（用来配置分布式的多机器，最多支持32个机器）

datacenter_id 是 `0-31`的数据ID（用来配置某个机器下面的某某服务，每台机器最多支持32个服务）

之所以需要手动配置，是防止使用者误操作（没有做配置，会报错提醒你设置）；

配置完以后，如下操作，返回是Bigint类型的ID

关于Bigint类型的说明，请参考：https://www.axihe.com/api/js-es/ob-bigint/overview.html

## 测试

```
npm run test
```


## 雪花算法

网上搜到的方案基本是按照推特的方案（10位的数据机器位分成 5位机器ID + 5位数据ID ），目前代码按照这个方案来做的；

方案的实现参考：https://github.com/twitter-archive/snowflake

## 名词说明

Twitter_Snowflake

SnowFlake的结构如下(每部分用-分开):
```
0 - 0000000000 0000000000 0000000000 0000000000 0 - 00000 - 00000 - 000000000000 
A-|--------------------B--------------------------|-------C-------|------D------|
```

- A区：1位标识，由于long基本类型在Java中是带符号的，最高位是符号位，正数是0，负数是1，所以id一般是正数，最高位是0
- B区：41位时间截(毫秒级)，注意，41位时间截不是存储当前时间的时间截，而是存储时间截的差值（当前时间截 - 开始时间截)得到的值， 这里的的开始时间截，一般是我们的id生成器开始使用的时间，由我们程序来指定的（如下下面程序IdWorker类的startTime属性）。41位的时间截，可以使用69年，

    ```
    (1n << 41n) / (1000n * 60n * 60n * 24n * 365n) = 69n
    ```

- C区：10位的数据机器位，可以部署在1024个节点，包括5位datacenterId和5位workerId（2^52^5 = 1024）

- D区：12位序列，毫秒内的计数，12位 的计数顺序号支持每个节点每毫秒(同一机器，同一时间截)产生4096个ID序号（2^12=4096）

加起来刚好64位，为一个Long型。

## SnowFlake的优点

* 生成 ID 时不依赖于数据库，完全在内存生成，高性能高可用。
* 容量大，每秒可生成几百万 ID
    - 理论1S内生成的ID数量是 1000*4096 = 4096000（四百零九万六千个）。
    - 实际测试中是每秒180W的ID（MAC Pro 固态硬盘 8G内存）
* 所有生成的 id 整体上按照时间自增排序，后续插入数据库的索引树的时候，性能较高。
* 整个分布式系统内不会产生ID碰撞(由机器ID作区分)

## SnowFlake的缺点

* 依赖于系统时钟的一致性。如果某台机器的系统时钟回拨，有可能造成 ID 冲突，或者 ID 乱序。
* 在启动之前，如果这台机器的系统时间回拨过，那么有可能出现 ID 重复的危险。

## 常见问题

* A:上次生成 ID 的时间戳，是在内存中，如果系统时钟回退后重启，怎么保证
    - 无法保证，只能流程上控制系统时钟不回退。
* A:41 位 `(timestamp - this.twepoch) << this.timestampLeftShift` 超过长整型怎么办？
    - `this.twepoch` 可以设置当前开始使用系统时的时间，可以保证 69 年不超

* A:运行时候遇到下面报错，如何解决

    ```
    /Users/YourPath/id-worker/lib/snowflake.js:10
            this.sequenceBits = 12n;
                                ^^

    SyntaxError: Invalid or unexpected token
        at createScript (vm.js:80:10)
        at Object.runInThisContext (vm.js:139:10)
        at Module._compile (module.js:617:28)
        at Object.Module._extensions..js (module.js:664:10)
        at Module.load (module.js:566:32)
        at tryModuleLoad (module.js:506:12)
        at Function.Module._load (module.js:498:3)
        at Module.require (module.js:597:17)
        at require (internal/module.js:11:18)
        at Object.<anonymous> (/Users/YourPath/id-worker/index.js:1:81)
    ```
    - 是因为使用者的 `Node.js` 版本不支持`Bigint`导致的 , 推荐使用 `Nodejs 12.14.0` 这个长期支持版本(LTS)

## 性能测试


```
生成100W条ID，      时间约      500-560ms
生成180W条ID，      时间约      900-995ms
生成409.6WW条ID，   时间约约    2000-2260ms
```

具体可以自己跑下本项目的test文件