---
layout: post
title:  PHP DateTime Differences (Signed!)
date:   2013-04-18 18:55:15
tags:   [php,datetime]
excerpt: 'Note to self: remember to add "%R" (or "%r") when formatting a date difference in PHP.'
redirect_from:
  - /blog/2013/04/php-datetime-differences-signed/
  - /2013/04/18/php-datetime-differences-signed/
---
Note to self: remember to add "%R" (or "%r") when formatting a date difference in PHP. According to the <a href="http://us2.php.net/manual/en/datetime.diff.php" target="_blank">PHP manual</a>, the optional second parameter to `DateTime::Diff()` determines whether the difference between two dates is given as an absolute, i.e., forced to positive. That does NOT mean that "%a" returns a signed integer by default. Consider the following example:

```php
<?php
$date1 = new DateTime('2013-04-10');
$date2 = new DateTime('2013-04-12');
$diff = $date1->diff($date2);
echo $diff->format('%a')."\n";
```

The "%a" format returns the total number of days, in this case 2. Since the second parameter is not set and thus defaults to false, you might think that this means `$date2` is two days greater than `$date1`. Wrong &ndash; it could just as easily mean the opposite. Change `$date2` to "2013-04-08" and you get the same result! Regardless of the optional `$absolute` parameter, "%a" (and its equivalent, `$diff->days`) never includes a sign. `$diff->days` and `$diff->format('%a')` will always return the absolute number of days between the two dates. If you care about which came first, use `$diff->format('%r%a')` instead.