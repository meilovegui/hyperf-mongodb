### 默认使用mongodb提供的库来封装,官方git地址：https://github.com/mongodb/mongo-php-library
#### 1、支持类似mysql orm的一些操作
#### 2、支持迁移文件
#### 3、只支持hyperf框架，由于swoole协程不支持mongodb，所以所有的方法都采用task进程来实现，该包已经封装好所有的方法都会投递到task进程进行操作，task进程建议开启多一点
#### 4、该包默认使用了连接池

### 使用
#### 1、拉取包
```
composer require zrone/hyperf-mongodb
```
#### 2、发布配置
```
php bin/hyperf.php mongodb:publish --config
```
#### 3、配置介绍
```
<?php
declare(strict_types=1);
return [
    'default' => [
        'uri_options' => [
            'ssl' => true,
            'username' => env('MONGODB_USERNAME', ''),
            'password' => env('MONGODB_PASSWORD', ''),
//            'authMechanism' => env('MONGODB_AUTH_MECHANISM', 'SCRAM-SHA-256'),
            //设置复制集,没有不设置
//        'replicaSet' => 'rs0',
        ],
        'host' => env('MONGODB_HOST', '127.0.0.1'),
        'port' => env('MONGODB_PORT', 27017),
        'db' => env('MONGODB_DB', 'test'),
        'driver_options' => [],
        'migration' => [
            'path' => BASE_PATH . '/migrations/mongodb', // 迁移文件的路径
        ],
        'pool' => [
            'min_connections' => 1,
            'max_connections' => 100,
            'connect_timeout' => 10.0,
            'wait_timeout' => 3.0,
            'heartbeat' => -1,
            'max_idle_time' => (float)env('MONGODB_MAX_IDLE_TIME', 60),
        ],
    ],
];
```
#### 4、生成迁移文件
```
php bin/hyperf.php mongodb:migration Test
```
上面的命令会自动生成一个迁移文件，会生成一个文件到配置文件指定的迁移目录中
#### 5、迁移文件例子
```
<?php
declare(strict_types=1);
namespace Zrone\MongoDb\Example\Migrations;
use Zrone\MongoDb\MongoDbMigration;
class CreateTestCollection extends MongoDbMigration
{
    /**
     * 支持很多方法，请详细去看MongoDbMigration这个类
     * @throws \Zrone\MongoDb\Exception\MongoDBException
     */
    public function up()
    {
        $msg = [];
        $msg[] = $this->createCollection('test'); // 创建一个表
        $data = [
            ['dd' => 1, 'tt' => 2],
            ['dd' => 2, 'tt' => 4],
        ];
        $msg[] = $this->insertMany('test', $data); // 插入多条数据
        $msg[] = $this->createIndex('test', ['dd' => 1, 'tt' => 1]); // 在该表上创建索引
        $msg[] = $this->createIndexes('test', [['dd' => 1], ['tt' => 1]]); // 在该表上批量创建索引
        $msg[] = $this->dropCollection('test'); // 删除一个表
        return $msg;
    }

    /**
     * 迁移失败时会执行
     * @throws \Zrone\MongoDb\Exception\MongoDBException
     */
    public function down()
    {
        return 'error';
    }
}
```
#### 6、迁移命令
```
php bin/hyperf.php mongodb:migrate 
```
上面这个命令会迁移你所有生成的文件，迁移文件路径在配置文件里面配置
#### 7、开发使用
1、上面能像orm一样能进行迁移了，解决了升级的问题，下面我们来说一下开发时候怎么使用   
2、在你的项目里面新建一个目录，该目录叫mongo(自行命名，类似orm的model)  
3、比如我现在项目里面有一个库，叫test，test里面有两个collection，名字为co1,co2(你把它当成mysql的表一样)   
4、我在mongo目录新建两个文件，叫Co1Mongo和Co2Mongo，都继承\Zrone\MongoDb\MongoDb   
```
<?php
declare(strict_types=1);
namespace TmgAddons\WebQySession\Admin\Mongo;

use Zrone\MongoDb\MongoDb;
class TestMongo extends MongoDb
{
    /**
     * mongodb表
     * @var null
     */
    public $collectionName = 'co1';
}
```
5、查询co1中的一条数据
```
<?php
declare(strict_types=1);
namespace TmgAddons\WebQySession\Admin\Controller;

use Hyperf\Di\Annotation\Inject;
use Hyperf\HttpServer\Annotation\Controller;
use Hyperf\HttpServer\Annotation\GetMapping;
use TmgAddons\WebQySession\Admin\Mongo\Co1Mongo;

/**
 * @Controller(prefix="/test")
 * Class TestController
 * @package TmgAddons\WebQySession\Admin\Controller
 */
class TestController
{
    /**
     * @Inject()
     * @var Co1Mongo
     */
    private $co1Mongo;

    /**
     * @GetMapping(path="")
     * @return array
     * @throws \Zrone\MongoDb\Exception\MongoDBException
     */
    public function test()
    {
        return response_success($this->co1Mongo->findOne());
    }
}

调用test方法时，就能查出co1表中的一条数据了，是不是很简单！
```
6、支持有多种方法，详细你可以到Zrone\MongoDb\MongoDb查看，或者你可以去看官方的php-mongodb文档，https://docs.mongodb.com/php-library/v1.5/reference/method/MongoDBCollection-createIndexes/
#### 8、结束
如果你有使用的问题或者建议，欢迎你提一个isset，由于太匆忙，等我开发完现在的项目，我会重新优化和迭代这个包，如果开发中有遇到问题或者有更好的写法，我会迭代到这个包这里。
#### 9、注意
如果你使用的是mongodb默认生成_id,那么更新和删除我默认已经帮你使用MongoDB\BSON\ObjectId进行了转换，所以你无需再转换，你直接把_id对应的字符串传进去就好了，比如要根据_id获取某条数据，$filter=['_id' => 'xxxxxx']即可。获取数据时，我也默认帮你把_id转成了字符串

#### 10、遇到问题
1、因为我使用了hyperf框架，所以基本都是单例模式。我model里面使用\Zrone\MongoDb\MongoDb 的setCollectionName方法来切换表时，必须处理完逻辑后然后再切回原始的collectionName，否则你查询的会是你切换的表
