---
title: "xhprof, profiling PHP: Installation & configuration"
date: 2010-05-10
summary: "Profile PHP code like nobody else with xhprof"
tags:
  - php
---

[xhprof](http://pecl.php.net/package/xhprof) is a opensource PHP extension developed by Facebook which allows to profile PHP code. It can be a very powerful tool if you're able to understand what it outputs, and allows you to optimize your PHP application.

Installation
------------

Like any other php module, install goes through phpize:

```sh
$ wget http://pecl.php.net/get/xhprof-0.9.2.tgz
$ tar xvfz xhprof-0.9.2.tgz
$ cd xhprof-0.9.2/
$ cd extension
$ phpize
Configuring for:
PHP Api Version:         20041225
Zend Module Api No:      20060613
Zend Extension Api No:   220060519
$ ./configure
[...]
$ make
[...]
$ make install
[...]
```

You'll have to add a new section in *php.ini*:

```ini
[xhprof]
extension=xhprof.so
xhprof.output_dir=/tmp/xhprof
```

And you'll create the output directory:

```sh
$ mkdir /tmp/xhprof ; chmod a+rwt /tmp/xhprof
```

And finally, you'll just have to install xhprof vizualisation code, and create a virtualhost for it (if you want to):

```sh
$ cp -Rp xhprof_html xhprof_lib /www/xhprof/
```

I've created a small script to retrieve quickly all available reports:

```php
<?php
echo "<b>Available reports</b><br /><br />";

$dir = opendir('/tmp/xhprof');
if( ! $dir )
    die('No directory.');

while(FALSE != ($file = readdir($dir)))
{
    if($file[0] === '.')
        continue;
    $rep = explode('.', $file);
    echo "<a href='/xhprof_html/?run=" . $rep[0] . "&source=" . $rep[1] ."'>";
    echo $file;
    echo "</a><br />";
}

closedir($dir);
```

Code profiling
--------------

To generate a report, you'll have to start analysis engine and stop it at the end. In the following snippet, you'll analyse the *foo()* function code:

```php
<?php

// Initializing xhprof
xhprof_enable( XHPROF_FLAGS_CPU + XHPROF_FLAGS_MEMORY );

foo();

// Stopping xhprof & writing report
$xhprof_data = xhprof_disable();

include_once "/www/xhprof/xhprof_lib/utils/xhprof_lib.php";
include_once "/www/xhprof/xhprof_lib/utils/xhprof_runs.php";

$xhprof_runs = new XHProfRuns_Default();
$run_id = $xhprof_runs->save_run($xhprof_data, "foo");
```

Reports will be written in */tmp/xhprof* and can be viewable directly with xhprof GUI. For more information, check the [xhprof manual](http://php.net/manual/en/book.xhprof.php).
