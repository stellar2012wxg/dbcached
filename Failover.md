### dbcached 的故障转移支持、设计方向以及与 Memcachedb 的不同之处 ###

dbcached 与新浪的另一开源产品 Memcachedb 一样，都支持 Memcached 协议，具有持久化存储功能，但两者有不同之处：

#### 1、故障转移 ####

先引用 memcached 官方网站和 PHP 手册中的两段话：

http://www.danga.com/memcached/

If a host goes down, the API re-maps that dead host's requests onto the servers that are available.

http://cn.php.net/manual/zh/function.Memcache-addServer.php

Failover may occur at any stage in any of the methods, as long as other servers are available the request the user won't notice. Any kind of socket or Memcached server level errors (except out-of-memory) may trigger the failover. Normal client errors such as adding an existing key will not trigger a failover.

再以一个例子来说明：
```
<?php
/* OO API */
$memcache = new Memcache;
$memcache->addServer('192.168.0.5', 11211);
$memcache->addServer('192.168.0.6', 11211);
$memcache->addServer('192.168.0.7', 11211);
$memcache->addServer('192.168.0.8', 11211);
?>
```
当 Memcache 客户端使用 addServer 服务器池时，是根据“crc32(key) % current\_server\_num”哈希算法将 key 哈希到不同的服务器的，PHP、C 和 python 的客户端都是如此的算法。

Memcache 客户端的 addserver 具有故障转移机制，当 addserver 了4台 Memcached 服务器，而其中一台宕机了，那么 current\_server\_num 会由原先的4变成3，这时就会重新哈希了。

举个例子，假设在4台服务器都正常的情况下，key 值 aaa 被哈希到服务器 192.168.0.5 上，如果这时 192.168.0.7 宕机了，key 值 aaa 就可能就会被重新哈希到服务器 192.168.0.6 上，这时就找不到数据了。当然，假设 key 值 aaa 两次哈希的值是一样，它将仍被哈希到 192.168.0.5 上，但是不管怎样，都会导致一部分数据丢失。

Memcachedb 可以保证数据的持久化存储，但目前还没有解决 Memcache 服务器池故障转移导致的数据丢失。而 dbcached 可以，它在未命中时会请求后端的 NMDB 取回数据。在接下来的版本中，dbcached 后端的 NMDB 将设计为两台，进行互备，届时无论前端 dbcached 中的某几台挂了还是后端 NMDB 中的一台挂了，数据都不会丢失。

#### 2、设计方向 ####

dbcached 和 Memcachedb 的设计方向不同，dbcached 的设计方向是发挥 Memcached 的内存缓存性能优势，使之成为一个具有“故障转移”、“数据持久化存储”、“多服务器同时读写”的高并发内存缓存系统，它是围绕 Memcached 进行开发的。而 Memcachedb 只使用了 Memcached 的协议和网络层，抛弃了 Memcached 的内存管理部分，使用 Berkeley DB 数据库自身的缓存来实现，是围绕 Berkeley DB 进行开发的，目前支持类似 MySQL 主辅库同步的方式实现读写分离，支持“主服务器可读写、辅助服务器只读”模式。