---
title: "PHP: Misknown OOP functions"
date: 2012-07-25T11:49:00
summary: "Some of unknown useful PHP functions"
tags:
  - php
---
While reading PHP documentation, especially the [OOP chapter](http://www.php.net/manual/en/language.oop5.php), I've discovered a few functions I didn't know of, even if I use a lot PHP in my daily job.
Please note that PHP5 was introduced in 2004, and these methods may evolve.


__autoload
----------

When calling a class (or an interface) still not declared, PHP will call the [__autoload](http://php.net/manual/en/language.oop5.autoload.php) function, if it was declared sooner.
It can be usefull to reduce the number of require/include calls and parsing PHP files only on demand.

Simple example:

```php
<?php // Objet.class.php
class Objet
{
};
```

```php
<?php // test.php
function __autoload($class_name)
{
    echo "Autoloading class $class_name\n";
    $class_file = $class_name . '.class.php';
    if(FALSE === file_exists($class_file))
    {
        throw new Exception('Class "' . $class_name . '" could not be autoloaded (file missing)');
    }
    require_once $class_name . '.class.php';
    if(FALSE === class_exists($class_name))
    {
        throw new Exception('Class "' . $class_name . '" could not be autoloaded (not in file)');
    }
    echo "Autoloading done!";
    return true;
}

// Using "Object" class defined in Object.class.php
$obj = new Objet;

// Trying to use "ObjetNonDefini" will go in error:
$obj2 = new ObjetNonDefini;
```

```sh
$ php test.php
Autoloading class Objet
Autoloading done!
Autoloading class ObjetNonDefini
PHP Fatal error:  Uncaught exception 'Exception' with message 'Class "ObjetNonDefini" could not be autoloaded (file missing)' in /home/mycroft/tmp/php/test.php:9
Stack trace:
#0 /home/mycroft/tmp/php/test.php(26): __autoload('ObjetNonDefini')
#1 {main}
  thrown in /home/mycroft/tmp/php/test.php on line 9
```


__construct / __destruct
------------------------

Like other OOP languages, PHP provides generic [constructors/destructors](http://www.php.net/manual/en/language.oop5.decon.php) that will be called on object instantiation and destruction.
Note that destructor will be called even if object is not explicitaly destroyed (example: at the end of a script).

```php
<?php // Objet.class.php
class Objet
{
    public function __construct()
    {
        echo "Constructor\n";
    }

    public function __destruct()
    {
        echo "Destructor\n";
    }
};
```

```php
<?php // test.php
require_once("Objet.class.php");
$obj = new Objet;
echo "Fin du script.\n";
```

```sh
$ php test.php
Constructor
Fin du script.
Destructor
```


Visibility of private methods
-----------------------------

PHP also manages private methods and variables. They are only accessible in their own instanciated object. But PHP allows to call a private method from an object instance from another object, as long as this object is from the same type. This is fully described in the [OOP Visibility](http://www.php.net/manual/en/language.oop5.visibility.php).

Example:

```php
<?php // Objet.class.php
class Objet
{
    private $obj_name = NULL;

    public function __construct($name = NULL)
    {
        $this->obj_name = $name;
    }

    // Private method
    private function privateMethod()
    {
        echo "Methode privée dans " . $this->obj_name . "\n";
    }

    // This method will allow foreign objects to call our private method.
    public function test(Objet $t)
    {
        $t->privateMethod();
    }
};
```

```php
<?php // fichier test.php
require_once("Objet.class.php");
$obj = new Objet('objet n°1');
$obj2 = new Objet('objet n°2');

// The "obj" object will call private method in obj2:
$obj->test($obj2);
```

And this works:

```sh
$ php test.php 
Methode privée dans objet n°2
```


__toString
----------

The [__toString function](http://www.php.net/manual/en/language.oop5.magic.php#language.oop5.magic.tostring) is a simple function allowing to define the class interacts when a String conversion of it is required (by using "print").

```php
    // Using the last object:
    public function __toString()
    {
        return "class " . $this->obj_name;
    }
```

```php
// In test.php
echo $obj;
// Will show 'class objet n°1'
```


__invoke
--------

In the same fashion, it is possible to define how the object will interact when being called as a function. This is done with the [__invoke function](http://www.php.net/manual/en/language.oop5.magic.php#language.oop5.magic.invoke)

```php
    // Using the last object:
    public function __invoke()
    {
        echo "Appel de echo sur " . $this . "\n";
    }
```

```php
<?php // In test.php:
require_once("Objet.class.php");
$obj = new Objet('objet n°1');
$obj();
```

And when executing:

```sh
$ php test.php
Appel de echo sur class objet n°1
```

__get, __set, __isset, __unset
------------------------------

PHP allows affectation on any class instance any kind of values (like a hash table), as long as this is not a private variable:

```php
<?php

class Objet {};
$obj = new Objet;
$obj->exemple = 1;
echo $obj->exemple . "\n"; // prints '1'
```

When affecting a variable this way, PHP will call the [__set/__get functions](http://www.php.net/manual/en/language.oop5.overloading.php#language.oop5.overloading.members). And because of this, it is possible, with reimplementing those functions, to define a new behavior:

```php
<?php // Objet.class.php

class Objet
{
    private $data;
    public function __construct()
    {
        $data = array();
    }

    public function __set($name, $value)
    {
        echo "Setting $name to $value\n";
        $this->data[ $name ] = $value;
    }

    public function __get($name)
    {
        echo "returns $name\n";
        return $this->data[ $name ];
    }
};
```

```php
<?php // In test.php

require_once("Objet.class.php");
$obj = new Objet;
$obj->priv = 1;
echo $obj->priv;
```

This example will return:

```sh
$ php test.php
Setting priv to 1
return priv
1
```

This also allows to protect the class contents. Note that if __set/__get is redefined, there won't be anymore errors when trying to access private variables.


__call et __callStatic
----------------------

As for variables, it is possible to catch undefined function (static or not) calls in classes, with using [_call / __callStatic functions](http://www.php.net/manual/en/language.oop5.overloading.php#language.oop5.overloading.methods).

```php
<?php // Objet.class.php
class Objet
{
    public function __call($func, $args)
    {
        echo "Calling $func\n";
    }

    public static function __callStatic($func, $args)
    {
        echo "Calling static $func\n";
    }
};
```

```php
<?php // test.php

class Objet {};
$obj = new Objet;
$obj->exemple = 1; echo $obj->exemple . "\n";
```

At execution:

```sh
$ php test.php
Calling hello
Calling static hello
```

Those two functions, with the help of variable setters/getters, allow to entirely wrapper classes' instances, allowing uses more interesting that simple inheritance.

__clone
-------

[__clone](http://www.php.net/manual/en/language.oop5.cloning.php) is called when asking for a copy of an object instance. This will result in 2 totally independant instances with the same content. While I'm not sure that clone is commonly used, *__clone* may be used to modify an object after cloning, or to not effectively clone the object ot use a singleton.

__sleep, __wakeup
-----------------

Finally, [__sleep/__wakeup](http://www.php.net/manual/en/language.oop5.magic.php#language.oop5.magic.sleep) are used when (un)serializing objects.

```php
<?php // Objet.class.php

class Objet
{
    private $file_name = NULL;
    private $file_hd = NULL;

    public function __construct($file_name)
    {
        $this->file_name = $file_name;
        $this->file_open();
    }

    public function __destruct()
    {
        if($this->file_hd !== NULL)
            $this->file_close();
    }

    private function file_open()
    {
        echo "Opening file\n";
        $this->file_hd = fopen($this->file_name, "a+");
    }

    private function file_close()
    {
        if(NULL === $this->file_hd)
            return false;

        echo "Closing file\n";
        fclose($this->file_hd);
        $this->file_hd = NULL;
    }

    public function __sleep()
    {
        echo "* Serialization.\n";
        $this->file_close();

        return(array('file_name'));
    }

    public function __wakeup()
    {
        echo "* Unserialization.\n";
        $this->file_open();
    }
};
```

Et testons tout ça:

```php
<?php // test.php

require_once("Objet.class.php");

$obj = new Objet('helloworld.txt');

$serializedObj = serialize($obj);
unset($obj);
var_dump($serializedObj);

$newObj = unserialize($serializedObj);

echo "Fin du script\n";
```

We'll check the behavior:

```sh
$ php test.php 
Opening file
* Serialization.
Closing file
string(62) "O:5:"Objet":1:{s:16:"Objetfile_name";s:14:"helloworld.txt";}"
* Unserialization.
Opening file
Fin du script
Closing file
```

Also, see [9 magic methods for PHP](http://www.lornajane.net/posts/2012/9-magic-methods-in-php).

