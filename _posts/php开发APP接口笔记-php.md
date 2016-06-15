---
title: php开发APP接口笔记
date: 2016-06-10 16:22:21
tags: [php,服务器端]
---

# app接口开发记录

## 接口返回数据格式

* xml

```
xml的节点可以自定义不想html的标签，不能自定义。
```

* json

```
json_encode($arr); //json数据，json数据只接受utf-8数据,其他编码的字符串都会返回null
```

## json封装通信数据

```
<?php

/**
* 
*/
class ResponseJson
{
	/*
	*按json方式输出通信数据
	*@param integer $code 状态码
	*@param string $message 提示信息
	*@param array $data 数据
	*return string
	*/
	public static function json($code, $message='',$data = array()){

		if (!is_numeric($code)) {
			return '';
		}

		$result = array(
			'code' => $code, 
			'message' => $message,
			'data' => $data
		);

		echo json_encode($result);
	}
}

```

## 生成xml数据

* 1、组装字符串

* 2、使用系统类

```
 * DomDocument
 * XMLWriter
 * SimpleXML
```

### xml例子

```
	public static function xml() {

		header("Content-Type:text/xml");

		$xml = "<?xml version='1.0' encoding='UTF-8'?>";
		$xml .= "<root>\n";
		$xml .= "<code>200</code>\n";
		$xml .= "<message> 数据返回成功 </message>\n";
		$xml .= "<data>\n";
		$xml .= "<id>1</id>\n";
		$xml .= "</data>\n";
		$xml .= "</root>\n";

		echo $xml;
	}

```

### xml 方式封装接口数据方法

* 封装方法

```
xmlEncode($code,$message = "", $data = array())
```

* data数据分析

1、 array('index' = > 'api');
2、 array(1,7,89)


**`封装json、xml的例子`**

```
<?php

class Response {
	const JSON = "json";
	/**
	* 按综合方式输出通信数据
	* @param integer $code 状态码
	* @param string $message 提示信息
	* @param array $data 数据
	* @param string $type 数据类型
	* return string
	*/
	public static function show($code, $message = '', $data = array(), $type = self::JSON) {
		if(!is_numeric($code)) {
			return '';
		}

		$type = isset($_GET['format']) ? $_GET['format'] : self::JSON;

		$result = array(
			'code' => $code,
			'message' => $message,
			'data' => $data,
		);

		if($type == 'json') {
			self::json($code, $message, $data);
			exit;
		} elseif($type == 'array') {
			var_dump($result);
		} elseif($type == 'xml') {
			self::xmlEncode($code, $message, $data);
			exit;
		} else {
			// TODO
		}
	}
	/**
	* 按json方式输出通信数据
	* @param integer $code 状态码
	* @param string $message 提示信息
	* @param array $data 数据
	* return string
	*/
	public static function json($code, $message = '', $data = array()) {
		
		if(!is_numeric($code)) {
			return '';
		}

		$result = array(
			'code' => $code,
			'message' => $message,
			'data' => $data
		);

		echo json_encode($result);
		exit;
	}

	/**
	* 按xml方式输出通信数据
	* @param integer $code 状态码
	* @param string $message 提示信息
	* @param array $data 数据
	* return string
	*/
	public static function xmlEncode($code, $message, $data = array()) {
		if(!is_numeric($code)) {
			return '';
		}

		$result = array(
			'code' => $code,
			'message' => $message,
			'data' => $data,
		);

		header("Content-Type:text/xml");
		$xml = "<?xml version='1.0' encoding='UTF-8'?>\n";
		$xml .= "<root>\n";

		$xml .= self::xmlToEncode($result);

		$xml .= "</root>";
		echo $xml;
	}

	public static function xmlToEncode($data) {

		$xml = "";
		foreach($data as $key => $value) {
			$attr = "";
			if(is_numeric($key)) {
				$attr = " id='{$key}'";
				$key = "item";
			}
			$xml .= "<{$key}{$attr}>";
			$xml .= is_array($value) ? self::xmlToEncode($value) : $value;
			$xml .= "</{$key}>\n";
		}
		return $xml;
	}

}
```



# 核心技术

## 缓存技术

### 静态缓存
保存在磁盘上的静态文件，用php生成数据数据放入静态文件中

**`操作文件例子：`**

```
<?php

class File{
	private $_dir;

	const EXT = '.txt';

	public function __construct(){
		//获取当前文件所在文件夹
		$_dir = dirname(__FILE__).'/files/';
	}

	/*
	*key 文件名
	*value 文件内容
	*path 文件路径
	*/
	public function cacheData($key,$value='',$path=''){
		//fwrite写入文件 fopen 如果文件不在就创建
		//或者 file_put_contents(filename, data),data一定要是string类型
		$filename = $this->_dir.$path.$key.self::EXT;

		if($value !== ''){

			if (is_null($value)) {
				// 删除文件函数
				return @unlink($filename);
			}

			//获取路径名称
			$dir = dirname($filename);
			//是否存目录
			if (!is_dir($dir)) {
				//不存在的话，就创建文件夹
				mkdir($dir);
			}

		//如果写入错误，返回false，否则返回success。。//将value值写入缓存
		return file_put_contents($filename,json_encode($value));
		}

		//判断文件是否存在
		if (!is_file($filename)) {
			return FALSE;
		}else {
			// 获取文件内容，true参数表示返回数组，不传这个值会返回对象
			return json_decode(file_get_contents($filename),true);
		}

	}

}
```


#### php操作缓存

* 生成缓存

```
file_put_contents($filename,json_encode($value));
```

* 获取缓存

```
json_decode(file_get_contents($filename),true);
```

* 删除缓存

```
 @unlink($filename);
```

### mysql

```
登录命令：用户名，回车后输入密码
mysql -u root -p 

```

### Memcache , Redis

```
1、Memcache和Redis都是用来管理数据的
2、他们数据都是存放在内存里的
3、Redis可以定期江数据备份到磁盘（持久化）
4、Memcache只是简单的key/value缓存
5、Redis不仅仅支持简单的k/v类型的数据，同时还提供list、set、hash等数据结构的存储。
```

#### Redis数据操作

```
1、开启redis服务器
➜  redis-3.2.0 git:(master) redis-server redis.conf
➜  redis-3.2.0 git:(master) cd src
➜  src git:(master) ./redis-cli


2、设置缓存值 set key_name 'value'


3、获取redis缓存 get key_name


4、设置过期时间 setex key 10 'cache' (注意单位秒)


5、删除缓存 del key
```

`php操作Redis`

```
1、安装phpredis扩展
2、php链接redis服务器  connect(127.0.0.1)
```

## 定时任务

* 1、定时任务服务提供crontab命令来设定服务,  `crontab -e`编辑某个用户的cron服务

```
sudo crontab -e
```
进入编辑界面

* 2、`crontab -l` 查看设定的定时任务

* 3、`crontab -r` 删除某个用户的所有cron服务

### crontab 定时任务格式

```
分		小时		日		月		星期		命令	
*		*		*		*		*	
0-59 	0-23	1-31	1-12	0-6		command

注： "*" 代表取值范围内的数字
	"/"	代表每、比如每分钟等

例子：

*/1 * * * * php /data/www/cron.php  意思是每分钟执行 cron.php  （*是每个可能的取值，*/1 保证每分钟，因为*/1的结果不可能是数字）

50 7 * * *  /sbin/service sshd start 意思是每天7：50 开启ssh服务

```

##### 定时执行php程序

* 1、编写好php程序

```
$connect = mysql_connect('localhost:3306','***','***');
	mysql_select_db('php100',$connect);

	$sql = "insert into news(title) values('title1')";
	mysql_query($sql,$connect);
```

* 2、配置 crontab定时任务

```
sudo crontab -e

```



# 接口实例

### 学习要点
1、掌握单例设计模式

2、php是怎么连接数据库的

## 单例模式连接数据库

* 1、单例模式三大原则

```
-单例模式三大原则
1、构造函数需要标记为非public（防止外部使用new操作符创建对象),单例类不能在其他类中实例化，只能被其自身实例化；
2、拥有一个保存类的实例的静态成员变量 $_instance;
3、拥有一个访问这个实例的公共的静态方法。
```

**`例子代码`**

```
<?php
	
	//php 单例模式
	/**
	* 
	*/
	class DB
	{
		static private $_instance;
		static private $_connectSource;
		private $_dbConfig = array(
				'host' => 'localhost:3306',
				'user' => 'root',
				'password' => '***',
				'database' => '****',
			);

		private	function __construct()
		{
			
		}

		static public function getInstance(){
			if (!(self::$_instance instanceof self)) {
				self::$_instance = new self();
			}

			return self::$_instance;
		}


		public function connect(){

			if (self::$_connectSource) {
				return self::$_connectSource;
			}

			self::$_connectSource = mysql_connect($this->_dbConfig['host'],$this->_dbConfig['user'],$this->_dbConfig['password']);

			if (!self::$_connectSource) {
				die('mysql connect error'.mysql.error());
			}

			mysql_select_db($this->_dbConfig['database'],self::$_connectSource);

			mysql_query("set names UTF8",self::$_connectSource);

			return self::$_connectSource;
		}

	}

	$connect = DB::getInstance()->connect();

	var_dump($connect);

	$sql = "select * from news";

	$result = mysql_query($sql,$connect);
	echo mysql_num_rows($result);

```




## 开发首页 接口 以及客户端使用实例

### 方案一
读取数据库方式开发首页接口

```
从数据库获取信息 -> 封装 -> 生成接口数据 

应用场景：
数据实效性比较高的系统

```

**`例如：`**

```
<?php
	
	header("Content-Type: text/html;charset=utf-8");

	//http://app.com/list.php?page=1&pagesize=12
	require_once('./response.php');
	require_once('./DBOnceInstance.php');

	$page = isset($_GET['page']) ? $_GET['page'] : 1;
	$pagesize = isset($_GET['pagesize']) ? $_GET['pagesize'] : 1;
	if (!is_numeric($page) || !is_numeric($pagesize)) {
		return Response::show(401,'数据不合法');
	}

	$offset = ($page-1) * $pagesize;
	// limit 0,6  起始值是0，取6个数据
	$sql = "select * from video where status = 1 order by orderby desc limit ".$offset.",".$pagesize;

	try {
		$connect = DB::getInstance()->connect();
	} catch (Exception $e) {
		//$e->getMessage();
		return Response::show(403,'数据库连接失败');
	}
	
	$result = mysql_query($sql,$connect);
	
	while ($video = mysql_fetch_assoc($result)) {
		$videos[] = $video;
	}

	if ($videos) {
		return Response::show(200,'首页数据获取成功',$videos);
	}else {
		return Response::show(400,'fail');
	}

```



### 方案二
读取缓存方式开发首页接口

```
从数据库获取信息 -> 

再次请求 -> 缓存 -> 封装 -> 返回数据

用途：减少数据库压力

例如：搜狐、youku

```




### 方案三
定时读取缓存方式开发首页接口

```
数据库 -> crontab -> 缓存

http请求 -> 缓存   -> 封装返回数据

```



# 遇到的问题

* 用phpMyAdmin创建表的时候出现 `1089 incocrect prefix key` 的错误

```
解决：
定义主键的时候，选择AUTO_INCREMENT ， 不要填写 类型的长度。
```












