#给 CI 插上翅膀——在 CodeIgniter 2 中使用 Laravel Eloquent ORM

## 说明

> 原文发表在我的个人网站：[给 CI 插上翅膀——在 CodeIgniter 2 中使用 Laravel Eloquent ORM](http://lvwenhan.com/php/414.html)

##背景介绍

CodeIgniter 框架和 Laravel 框架基本可以看做是之前若干年和这两年的 PHP 框架霸主，使用率和出镜率最高的框架。

CI 是一个轻型框架，只提供了 路由、MVC 分离、视图加载器、Active Record 等一些基本功能，但这恰恰是其使用率高的原因：提供的东西少而精，适用于绝大多数场景。CI 的文档堪称开源软件的典范，非常之清晰、详尽，对新手非常友好，十分容易上手。

Laravel 是这两年刚刚兴起的重型全功能框架，可以极大地提高开发效率，但是 Laravel 4 的文档并没有 3 那么清晰，中文资料也非常少，很多人在学习的时候遇到了比较大的困难。例如复杂的路由系统让很多用惯了 CI 自动映射的人无所适从，文档又只有寥寥几句话，导致相当一部分人被拦在了使用框架的第一步，学习的热情也被浇灭。

我前段时间写了系列教程 [Laravel 4 系列入门教程【最适合中国人的Laravel教程】](http://lvwenhan.com/laravel/398.html)，访问量和反响都还不错，需要的人可以看看。

繁琐的路由让很多人怀念 CI 的自动映射，繁重的框架基础工作（一个 Hello World 页面需要载入 150 多个文件）也让 Laravel 的性能在一些场景下不能满足要求。很多人用了一段时间后发现，Laravel 中的 Eloquent ORM 是跟 CI 比最强大的地方，于是就想把 Eloquent 移植到 CI 上，我以前也想过，无奈实力不够无从下手。现在终于知道怎么搞了，下面我们正式开始。

##基础准备

PHP 版本要求 >= 5.4，这是 Eloquent 的最低要求。

下载 CodeIgniter 2.2.0，地址是 [http://www.codeigniter.com/download](http://www.codeigniter.com/download)，下载完成后解压到某个地方，配置好 HTTP 服务软件，把网站跑起来。如果你已经看到了以下画面，就可以继续往下做了：
![pic](http://lvwenhan.com/content/uploadfile/201410/6aab1413966279.jpg)

##开始嫁接

我们使用 Composer 来载入和管理 Eloquent。Composer 会生成一个自动加载（`autoload`）文件，我们只需要 `require` 这个文件，就可以使用所有通过 Composer 安装的包。现在我们要在 CodeIgniter 项目中使用 Composer，在其根目录下新建 composer.json：

```json
{
  "require": {
    "php": ">=5.4.0",
    "illuminate/database": "*"
  },
}
```

然后运行 `composer update`，稍等片刻，Composer 体系创建完成，同时 illuminate/database 包也已经安装完成。

然后新建 `application/third_party/eloquent.php`：

```php
<?php

defined('BASEPATH') OR exit('No direct script access allowed');

use Illuminate\Database\Capsule\Manager as Capsule;

// Autoload 自动载入
require BASEPATH.'../vendor/autoload.php';

// 载入数据库配置文件
require_once APPPATH.'config/database.php';

// Eloquent ORM
$capsule = new Capsule;

$capsule->addConnection($db['eloquent']);

$capsule->bootEloquent();
```

这个文件将会帮我们引入 Composer 的自动加载文件，同时会帮我们初始化 Eloquent，这个文件载入了一个数据库配置文件，在 `application/config/database.php` 的最后新增（注意替换数据库名称和密码）：

```php
$db['eloquent'] = [
  'driver'    => 'mysql',
  'host'      => 'localhost',
  'database'  => 'ci',
  'username'  => 'root',
  'password'  => 'password',
  'charset'   => 'utf8',
  'collation' => 'utf8_general_ci',
  'prefix'    => ''
  ];
```

接下来我么需要在 CI 应用启动的时候引入上面那个文件，在最外面的 `index.php` 的后部增加：

```php
/*
 * --------------------------------------------------------------------
 * LOAD Laravel Eloquent ORM
 * --------------------------------------------------------------------
 *
 */

require APPPATH.'third_party/eloquent.php';
```

> 注意，这段代码一定要放在 `require_once BASEPATH.'core/CodeIgniter.php';` 这一行的 ***前面***！

然后，开始使用 Eloquent，修改 `application/controllers/welcome.php` 中的 `index()` 为：

```php
public function index()
{
  $data['article'] = Article::first();
  $this->load->view('home', $data);
}
```

新建 `application/views/home.php` 文件：
```php
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>CI2 with Eloquent</title>
</head>
<body>

  <h1>
    <?php echo $article->title; ?>
  </h1>
  <div class="content">
    <p>
      <?php echo $article->content; ?>
    </p>
  </div>

</body>
</html>
```

现在让我们向数据库中填充需要使用的数据，运行 SQL 语句：
```sql
DROP TABLE IF EXISTS `articles`;

CREATE TABLE `articles` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `title` varchar(255) DEFAULT NULL,
  `content` longtext,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

LOCK TABLES `articles` WRITE;
/*!40000 ALTER TABLE `articles` DISABLE KEYS */;

INSERT INTO `articles` (`id`, `title`, `content`)
VALUES
  (1,'我是标题','<h3>我是内容呀~~</h3><p>我真的是内容，不信算了，哼~ O(∩_∩)O</p>'),
  (2,'我是标题','<h3>我是内容呀~~</h3><p>我真的是内容，不信算了，哼~ O(∩_∩)O</p>');

/*!40000 ALTER TABLE `articles` ENABLE KEYS */;
UNLOCK TABLES;
```

然后建立模型，新建 `application/models/Article.php` 文件：

```php
<?php

defined('BASEPATH') OR exit('No direct script access allowed');

/**
* Article Model
*/
class Article extends Illuminate\Database\Eloquent\Model
{
  public $timestamps = false;
}
```

最后，修改 `composer.json` 将 models 文件夹加入自动加载：

```json
{
  "require": {
    "php": ">=5.4.0",
    "illuminate/database": "*"
  },
  "autoload": {
    "classmap": [
      "application/models"
    ]
  }
}
```

运行 `composer dump-autoload`，刷新页面！你将看到以下画面：

![pic](http://lvwenhan.com/content/uploadfile/201410/76b01413970486.jpg)

**恭喜你！Eloquent 嫁接到 CodeIgniter 2 成功！**

CodeIgniter 3 正处于 DEV 阶段，原生支持 Composer，可以直接修改 composer.json 文件，配置方式和 2 完全一致。

##资源汇总

1. CodeIgniter 2 with Eloquent，GitHub 地址：[https://github.com/johnlui/CodeIgniter-2-with-Eloquent](https://github.com/johnlui/CodeIgniter-2-with-Eloquent)
2. [利用 Composer 一步一步构建自己的 PHP 框架（四）——使用 ORM](http://lvwenhan.com/php/409.html)
3. [Eloquent 中文文档](http://laravel-china.org/docs/eloquent)
4. [Download CodeIgniter](http://www.codeigniter.com/download)
5. [illuminate/database](https://github.com/illuminate/database)

### License

CodeIgniter-2-with-Eloquent is open-sourced software licensed under the [MIT license](http://opensource.org/licenses/MIT)
