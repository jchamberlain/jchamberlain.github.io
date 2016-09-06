---
layout: post
title:  PHP Property Visibilty
date:   2013-04-05 15:58:00
tags: [php,oop,encapsulation]
excerpt: Of course I knew that private properties and methods are visible only to the class that defined them, but one particular implication of that fact had not yet crossed my mind.
redirect_from:
  - /blog/2013/04/php-property-visibility/
  - /2013/04/05/php-property-visibility/
---
It seems like parents just can't keep track of their children these days! A while back I wrote a little code that seemed fine to me but returned a surprising fatal error. Of course I knew that private properties and methods are visible only to the class that defined them, but one particular implication of that fact had not yet crossed my mind. Consider the following bits of code:

```php
<?php // ParentClass.php
class ParentClass {
  private $privateVar = 'private';
  protected $protectedVar = 'protected';
  public $publicVar = 'public';

  public function testMethod() {
    echo '<pre>';
    echo $this->privateVar."\n";
    echo $this->protectedVar."\n";
    echo $this->publicVar."\n";
    echo '</pre>';
  }
}

```

```php
<?php // index.php
include 'ParentClass.php';

$test = new ParentClass();
$test->testMethod();
```

When index.php is loaded, we'll see all three properties displayed just as expected.
<pre>private
protected
public</pre>

Let's extend the parent class:

```php
<?php // ChildClass.php
class ChildClass extends ParentClass {
  private $privateVar = 'private child';
  protected $protectedVar = 'protected child';
  public $publicVar = 'public child';
}
```

```php
<?php // index.php
include 'ParentClass.php';
include 'ChildClass.php';

$test = new ChildClass();
$test->testMethod();
```

Having overridden all three properties of the parent class, loading index.php yields a different but expected (for me) result.
<pre>private
protected child
public child</pre>

The original testMethod() is defined in ParentClass and thus has access only to the original $privateVar, not the new version in ChildClass. The protected and public properties are overridden and thus show their new values.

Here's where I ran into trouble. Consider the following method, simplified from the controller class of an old framework I built a while back:

```php
<?php
// ...
protected function loadView($name, $data=array()) {
  $file = APATH.'views/'.$name.'.php';
  if(file_exists($file)) {
    extract($data);
    ob_start();
    include($file);
    $html = ob_get_contents()."\n";
    ob_end_clean();
    $this->html .= $html;
    return true;
  }
  return false;
}
```

As you can see, it's including a view file, giving it access to certain data, and then saving the output. Since the file is included within a class method, whatever's in the file has access to the class properties and methods just like any other part of the class, so sometimes for convenience I would use $this within the view file (probably a bad idea), such as when grabbing `$this->user`. Once I tried accessing a private property from the view and was surprised by the accompanying fatal error. It went something like this:

view:

```php
<?=$this->pageID?>
```

controller:

```php
<?php

class Page extends Controller {
  private $pageID = 1;

  public function index() {
    // ...
    $this->loadView('view.php', array());
  }
}
```

The `loadView()` method was defined in the `Controller` class, but run in the `Page` class where the private `$pageID` was declared. There was no `$pageID` in the `Controller` class, but instead of just echoing `NULL`, PHP recognized my code as an illegal attempt to access a private property in `Page` from a method in `Controller` and returned a fatal error. And that's what really confused me. Instead of acting like the property didn't exist in `Controller` (it didn't), the interpreter correctly ascertained what I wanted to do and refused to do it.

So apparently the PHP interpreter is smarter than I am.