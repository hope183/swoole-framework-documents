&#160;&#160;&#160;&#160;消息服务器使用socket，为避免服务器过载，单台只允许500个socket连接，当一台不够的时候，扩充消息服务器是必然，问题来了，如何让链接在不同消息服务器上的用户可以实现消息发送呢？
&#160;&#160;&#160;&#160;要实现消息互通就必须要让这些消息服务器本身能互通，想了两个方式，一种是消息服务器之间交叉链接，另一种是增加一个特殊的消息服务器，这个消息服务器不对外开放，只负责消息转发和推送。
&#160;&#160;&#160;&#160;**下列测试不考虑防火墙等。仅测试可行性和效率。**

## 测试环境

*   ##### 消息服务器
    
        192.168.0.201 9501 192.168.0.202 9501
        

*   ##### 转发服务器
    
        192.168.0.203 9501
        

*   ##### 公共缓存
    
        Redis 192.168.0.231 6379
        

*   ##### 软件环境
    
        centos 6.5 mini swoole php
        

## 流程图

*   整个流程图如下： 

![enter image description here][1]

*   流程图说明：  
    `client1`可向`client2`或者其他`client`发送消息，并接收其他`client`发送的消息.
    
    `Redis`中保存`client`连接的信息，给每个用户分配唯一的`key`,包括链接的哪台服务器,转发服务器定时检测消息服务器，如消息服务器挂掉，由转发服务器清理掉Redis已经挂掉的所有链接。

*   完整的流程：
    
    1\.`Client1`给`Client2`发送一条消息
    
    2\.`Socket1`接收到消息，根据`key从Redis`取出`Client2`的连接信息，连接在本机，直接推送给`Client2`，流程结束。
    
    3\.如果连接不在本机，把消息推送到转发服务器,由转发服务器把该消息推送给连接所在消息服务器，消息服务器接收消息，推送给`Client2`。
    
    4\.消息发送结束。

## 编码实现

Socket 在socket1上创建一个server.php,内容如下：

    <?php //服务端 
     $serv = new swoole_server("0.0.0.0", 9501);
    
    //redis
    $redis = new \Redis();        
    $redis->connect("192.168.0.231", 6379);
    
    //client
    $proxy = new swoole_client(SWOOLE_TCP | SWOOLE_KEEP);
    $proxy->connect("192.168.0.203", 9501);
    
    $serv->on('start', function($serv) {
        echo "Service:Start...";
    });
    $serv->on('connect', function ($serv, $fd) {
    
    });
    $serv->on('receive', function ($serv, $fd, $from_id, $data) {
        global $redis;
    
        $data = (array) json_decode($data);
        $cmd = $data['cmd'];
    
        switch ($cmd) {
    
            case "login"://登陆
                //保存连接信息
                $save = array(
                    'fd' => $fd,
                    'socket_ip' => "192.168.0.201"
                );
                $redis->set($data['name'], serialize($save));
                break;
    
            case "chat":
                $recv = unserialize($redis->get($data['recv']));
                if ($recv['socket_ip'] != "192.168.0.201") {
                    //需要转发
                    $data['cmd'] = 'forward';
                    $data['recv_ip'] = $recv['socket_ip'];
                    $serv->task(json_encode($data));
                } else {
                    //直接发送
                    $serv->send($recv['fd'], "{$data['send']}给您发了消息：{$data['content']}");
                }
                break;
    
            case "forward"://接收转发消息
                $recv = unserialize($redis->get($data['recv']));
                $serv->send($recv['fd'], "{$data['send']}给您发了消息：{$data['content']}");
    
                break;
        }
        //$serv->send($fd, 'Swoole: ' . $data);
    });
    $serv->on('task', function ($serv, $task_id, $from_id, $data) {
        global $proxy;
        $proxy->send($data);
    });
    
    $serv->on('finish', function ($serv, $task_id, $data) {
    
    });
    $serv->on('close', function ($serv, $fd) {
        echo "Client: Close.\n";
    });
    
    $serv->set(array('task_worker_num' => 4));
    
    $serv->start();
    

在socket2上只需把ip变更一下即可。192.168.0.201变更为192.168.0.202.

Proxy 在转发服务器上建立脚本proxy.php，内容如下：

    $serv = new swoole_server("0.0.0.0", 9501); //服务端
    $serv->on('start', function($serv) {
        echo "Service:Start...";
    });
    $serv->on('connect', function ($serv, $fd) {
    
    });
    $serv->on('receive', function ($serv, $fd, $from_id, $data) {
        global $redis;
        $serv->task($data);
    });
    $serv->on('task', function ($serv, $task_id, $from_id, $data) {
        $forward = (array) json_decode($data);
        $client = new swoole_client(SWOOLE_SOCK_TCP, SWOOLE_SOCK_SYNC);
    
        $client->connect($forward['recv_ip'], 9501);
        unset($forward['recv_ip']);
        $client->send(json_encode($forward));
        $client->close();
    });
    
    $serv->on('finish', function ($serv, $task_id, $data) {
    
    });
    $serv->on('close', function ($serv, $fd) {
        echo "Client: Close.\n";
    });
    
    $serv->set(array('task_worker_num' => 4));
    
    $serv->start();
    

## 测试

**注意开启顺序**  
1\.开启转发服务器php proxy.php  
2\.分别开启socket服务器php server.php

![enter image description here][2]

可以在转发服务器上看到两个消息服务器已经连接  
3\.开始测试，分别打开两个telnet,连接两个消息服务器，发送消息测试：  
登陆

![enter image description here][3]

发送消息测试

![enter image description here][4]

消息成功接收。

基于强大的`swoole`扩展，让php高效的实现这些成为可能，目前消息服务器到转发服务器是长连接，转发服务器到消息服务器是短连接，存在性能瓶颈，也浪费了连接资源。下一步改造成长连接，消息服务器的client使用异步。

 [1]: http://blog.molibei.com/wp-content/uploads/2014/10/dispersed-socket.jpg
 [2]: http://blog.molibei.com/wp-content/uploads/2014/10/socket-accept.jpg
 [3]: http://blog.molibei.com/wp-content/uploads/2014/10/socket-login-201.jpg
 [4]: http://blog.molibei.com/wp-content/uploads/2014/10/socket-msg-recv.jpg
