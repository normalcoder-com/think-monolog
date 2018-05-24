* [项目介绍](#%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D)
* [安装使用](#%E5%AE%89%E8%A3%85%E4%BD%BF%E7%94%A8)
    * [安装](#%E5%AE%89%E8%A3%85)
    * [如何使用](#%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8)
* [自定义](#%E8%87%AA%E5%AE%9A%E4%B9%89)
    * [默认行为](#%E9%BB%98%E8%AE%A4%E8%A1%8C%E4%B8%BA)
    * [示例：](#%E7%A4%BA%E4%BE%8B%EF%BC%9A)
    * [接管TP默认trace行为](#%E6%8E%A5%E7%AE%A1TP%E9%BB%98%E8%AE%A4trace%E8%A1%8C%E4%B8%BA)
* [关于](#%E5%85%B3%E4%BA%8E)
    * [Monolog简介](#Monolog%E7%AE%80%E4%BB%8B)
* [感谢](#%E6%84%9F%E8%B0%A2)



<a name="%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D"></a>

# 项目介绍
ThinkPHP 3.2 集成 Monolog 的扩展组件

---

<a name="%E5%AE%89%E8%A3%85%E4%BD%BF%E7%94%A8"></a>

# 安装使用
[如何安装Composer - composer中文文档](http://www.kancloud.cn/thinkphp/composer)

<a name="%E5%AE%89%E8%A3%85"></a>

### 安装
```
composer require normal/think-monolog:dev-master
```
<a name="%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8"></a>

### 如何使用
安装完成后, 就可以立即在应用的代码中这样使用 Monolog:

```
\normalcoder\Think\Logger::debug('这是一条debug日志');
\normalcoder\Think\Logger::info('这是一条info日志');
\normalcoder\Think\Logger::warn('这是一条warn日志');
\normalcoder\Think\Logger::error('这是一条error日志');
```

> #### 注意:
> 由于 `SHOW_PAGE_TRACE` 设为 `true` 以后, TP不再将trace数据记录到log.
> 
> 也就是说, 在不修改TP源码的情况下想用monolog收集trace数据, TRACE BAR 和 monolog 你只能二选一。
> 
> 为了不影响升级框架, 对框架的功能扩展绝不修改源码。
> 
> 因此, 集成monolog后, 为了能收集到trace数据, 在内部已将 `SHOW_PAGE_TRACE` 设为了 `false`。目前默认为不记录 trace 数据

---

<a name="%E8%87%AA%E5%AE%9A%E4%B9%89"></a>

# 自定义
<a name="%E9%BB%98%E8%AE%A4%E8%A1%8C%E4%B8%BA"></a>

### 默认行为

think-monolog 默认向monolog注册了 StreamHandler, 日志级别为debug, 这就是为什么安装后可以直接使用的原因.

既然我们用monolog, 肯定是为了使用其提供的丰富的 handlers. 而不是为了仅仅在文件中记录日志. 下面将通过一个实例说明如何自定义 monolog


<a name="%E7%A4%BA%E4%BE%8B%EF%BC%9A"></a>

### 示例：

自己建一个行为类, 在这个行为类中完成 monolog 实例的 handlers 和 processors 的添加。

创建 `Common/Behavior/MonologBehavior.class.php` :

```
<?php
namespace Common\Behavior;

use Think\Behavior;
use normalcoder\Think\Logger;
use Monolog\Handler\MongoDBHandler;

class MonologBehavior extends Behavior
{
    public function run( &$params )
    {
        /**
         think-monolog 默认注册的StreamHandler的日志级别为 debug. 
         如果你想改变它的级别或者不想使用StreamHandler, 就需要先取出这个handler.
         假设,我们现在的在生产环境下的日志需求是这样:
            1. 只想在本地文件中记录Error以上级别的日志供常规检查
            2. info 以上的日志向发到外部的 MongoDb 数据库中,供日志监控和分析
            3. 不记录任何debug信息.
        */       
        $logger = Logger::getLogger();
        $stream_handler = $logger->popHandler();  // 取出 StreamHandler 对象
        $stream_handler->setLevel(Logger::ERROR); // 重设其日志级别
        $logger->pushHandler($stream_handler);    // 注册修改后的StreamHandler 对象
        
        $mongodb = new MongoDBHandler(new \Mongo("mongodb://***.***.***.***:27017"), "logs", "prod", Logger::INFO);
        $logger->pushHandler($mongodb); // 文件
    }
}
```

在`Common/Conf/tags.php` 增加一个`app_begin`行为:

```
return array(
    'app_begin' =>array(
        'Common\Behavior\MonologBehavior'
        ),
);
```

<a name="%E6%8E%A5%E7%AE%A1TP%E9%BB%98%E8%AE%A4trace%E8%A1%8C%E4%B8%BA"></a>

#### 接管TP默认trace行为

默认情况, think-monolog 并不会接管ThinkPHP的 trace 逻辑. 二者互不影响.

如果你希望 think-monolog 接管ThinkPHP的trace逻辑, 只需要将 `LOG_TYPE` 配置设为`monolog`.
这样配置以后, `SHOW_PAGE_TRACE` 将强制关闭, 以便monolog完全接管日志工作.

> `注意:`由于ThinkPHP日志格式设计的问题，对日志的格式解析往往不尽人意。在 think-monolog 接管 trace 后，默认情况下不对 trace 进行记录。

如果需要记录 trace，可以在 Monolog.php 中打开查找如下代码，去掉注释即可。取消注释后，你可以像过去一样使用TP的 `trace` 函数记录日志, 所有的trace数据依然是以**一条日志**的形式在请求结束时被monolog记录。

```
if (false !== strpos($log, 'INFO: [ app_begin ] --START--')) { //取消对ThinkPHP运行生命周期 trace Log 的记录
    //$logger->addRecord(Mlogger::EMERGENCY,"\r\n".$log); 
} else {
}
```

如果你希望单独记录一些日志, 依然需要使用 monolog:

```
\normalcoder\Think\Logger::debug('这是一条debug日志');
\normalcoder\Think\Logger::info('这是一条info日志');
\normalcoder\Think\Logger::warn('这是一条warn日志');
\normalcoder\Think\Logger::error('这是一条error日志');
```
> #### 注意: 
> handler的日志级别设置仅对直接通过 monolog 添加的日志有效。无论handler的日志级别如何, trace 日志一定会被无条件记录。

因此, 接管后不建议使用trace函数记录日志。

---

<a name="%E5%85%B3%E4%BA%8E"></a>

# 关于

<a name="Monolog%E7%AE%80%E4%BB%8B"></a>

###Monolog简介

Monolog 是 Laravel,Symfony,Silex 默认集成的日志库, 同时大量其他框架提供了集成扩展。它是一款极为流行的 php log 库, 自带超多handler, 长期维护, 稳定更新。

Monolog可以把你的日志发送到文件，sockets，收件箱，数据库和各种web服务器上。一些特殊的组件可以给你带来特殊的日志策略。同时，它支持以各种方式记录日志: 记录到文件,mail,nosql,mail,irc,firephp,elasticsearch服务器....

> #### 相关Github仓库：
> 
> Monolog: <https://github.com/Seldaek/monolog>
> 
> Monolog docs: <https://github.com/Seldaek/monolog/tree/master/doc>

---

<a name="%E6%84%9F%E8%B0%A2"></a>

# 感谢

本扩展是基于 [snowair/think-monolog](https://github.com/snowair/think-monolog) 修改而来，再次感谢 [@snowair](https://github.com/snowair) 。



