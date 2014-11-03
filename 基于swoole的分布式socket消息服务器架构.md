昨天初步实现了不同聊天服务器消息互发,但效率不好，且有性能瓶颈，Swoole的作者韩天峰提醒可以使用长连接转发消息，今天测试了一遍，效果不错。

上一篇的逻辑讲的不是很清楚，这次清楚的描述下：现有`A`、`B`两台消息服务器，`用户1`连接在服务器`A`，`用户2`连接在服务器`B`，而`用户1`需要给`用户2`发送消息，由于`Socket`连接在不同服务器，无法直接互通。现有两种办法解决：

**1. 消息服务器进行串联**

*   **优点**
    
    不用单独增加服务器来做`proxy`，服务器与服务器之间可酌情采用**长连接/短链接**。
    
    消息少一次传输，可以减少网络消耗。

*   **缺点**
    
    每台服务器必须和剩下的所有服务器建立连接。如果其中任意一个连接断掉，都会影响服务，服务器多的时候也不利于**监控服务状态**。

**2. 单独使用服务器做转发 并联**

*   **优点**
    
    消息服务器专注于消息收发，不再发起`Client`，专注于服务端本身。
    
    转发服务器发起`Client`连接所有消息服务器，利于监控服务器状态，而且`转发服务器`宕机不影响`消息服务器`，对用户的影响时间短。
    
    可以使用`keepalived`来给`转发服务器`做高可用，或者把转发服务器做成`连接池`，进一步减轻单台服务器的压力。
    
    更可以把一个转发服务器和几台消息服务器做成一个节点，再把这些节点组合起来，可以满足超大的即时通讯服务。

*   **缺点**
    
    由于消息通过转发服务器，会比直接串联多一次数据传输，增加了网络消耗。
    
    暂时没想到其他的。

## 开始搭建

*   **流程图**
    
    ![enter image description here][1]

*   **环境**
    
    所有测试在虚拟机上进行：
    
    *   **主机**
        
            os  Ubuntu14.10 64位
            cpu 4 core
            mem 16 G
            soft Virtual box           
    
    *   **虚拟机**
        
            centos 6.5 mini swoole php//
            192.168.0.201   //socket1 消息服务器
            192.168.0.202   //socket2 消息服务器
            192.168.0.203   //proxy   转发服务器
            192.168.0.231   //Redis           
    
    *   **客户端**
        
            telnet 
            
## 编码

*   **消息服务器**
    
    由于消息服务器不再发起`客户端连接`,所以代码要精简很多。直接上测试代码：
    
        <?php
        
        /**
         * @filename server.php
         * @encoding UTF-8
         * @author CaiXin Liao <king.liao@qq.com>
         * @link http://www.51zhima.com
         * @copyright 51zhima@CopyRight
         * @datetime 2014-10-28 15:09:54
         * @version 1.0
         * @Description 
         */
        //服务端
        $serv = new swoole_server("0.0.0.0", 9501);
        
        //redis
        $redis = new \Redis();
        $redis->connect("192.168.0.231", 6379);
        
        //Server
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
                    if ($recv['socket_ip'] != "192.168.0.201") {//发消息给proxy
                       $proxy = unserialize($redis->get('192.168.0.201_router'));
                        //需要转发
                        $data['recv_ip'] = $recv['socket_ip'];
                        $serv->send($proxy['fd'], json_encode($data));
                    } else {
                        //直接发送
                        $serv->send($recv['fd'], "{$data['send']}给您发了消息：{$data['content']}\n");
                    }
                    break;
            }
        });
        $serv->on('close', function ($serv, $fd) {
            echo "Client: Close.\n";
        });
        
        $serv->start();
        
    不同的消息服务器只需修改ip即可。

*   **转发服务器**
    
    转发服务器需要遍历连接所有的消息服务器，并且采用`异步`的方式，测试代码：
    
        <?php
        
        /**
         * @filename proxy.php
         * @encoding UTF-8
         * @author CaiXin Liao <king.liao@qq.com>
         * @link http://www.51zhima.com
         * @copyright 51zhima@CopyRight
         * @datetime 2014-10-29 15:50:37
         * @version 1.0
         * @Description 
         */
        
        $clients = array();
        $servers = array(
            '192.168.0.201',
            '192.168.0.202',
        );
        for ($i = 0; $i < count($servers); $i++) {
            $clients[$servers[$i]] = new swoole_client(SWOOLE_SOCK_TCP, SWOOLE_SOCK_ASYNC);
            $clients[$servers[$i]]->remote_ip = $servers[$i];
            $clients[$servers[$i]]->on("connect", function(swoole_client $cli) {
                $data = array(
                    'cmd'=>'login',
                    'name'=>$cli->remote_ip . '_router',
                );
                $cli->send(json_encode($data));
                echo $cli->remote_ip . " Connect Success \n";
            });
        
            $clients[$servers[$i]]->on("receive", function (swoole_client $cli, $data) {
                $msg = (array) json_decode($data);
                $remote_ip = $msg['recv_ip'];
                unset($msg['recv_ip']);
                global $clients;
                $clients[$remote_ip]->send(json_encode($msg));
            });
            $clients[$servers[$i]]->on("error", function(swoole_client $cli) {
                echo "{$cli->remote_ip} error\n";
            });
        
            $clients[$servers[$i]]->on("close", function(swoole_client $cli) {
                echo "{$cli->remote_ip} Connection close\n";
            });
            $clients[$servers[$i]]->connect($servers[$i], 9501, 0.5);
        }

## 测试

  开启所有的虚拟机，同步脚本，这里在`proxy脚本`里没有做监控和连接状态检测，这里只做简单的消息发送测试。

  * **开启消息服务器1：**

    ![enter image description here][2]

  * **开启消息服务器2：**

    ![enter image description here][3]

  * **开启转发服务器**

    ![enter image description here][4]

  * **telnet登陆两个用户到socket1和socket2**

    ![enter image description here][5]

  * **现在用户2给用户1发送消息测试**

    ![enter image description here][6]

    成功收到消息，后面就是进行针对性的优化了。

  * **消息格式**

        {"cmd":"login","name":"zhima201"} //登陆消息
        {"cmd":"chat","send":"zhima201","recv":"zhima202","content":"how are youe"}//文本消息

 [1]: http://blog.molibei.com/wp-content/uploads/2014/10/socket-dispersed-v2.jpg
 [2]: http://blog.molibei.com/wp-content/uploads/2014/10/socket-msg1-start.jpg
 [3]: http://blog.molibei.com/wp-content/uploads/2014/10/socket-msg2-start.jpg
 [4]: http://blog.molibei.com/wp-content/uploads/2014/10/socket-proxy-start.jpg
 [5]: http://blog.molibei.com/wp-content/uploads/2014/10/socket-login-user.jpg
 [6]: http://blog.molibei.com/wp-content/uploads/2014/10/socket-user2-send-user1.jpg
