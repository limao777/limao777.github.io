---
title: 基于swoole PHP实现EDP协议服务端
date: 2018-05-11 15:26:13
category: PHP
tags:
- PHP
- EDP
- 协议
---

## 简介
&emsp;&emsp;EDP（Enhanced Device Protocol）协议是一种极简的设备协议，主要用于物联网设备，并且比MQTT还要轻量，其协议文档可以在中移动OneNET官方网站下载。

https://open.iot.10086.cn

&emsp;&emsp;现在主要来讲讲怎么用PHP长连接方式实现一个简易的EDP协议，也是大概两年多前闲着没事自娱自乐做的一个小东西，以期更快地了解、理解这个协议。因当时暂未把QOS纳入该协议且作为大平台成员已知设备间命令会存在安全问题故PHP版本不包含QOS以及设备间命令功能。

## 技术准备
### 一个合适的常驻内存运行的PHP框架
&emsp;&emsp;可以使用原生的PHP Socket，也可以用国外的walkman之类的框架。当然，本篇是用的国产的swoole框架，各有各的优点也各有各的缺点，适合的就是好用的。

### 两个重要PHP内置方法
&emsp;&emsp;hex2bin()与bin2hex()，对这两个方法不很清楚的童鞋可以去php.net官网看看说明

### 一个进制转换方法
&emsp;&emsp;由于世界上最好的语言的超级字符串特性，通常在数据处理的时候需要将16进制与待处理数据进行转换，方法如下:
```PHP
<?php
    /**
     *
     * arr=1，正(小端)，arr=2，反(大端)
     *
     * @param
     *            N byte array $num
     */
    public static function c16to10($num, $arr = 2)
    {
        $res = $num;
        if (is_array($num)) {
            foreach ($num as $p => $v) {
                switch (strtoupper($num[$p])) {
                    case 'A':
                        $num[$p] = 10;
                        break;
                    case 'B':
                        $num[$p] = 11;
                        break;
                    case 'C':
                        $num[$p] = 12;
                        break;
                    case 'D':
                        $num[$p] = 13;
                        break;
                    case 'E':
                        $num[$p] = 14;
                        break;
                    case 'F':
                        $num[$p] = 15;
                        break;
                }
            }
            $res = 0;
            if ($arr == 1) {
                for ($p = 0; $p < count($num, COUNT_NORMAL); $p ++) {
    
                    $res = $res + $num[$p] * pow(16, $p);
                }
            } else {
                for ($p = count($num, COUNT_NORMAL) - 1; $p >= 0; $p --) {
    
                    $res = $res + $num[$p] * pow(16, count($num, COUNT_NORMAL) - 1 - $p);
                }
            }
        }
    
        return $res;
    }
```

## 看文档做解析
### 怎样才能将设备上报的数据转换成方便操作的数据
&emsp;&emsp;有了上面三项准备接下来可以真正开始做事情了，需要做的就是对照EDP文档进行一步一步地一个字节一个字节地解析：
&emsp;&emsp;1.将接收到的数据使用bin2hex转换成字符串
&emsp;&emsp;2.使用c16to10方法转换字符串得到相对更好处理的十进制字符串
&emsp;&emsp;经过以上两步得到的就是数字型字符串了，PHP中当然不区分这些，直接当作数字拿来使用即可

### 处理消息类型（第一个字节）
&emsp;&emsp;根据文档消息头第一个字节为消息类型，bit4为1表示连接请求，bit4与bit5为1表示透传数据，bit7为1表示储存数据，除了这三个位意外其他的都是填充0，那么我们可以根据转换过后的数字来进行判断，如连接请求类型为00010000，通过c16to10方法后就是数字16，同样地，透传则为00110000=48，储存则是10000000=128。再通过if或者switch等方法就可以进行后面的数据处理了。

### 连接鉴权的处理
&emsp;&emsp;从第十一个字节开始，后续分别是两个字节的设备长度，设备长度字节的设备ID号，然后是两个字节的鉴权长度以及鉴权长度字节的鉴权信息，以获取设备id长度为例，代码如下：
```PHP
	//$cmd代表收到的二进制数据，cursor代表要去获取数据的游标
        //获取dev_id长度
        $dev_length = Helper_Arr::c16to10([$cmd[$cursor], $cmd[$cursor+1], $cmd[$cursor+2], $cmd[$cursor+3]]);
        $cursor = $cursor + 4;
```
&emsp;&emsp;最后为了方便处理心跳包，使用memcache设置一个用户识别hash：
```PHP
           $cache = Cache_Memcached::getInstance();
            $c_key = md5('edp_conn' . $fd);
            $cache->set($c_key, $auth_str, 300);
            $c_key = md5('edp_fd' . $fd);
            $cache->set($c_key, $dev_id, 300);
```

### 心跳包的处理
&emsp;&emsp;同之前的鉴权处理，如果从缓存不能获取到了用户识别hash则认为已经断线了（socket超时设置需同缓存时间一致）
```PHP
    public static function dealPing($fd, $cmd){
        $cache = Cache_Memcached::getInstance();
        $c_key = md5('edp_conn' . $fd);
        $res = $cache->get($c_key);
        if(!empty($res)){
            $cache->set($c_key, $res, 300);
            $c_key = md5('edp_fd' . $fd);
            $res = $cache->get($c_key);
            $cache->set($c_key, $res, 300);
            return 'd000';
        }else{
            return '20020002';
        }
        
    }
```

### 完整示例：
* 接入机
&emsp;&emsp;使用CGI的方式执行接入机文件：
```PHP
<?php
if (! empty($_SERVER['REQUEST_URI'])) {
    exit();
}

// set_time_limit(0);
ini_set('memory_limit', '128M');
date_default_timezone_set("Asia/Chongqing");
define("ROOT", realpath(dirname(__FILE__)));
// define("ROOT_APP", realpath(ROOT . "/../app"));

// 加载APPLIB基础bootstrap
require_once (ROOT . '/../libs/AppLib/bootstrap.php');    //该文件主要作用为自动加载相关的包、方法等，因附属框架太复杂此处不放出。如果后续有使用PSR0类型的自动加载方式，自行修改代码手动加载即可

// 开始

$request_server = new requestServer(29876);	//29876为监听的端口

class requestServer
{

    public $url_pre = "";

    private $serv;

    public $log_file = '/tmp/EDP.log';
    
    public function __construct($port)
    {
        $dev_hostnames = array(
            'test-VirtualBox',
        );
        $test_hostnames = array(
            'OneNetTest_node02'
        );
        $current_host_name = php_uname('n');
        if (in_array($current_host_name, $dev_hostnames)) {
            $this->url_pre = '127.0.0.1';
            $mem_connect_addr = '127.0.0.1';
        } elseif (in_array($current_host_name, $test_hostnames)) {
            $this->url_pre = '127.0.0.1';
            $mem_connect_addr = '127.0.0.1';
        } else {
            $this->url_pre = '127.0.0.1';
            $mem_connect_addr = '127.0.0.1';
        }
        //此处根据三种开发环境配置三种不同配置文件，如果只有一种则直接写死即可

        $this->serv = new swoole_server("0.0.0.0", $port);
        
        $this->serv->set(array(
            
            'log_file' => $this->log_file,
            'heartbeat_check_interval' => 280,
            'heartbeat_idle_time' => 300,
            
            // 'user' => 'apache', //监听1024以下端口时以xxx用户的权限来运行
            // 'group' => 'www-data',
            'worker_num' => 8,
            'daemonize' => true,
            'max_request' => 10000,
            'dispatch_mode' => 2
        ));
        $this->serv->on('Connect', array(
            $this,
            'onConnect'
        ));
        $this->serv->on('Receive', array(
            $this,
            'onReceive'
        ));
        $this->serv->on('Close', array(
            $this,
            'onClose'
        ));
        $this->serv->start();
    }

    public function onConnect(swoole_server $request_server, $fd)
    {
        // echo "server: handshake success with fd{$fd}\n";
    }

    public function onReceive(swoole_server $request_server, $fd, $from_id, $data)
    {
        if(empty($cache)){
            $cache = Cache_Memcached::getInstance();
        }
        $cache->set("edp_ip" . $fd,$request_server->connection_info($fd)['remote_ip'], 300);
        $ret_msg = $this->__dealCmd($fd, $data);
        if (! empty($ret_msg)) {
            $request_server->send($fd, hex2bin($ret_msg));
        }
        $deal_code = Service_Edp_Dealcode::dealCode($ret_msg);
        if ($deal_code !== 1) {
            if (! empty($deal_code)) {
                $request_server->send($fd, hex2bin($deal_code));
            }
            $request_server->close($fd);
        }
    }

    public function onClose($ser, $fd)
    {
        // echo "client {$fd} closed\n";
    }

    private function __dealCmd($fd, $cmd)
    {
        $cmd = bin2hex($cmd);
        
        $command = Helper_Arr::c16to10([
            $cmd[0],
            $cmd[1]
        ]);
        
        switch ($command) {
            case 16:
                return Service_Edp_Connect::dealConnect($fd, $cmd);
                break;
            case 128:
                return Service_Edp_Datas::save($fd, $cmd);
                break;
            case 192:
                return Service_Edp_Connect::dealPing($fd, $cmd);
                break;
            default:
                break;
        }
        
        return '400104';
    }

    protected function _log($msg)
    {
        // echo 'log: ' . $msg . "\n";
        file_put_contents($this->log_file, date('Y-m-d H:i:s') . ' - ' . $msg . "\n", FILE_APPEND);
        return;
    }
}
```

* 处理鉴权和心跳
```PHP
class Service_Edp_Connect extends Service_Base
{

    public function __construct()
    {
       
    }
   
    public static function dealConnect($fd, $cmd){
        $cursor = 22;
        
        //获取dev_id长度
        $dev_length = Helper_Arr::c16to10([$cmd[$cursor], $cmd[$cursor+1], $cmd[$cursor+2], $cmd[$cursor+3]]);
        $cursor = $cursor + 4;
        
        //取dev_id
        $temp = "";
        for ($i =0; $i<$dev_length*2;$i++){
            $temp .= $cmd[$cursor+$i] . "";
        }
        $dev_id = hex2bin($temp);
        $cursor = $cursor + $dev_length*2;
        
        //鉴权信息长度
        $auth_length = Helper_Arr::c16to10([$cmd[$cursor], $cmd[$cursor+1], $cmd[$cursor+2], $cmd[$cursor+3]]);
        $cursor = $cursor + 4;
        
        //鉴权信息(api-key)
        $temp = "";
        for ($i =0; $i<$auth_length*2;$i++){
            $temp .= $cmd[$cursor+$i] . "";
        }
        $auth_str = hex2bin($temp);
        $cursor = $cursor + $auth_length*2;
        
        //开始鉴权
        $auth = [
            'dev_id' => $dev_id,
            'api-key' => $auth_str
        ];
        if(Service_Device::checkAuth($auth)){
            $cache = Cache_Memcached::getInstance();
            $c_key = md5('edp_conn' . $fd);
            $cache->set($c_key, $auth_str, 300);
            $c_key = md5('edp_fd' . $fd);
            $cache->set($c_key, $dev_id, 300);
            return '20020000';
        }else{
            return '20020002';
        }
    }
    
    public static function dealPing($fd, $cmd){
        $cache = Cache_Memcached::getInstance();
        $c_key = md5('edp_conn' . $fd);
        $res = $cache->get($c_key);
        if(!empty($res)){
            $cache->set($c_key, $res, 300);
            $c_key = md5('edp_fd' . $fd);
            $res = $cache->get($c_key);
            $cache->set($c_key, $res, 300);
            return 'd000';
        }else{
            return '20020002';
        }
        
    }
  
}
```

* 保存上传数据到kafka
```PHP
class Service_Edp_Datas extends Service_Base
{
    public static function save($fd, $cmd)
    {
        $cursor = 2;
        $cache = Cache_Memcached::getInstance();
        
        // 先分离消息体长度
        for ($i = 0; $i < 4; $i ++) {
            $body_length_fenli = Helper_Arr::c16to10([
                $cmd[$cursor],
                $cmd[$cursor + 1]
            ]) & 0x80;
            $cursor = $cursor + 2;
            if (empty($body_length_fenli)) {
                break;
            }
        }
        
        // 过滤固定选项：标识
        $fix_note = Helper_Arr::c16to10([
            $cmd[$cursor],
            $cmd[$cursor + 1]
        ]);
        $cursor = $cursor + 2;
        
        $note_no = '';
        // 如果有目标或源地址，开始
        if ($fix_note == 128 || $fix_note == 192) {
            // 目的或源地址 - 长度
            $addr_length = Helper_Arr::c16to10([
                $cmd[$cursor],
                $cmd[$cursor + 1],
                $cmd[$cursor + 2],
                $cmd[$cursor + 3]
            ]);
            $cursor = $cursor + 4;
            
            // 目的或源地址 - 具体内容
            $temp = "";
            for ($i = 0; $i < $addr_length * 2; $i ++) {
                $temp .= $cmd[$cursor + $i] . "";
            }
            $addr = hex2bin($temp);
            $cursor = $cursor + $addr_length * 2;
            
            if ($fix_note == 192) {
                $note_no = $cmd[$cursor] . $cmd[$cursor + 1] . $cmd[$cursor + 2] . $cmd[$cursor + 3];
                $cursor = $cursor + 4;
            }
        }        // 如果有目标或源地址，结束
        elseif ($fix_note == 0) {
            $c_key = md5('edp_fd' . $fd);
            $addr = $cache->get($c_key);
            if (empty($addr)) {
                $addr = 0;
            }
        } elseif ($fix_note == 64) {
                $note_no = $cmd[$cursor] . $cmd[$cursor + 1] . $cmd[$cursor + 2] . $cmd[$cursor + 3];
                $cursor = $cursor + 4;
        } else {
            return '20020002';
        }
        // 如果没有目标或源地址，结束
        
        // //消息编号（似乎没有）
        // $msg_id = Helper_Arr::c16to10([$cmd[$cursor], $cmd[$cursor+1], $cmd[$cursor+2], $cmd[$cursor+3]]);
        // $cursor = $cursor + 4;
        
        // 数据消息类型
        $data_type = Helper_Arr::c16to10([
            $cmd[$cursor],
            $cmd[$cursor + 1]
        ]);
        $cursor = $cursor + 2;
        // 数据消息 - 长度
        $data_length = Helper_Arr::c16to10([
            $cmd[$cursor],
            $cmd[$cursor + 1],
            $cmd[$cursor + 2],
            $cmd[$cursor + 3]
        ]);
        $cursor = $cursor + 4;
        
        // 数据消息 - 具体内容
        $temp = "";
        for ($i = 0; $i < $data_length * 2; $i ++) {
            $temp .= $cmd[$cursor + $i] . "";
        }
        
        $data_body = hex2bin($temp);
        $cursor = $cursor + $data_length * 2;
        
        // 是二进制数据，特殊处理
        if ($data_type == 2) {
            $data_bin_length = Helper_Arr::c16to10([
                $cmd[$cursor],
                $cmd[$cursor + 1],
                $cmd[$cursor + 2],
                $cmd[$cursor + 3],
                $cmd[$cursor + 4],
                $cmd[$cursor + 5],
                $cmd[$cursor + 6],
                $cmd[$cursor + 7]
            ]);
            $cursor = $cursor + 8;
            
            $temp = "";
            for ($i = 0; $i < $data_bin_length * 2; $i ++) {
                $temp .= $cmd[$cursor + $i] . "";
            }
            $data_bin_body = hex2bin($temp);
            $cursor = $cursor + $data_bin_length * 2;
        }
        
        // --------------EDP包解析结束--------------//
        // 数据鉴权
        $c_key = md5('edp_conn' . $fd);
        $auth_str = $cache->get($c_key);
        if (! empty($auth_str)) {
            
            $kafka_array = [];
            $kafka_array['datastreams'] = [];
            
            switch ($data_type) {
                case 1:
                    $kafka_array = json_decode($data_body, TRUE);
                    break;
                case 2:
                    
                    // 二进制数据
                    $input_data = json_decode($data_body, TRUE);
                    if (empty($input_data) || ! is_array($input_data)) {
                        return '20020001';
                    }
                    if (is_array($input_data)) {
                        $kafka_array['datastreams'][0]['id'] = $input_data['ds_id'];
                        $kafka_array['datastreams'][0]['datapoints'][0]['at'] = empty($input_data['at']) ? date('Y-m-d\TH:i:s') : $input_data['at'];
                        $kafka_array['datastreams'][0]['datapoints'][0]['value'] = $data_bin_body;
                        $kafka_array['datastreams'][0]['datapoints'][0]['desc'] = $input_data['desc'];
                    }
                    break;
                case 3:
                    $input_data = json_decode($data_body, TRUE);
                    if (empty($input_data) || ! is_array($input_data)) {
                        return '20020001';
                    }
                    if (is_array($input_data)) {
                        $count = 0;
                        foreach ($input_data as $p => $v) {
                            $kafka_array['datastreams'][$count]['id'] = $p;
                            $kafka_array['datastreams'][$count]['datapoints'][0]['at'] = date('Y-m-d\TH:i:s');
                            $kafka_array['datastreams'][$count]['datapoints'][0]['value'] = $v;
                            $count ++;
                        }
                    }
                    break;
                case 4:
                    $input_data = json_decode($data_body, TRUE);
                    if (empty($input_data) || ! is_array($input_data)) {
                        return '20020001';
                    }
                    if (is_array($input_data)) {
                        $count = 0;
                        foreach ($input_data as $p => $v) {
                            $kafka_array['datastreams'][$count]['id'] = $p;
                            if (is_array($input_data[$p])) {
                                foreach ($input_data[$p] as $p2 => $v2) {
                                    $kafka_array['datastreams'][$count]['datapoints'][0]['at'] = $p2;
                                    $kafka_array['datastreams'][$count]['datapoints'][0]['value'] = $v2;
                                }
                            } else {
                                return '20020001';
                            }
                            $count ++;
                        } // foreach inputdata
                    }
                    break;
                case 5:
                    $split1 = mb_substr($data_body, 0, 1);
                    $split2 = mb_substr($data_body, 1, 1);
                    $real_str = mb_substr($data_body, 2, strlen($data_body) - 2);
                    $ds_array = explode($split2, $real_str);
                    if (is_array($ds_array)) {
                        foreach ($ds_array as $p => $v) {
                            $dp_array = explode($split1, $v);
                            if (count($dp_array, COUNT_NORMAL) == 2) {
                                $kafka_array['datastreams'][$p]['id'] = $dp_array[0];
                                $kafka_array['datastreams'][$p]['datapoints'][0]['at'] = date('Y-m-d\TH:i:s');
                                $kafka_array['datastreams'][$p]['datapoints'][0]['value'] = $dp_array[1];
                            } elseif (count($dp_array, COUNT_NORMAL) == 1) {
                                $kafka_array['datastreams'][$p]['id'] = $p;
                                $kafka_array['datastreams'][$p]['datapoints'][0]['at'] = date('Y-m-d\TH:i:s');
                                $kafka_array['datastreams'][$p]['datapoints'][0]['value'] = $dp_array[0];
                            } else {
                                $kafka_array['datastreams'][$p]['id'] = $dp_array[0];
                                $kafka_array['datastreams'][$p]['datapoints'][0]['at'] = $dp_array[1];
                                $kafka_array['datastreams'][$p]['datapoints'][0]['value'] = $dp_array[2];
                            }
                        }
                    } // if ds is array
                    break;
            } // switch
            $auth = [
                'dev_id' => $addr,
                'api-key' => $auth_str
            ];
            $ip = $cache->get("edp_ip" . $fd);
            if (! Helper_Kafka::sendToDp($auth, $kafka_array, TRUE, $ip)) {
                return '20020001';
            } else {
                if(empty($note_no)){
                return '900440' . $note_no . '00';
                }else{
                    return '';
                }
            }
        } else {
            return '20020002';
        }
    }
}
```

* 相关的其他支持类
```PHP
class Service_Edp_Dealcode extends Service_Base
{

    public static function dealCode($code)
    {
        $code = strtolower($code);
        if (strlen($code) > 10) {
            return 1;
        } elseif ($code == 'd000') {
            return 1;
        } elseif ($code == '20020000') {
            return 1;
        } elseif ($code == '') {
            return 1;
        } elseif ($code == '20020002') {
            return '400120';
        } else {
            return '400101';
        }
    }
}
```

```PHP
class Cache_Memcached
{
    protected static $_objList;
    private $_memcache;
    private $_config_name;
    

    private function __construct($config_name)
    {
        $this->_config_name = $config_name;
    }

    
    private function init()
    {

        $memcacheHosts = Config::get($this->_config_name);
        $memcacheHosts = explode(',', $memcacheHosts);

        if (empty($memcacheHosts)) {
            return FALSE;
        } else {
            $this->_memcache = new Memcache;
            foreach ($memcacheHosts as $m) {
                list($host, $port) = explode(':', $m);
                $this->_memcache->addServer($host, $port);
            }
        }
    }
    
    /**
     * 获取一个Cache_Memcache实例
     *
     * @param integer $clusterId cluster id
     * @return object
     */
    public static function getInstance($config_name = 'DefaultMemcacheServers')
    {
        if (empty(self::$_objList[$config_name])) {
            $obj = new self($config_name);
            $obj->init();
            self::$_objList[$config_name] = &$obj;
        }

        return self::$_objList[$config_name];
    }
    
    /**
     * 获取缓存的静态方法
     *
     * @param string|array $key
     * @param integer $clusterId cluster id
     * @return mixed
     */
    public static function sGet($key)
    {
        $instance = self::getInstance();
        return $instance->get($key);
    }
    
    /**
     * 设置缓存的静态方法
     *
     * @param string $key
     * @param mixed $val
     * @param integer $expires 过期时间（秒），0为永不过期
     * @param integer $clusterId cluster id
     * @return boolean
     */
    public static function sSet($key, $val, $compress = MEMCACHE_COMPRESSED, $expires = 0)
    {
        $instance = self::getInstance();
        return $instance->set($key, $val, $compress, $expires);
    }
    
    
    /**
     * 删除缓存数据的静态方法
     * 
     * @param string $key
     * @param integer $clusterId
     * @return boolean
     */
    public static function sDelete($key)
    {
        $instance = self::getInstance();
        return $instance->delete($key);    
    }
    
    /**
     * 增加缓存的静态方法
     * 
     * @param string $key
     * @param mixed $val
     * @param integer $expires
     * @param integer $clusterId
     * @return boolean
     */
    public static function sAdd($key, $val, $compress = MEMCACHE_COMPRESSED, $expires = 0)
    {
        $instance = self::getInstance();
        return $instance->add($key, $val, $compress, $expires);
    }
    
    /**
     * 获取缓存数据
     *
     * @param string $key
     * @return mixed 
     */
    public function get($key)
    {
        if (empty($this->_memcache)) {
            if ( ! $this->init()) {
                return FALSE;
            }
        }

        $res = $this->_memcache->get($key);
        return $res;    
    }
    
    /**
     * 写入缓存
     *
     * @param string  $key  数据对应的键名
     * @param mixed   $val  数据
     * @param integer $expires 缓存的时间（秒），设置为0表示永不过期
     * @return boolean
     */
    public function set($key, $val, $expires = 0)
    {
        if (empty($this->_memcache)) {
            if ( ! $this->init()) {
                return FALSE;
            }
        }
        if (is_numeric($val)) {
            $val = (string) $val;
        }

        $ret = $this->_memcache->set($key, $val, 0, $expires);  //兼容以前程序，直接写0
        
        return $ret;
    }
    
    /**
     * 删除缓存
     *
     * @param string $key 数据的键名
     * @return boolean
     */
    public function delete($key)
    {
        if (empty($this->_memcache)) {
            if ( ! $this->init()) {
                return FALSE;
            }
        }
        
        $ret = $this->_memcache->delete($key, 0);

        return $ret;
    }
    
    /**
     * 增加item的值
     *
     * @param string $key
     * @param integer $val
     * @return boolean
     */
    public function increment($key, $val = 1)
    {
        if (empty($this->_memcache)) {
            if ( ! $this->init()) {
                return FALSE;
            }
        }

        $ret = $this->_memcache->increment($key, $val);
        return $ret;
    }
    
    /**
     * 写入缓存当且仅当$key对应缓存不存在的时候
     *
     * @param string $key
     * @param mixed  $val
     * @param integer $expires
     * @return boolean
     */
    public function add($key, $val, $compress = MEMCACHE_COMPRESSED, $expires = 0)
    {
        if (empty($this->_memcache)) {
            if ( ! $this->init()) {
                return FALSE;
            }
        }

        $ret = $this->_memcache->add($key, $val, $compress, $expires);
        return $ret;
    }
    
    /**
     * 刷新
     *
     * @return boolean
     */
    public function flush()
    {
        if (empty($this->_memcache)) {
            if ( ! $this->init()) {
                return FALSE;
            }
        }

        $ret = $this->_memcache->flush();
        return $ret;
    }
}
```

```PHP
class Helper_Kafka
{
    
    private static $__kafka_addr = '127.0.0.1:9092';
    public static $error = '';
    
    public static function sendToDp($auth, $input_data, $check_data = TRUE, $ip = NULL, $topic = 'dp_test')
    {
        
        if(!is_array($auth)){
            self::$error = 'auth mot match';
            return FALSE;
        }
        
        $check_auth = Service_Device::checkAuth($auth);
        if(!$check_auth){
            self::$error = 'auth failed';
            return FALSE;
        }
        $check_dp = Service_Datapoints::checkPostDp($input_data);
        $input_kafka_array = [];
        
        if(empty($ip)){
           $input_kafka_array['ip'] = Service_Http::getClientIP();
        }else{
            $input_kafka_array['ip'] = $ip;
        }
        $input_kafka_array['auth'] = $auth;
        $input_kafka_array['data'] = $input_data;
        $input_kafka = json_encode($input_kafka_array, JSON_UNESCAPED_UNICODE);
        if($check_dp){
            self::storeToLocal($auth['dev_id'], $input_data);
            $kafka = new Kafka(self::$__kafka_addr);
            $res = $kafka->produce($topic, $input_kafka);
            $kafka = NULL;
            return TRUE;
        }else{
            self::$error = Service_Datapoints::$error;
            return FALSE;
        }
    }
    
    public static function storeToLocal($dev_id, $input_data){
        $cache = Cache_Memcached::getInstance();
        $cache->set($dev_id,$input_data, 80000*30);
    }
}
```