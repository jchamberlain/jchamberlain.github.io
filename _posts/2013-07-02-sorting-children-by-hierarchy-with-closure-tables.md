---
layout: post
title:  Sorting Children by Hierarchy with Closure Tables
date:   2013-07-02 21:49:28
tags: [sql,mysql,hierarchy,recursion]
excerpt: Unfortunately, this was for a well-established project built on MySQL, so the recursion and Common Table Expressions of SQL Server or PostgreSQL weren't options.
redirect_from:
  - /blog/2013/07/sorting-children-by-hierarchy-with-closure-tables/
  - /2013/07/02/sorting-children-by-hierarchy-with-closure-tables/
---
I recently ran into a problem quite familiar to developers: how to efficiently store and query a complex hierarchy several levels deep? Unfortunately, this was for a well-established project built on MySQL, so the recursion and Common Table Expressions of SQL Server or PostgreSQL weren't options.

I've worked with simple trees where a lowly Adjacency List pattern was sufficient, but this one could potentially grow very large over time and thus needed something more robust. Nested Sets seems to be all the rage, but it's so terribly inefficient at writing. I also looked at its more complex but speedy cousin, Nested Intervals, but that pattern's a bit more complicated than I was hoping. Materialized Paths didn't fit the bill either. I finally settled on Closure Tables, as described by <a href="http://stackoverflow.com/questions/192220/what-is-the-most-efficient-elegant-way-to-parse-a-flat-table-into-a-tree/192462#192462" target="_blank">Bill Karwin</a>.

The only time I got tripped up a little with closure tables was when I needed to select all nodes in a tree AND display them in the correct order.

Consider the following table of categories:

<table class="sql">
  <tr>
    <th width="10">id</th>
    <th width="20">name</th>
  </tr>
  <tr>
    <td>1</td>
    <td>Vertebrates</td>
  </tr>
  <tr>
    <td>2</td>
    <td>Invertebrates</td>
  </tr>
  <tr>
    <td>3</td>
    <td>Mammals</td>
  </tr>
  <tr>
    <td>4</td>
    <td>Dogs</td>
  </tr>
  <tr>
    <td>5</td>
    <td>Cats</td>
  </tr>
  <tr>
    <td>6</td>
    <td>Reptiles</td>
  </tr>
  <tr>
    <td>7</td>
    <td>Fish</td>
  </tr>
  <tr>
    <td>8</td>
    <td>Arthropods</td>
  </tr>
  <tr>
    <td>9</td>
    <td>Cnidarians</td>
  </tr>
  <tr>
    <td>10</td>
    <td>Spiders</td>
  </tr>
  <tr>
    <td>11</td>
    <td>Insects</td>
  </tr>
  <tr>
    <td>12</td>
    <td>Butterflies</td>
  </tr>
</table>


The closure table to describe our tree will look like this (I include the fictional root node id 0):

<table class="sql">
  <tr>
    <th>parent</th>
    <th>child</th>
    <th>depth</th>
  </tr>
  <tr>
    <td>0</td>
    <td>1</td>
    <td>1</td>
  </tr>
  <tr>
    <td>0</td>
    <td>2</td>
    <td>1</td>
  </tr>
  <tr>
    <td>0</td>
    <td>3</td>
    <td>2</td>
  </tr>
  <tr>
    <td>0</td>
    <td>4</td>
    <td>3</td>
  </tr>
  <tr>
    <td>0</td>
    <td>5</td>
    <td>3</td>
  </tr>
  <tr>
    <td>0</td>
    <td>6</td>
    <td>2</td>
  </tr>
  <tr>
    <td>0</td>
    <td>7</td>
    <td>2</td>
  </tr>
  <tr>
    <td>0</td>
    <td>8</td>
    <td>2</td>
  </tr>
  <tr>
    <td>0</td>
    <td>9</td>
    <td>2</td>
  </tr>
  <tr>
    <td>0</td>
    <td>10</td>
    <td>3</td>
  </tr>
  <tr>
    <td>0</td>
    <td>11</td>
    <td>3</td>
  </tr>
  <tr>
    <td>0</td>
    <td>12</td>
    <td>4</td>
  </tr>
  <tr>
    <td>1</td>
    <td>1</td>
    <td>0</td>
  </tr>
  <tr>
    <td>1</td>
    <td>3</td>
    <td>1</td>
  </tr>
  <tr>
    <td>1</td>
    <td>4</td>
    <td>2</td>
  </tr>
  <tr>
    <td>1</td>
    <td>5</td>
    <td>2</td>
  </tr>
  <tr>
    <td>1</td>
    <td>6</td>
    <td>1</td>
  </tr>
  <tr>
    <td>1</td>
    <td>7</td>
    <td>1</td>
  </tr>
  <tr>
    <td>2</td>
    <td>2</td>
    <td>0</td>
  </tr>
  <tr>
    <td>2</td>
    <td>8</td>
    <td>1</td>
  </tr>
  <tr>
    <td>2</td>
    <td>9</td>
    <td>1</td>
  </tr>
  <tr>
    <td>2</td>
    <td>10</td>
    <td>2</td>
  </tr>
  <tr>
    <td>2</td>
    <td>11</td>
    <td>2</td>
  </tr>
  <tr>
    <td>2</td>
    <td>12</td>
    <td>3</td>
  </tr>
  <tr>
    <td>3</td>
    <td>3</td>
    <td>0</td>
  </tr>
  <tr>
    <td>3</td>
    <td>4</td>
    <td>1</td>
  </tr>
  <tr>
    <td>3</td>
    <td>5</td>
    <td>1</td>
  </tr>
  <tr>
    <td>4</td>
    <td>4</td>
    <td>0</td>
  </tr>
  <tr>
    <td>5</td>
    <td>5</td>
    <td>0</td>
  </tr>
  <tr>
    <td>6</td>
    <td>6</td>
    <td>0</td>
  </tr>
  <tr>
    <td>7</td>
    <td>7</td>
    <td>0</td>
  </tr>
  <tr>
    <td>8</td>
    <td>8</td>
    <td>0</td>
  </tr>
  <tr>
    <td>8</td>
    <td>10</td>
    <td>1</td>
  </tr>
  <tr>
    <td>8</td>
    <td>11</td>
    <td>1</td>
  </tr>
  <tr>
    <td>8</td>
    <td>12</td>
    <td>2</td>
  </tr>
  <tr>
    <td>9</td>
    <td>9</td>
    <td>0</td>
  </tr>
  <tr>
    <td>10</td>
    <td>10</td>
    <td>0</td>
  </tr>
  <tr>
    <td>11</td>
    <td>11</td>
    <td>0</td>
  </tr>
  <tr>
    <td>11</td>
    <td>12</td>
    <td>1</td>
  </tr>
  <tr>
    <td>12</td>
    <td>12</td>
    <td>0</td>
  </tr>
</table>

So how do we select the whole tree of invertebrates? Easy:

```sql
SELECT c.*
FROM category c, closure cl
WHERE cl.child = c.id AND cl.parent = 2
```

Result:

<table class="sql">
  <tr>
    <th>id</th>
    <th>name</th>
  </tr>
  <tr>
    <td>2</td>
    <td>Invertebrates</td>
  </tr>
  <tr>
    <td>8</td>
    <td>Arthropods</td>
  </tr>
  <tr>
    <td>9</td>
    <td>Cnidarians</td>
  </tr>
  <tr>
    <td>10</td>
    <td>Spiders</td>
  </tr>
  <tr>
    <td>11</td>
    <td>Insects</td>
  </tr>
  <tr>
    <td>12</td>
    <td>Butterflies</td>
  </tr>
</table>

Hmmmnm. They're out of order. We want Spiders, Insects, and Butterflies to come right after their ancestor, Arthropods. Change the 2 to 0 in the previous to get the entire tree, and you'll notice the problem's even more obvious there.

There is no way to immediately query a closure table to sort rows by relationship. Fortunately, it's very easy to generate a string of the ancestry, i.e., a materialized path. We'll just join in the closure table a second time, use `GROUP_CONCAT()` to generate the path, and then order by path:

```sql
SELECT c.*, GROUP_CONCAT(cl2.parent ORDER BY cl2.depth DESC SEPARATOR ".") path
FROM category c
JOIN closure cl ON cl.child = c.id
JOIN closure cl2 ON cl2.child = cl.child
WHERE cl.parent = 2
GROUP BY c.id
ORDER BY path
```

The result:
<table class="sql">
  <tr>
    <th>id</th>
    <th>name</th>
    <th>path</th>
  </tr>
  <tr>
    <td>2</td>
    <td>Invertebrates</td>
    <td>0.2</td>
  </tr>

  <tr>
    <td>8</td>
    <td>Arthropods</td>
    <td>0.2.8</td>
  </tr>

  <tr>
    <td>10</td>
    <td>Spiders</td>
    <td>0.2.8.10</td>
  </tr>

  <tr>
    <td>11</td>
    <td>Insects</td>
    <td>0.2.8.11</td>
  </tr>

  <tr>
    <td>12</td>
    <td>Butterflies</td>
    <td>0.2.8.11.12</td>
  </tr>

  <tr>
    <td>9</td>
    <td>Cnidarians</td>
    <td>0.2.9</td>
  </tr>

</table>


Try it on the whole tree (parent = 0):
<table class="sql">
  <tr>
    <th>id</th>
    <th>name</th>
    <th>path</th>
  </tr>
  <tr>
    <td>1</td>
    <td>Vertebrates</td>
    <td>0.1</td>
  </tr>

  <tr>
    <td>3</td>
    <td>Mammals</td>
    <td>0.1.3</td>
  </tr>

  <tr>
    <td>4</td>
    <td>Dogs</td>
    <td>0.1.3.4</td>
  </tr>

  <tr>
    <td>5</td>
    <td>Cats</td>
    <td>0.1.3.5</td>
  </tr>

  <tr>
    <td>6</td>
    <td>Reptiles</td>
    <td>0.1.6</td>
  </tr>

  <tr>
    <td>7</td>
    <td>Fish</td>
    <td>0.1.7</td>
  </tr>

  <tr>
    <td>2</td>
    <td>Invertebrates</td>
    <td>0.2</td>
  </tr>

  <tr>
    <td>8</td>
    <td>Arthropods</td>
    <td>0.2.8</td>
  </tr>

  <tr>
    <td>10</td>
    <td>Spiders</td>
    <td>0.2.8.10</td>
  </tr>

  <tr>
    <td>11</td>
    <td>Insects</td>
    <td>0.2.8.11</td>
  </tr>

  <tr>
    <td>12</td>
    <td>Butterflies</td>
    <td>0.2.8.11.12</td>
  </tr>

  <tr>
    <td>9</td>
    <td>Cnidarians</td>
    <td>0.2.9</td>
  </tr>

</table>

Perfect.