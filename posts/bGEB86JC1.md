---
title: 'Redis的缓存淘汰策略LRU'
date: 2021-09-03 16:38:27
tags: [redis]
published: true
hideInList: false
feature: /post-images/bGEB86JC1.png
isTop: false
---
redis缓存淘汰策略与Redis键的过期删除策略并不完全相同，前者是在Redis内存使用超过一定值的时候（一般这个值可以配置）使用的淘汰策略；而后者是通过定期删除+惰性删除两者结合的方式进行内存淘汰的。



### Redis内存不足的缓存淘汰策略

- noeviction：当内存使用超过配置的时候会返回错误，不会驱逐任何键
- allkeys-lru：加入键的时候，如果过限，首先通过LRU算法驱逐最久没有使用的键
- volatile-lru：加入键的时候如果过限，首先从设置了过期时间的键集合中驱逐最久没有使用的键
- allkeys-random：加入键的时候如果过限，从所有key随机删除
- volatile-random：加入键的时候如果过限，从过期键的集合中随机驱逐
- volatile-ttl：从配置了过期时间的键中驱逐马上就要过期的键
- volatile-lfu：从所有配置了过期时间的键中驱逐使用频率最少的键
- allkeys-lfu：从所有键中驱逐使用频率最少的键



### lru算法实现

#### 取巧算法

```java

import java.util.LinkedHashMap;
import java.util.Map;

class LRUCache<K,V> extends LinkedHashMap<K,V> {

    private int capacity;

    /**
     * the ordering mode
     * - <tt>true</tt> for access-order,
     * - <tt>false</tt> for insertion-order
     * @param capacity
     */
    public LRUCache(int capacity) {
        super(capacity, 0.75F, true);
        this.capacity = capacity;
    }

    //数据超过容量大小删除
    @Override
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return super.size()>capacity;
    }

}
```

#### 数据结构实现

```java

import java.util.HashMap;
import java.util.Map;

class LRUCacheDemo {

    //构建承载体node
    class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev;
        Node<K, V> next;

        public Node() {
            this.prev = this.next = null;
        }

        public Node(K key, V value) {
            this.key = key;
            this.value = value;
            this.prev = this.next = null;
        }
    }

    //构建双向队列
    class DoubleLinkedList<K, V> {
        Node<K, V> head;
        Node<K, V> tail;

        public DoubleLinkedList() {
            head = new Node<>();
            tail = new Node<>();
            head.next = tail;
            tail.prev = head;
        }

        //添加头节点
        public void addHead(Node<K, V> node) {
            node.next = head.next;
            node.prev = head;
            head.next.prev = node;
            head.next = node;
        }

        //删除节点
        public void removeNode(Node<K, V> node) {
            node.next.prev = node.prev;
            node.prev.next = node.next;
            node.prev = null;
            node.next = null;
        }

        //获取最后一个节点
        public Node getLast() {
            return tail.prev;
        }
    }

    private int cacheSize;
    Map<Object, Node<Object, Object>> map;
    DoubleLinkedList<Object, Object> doubleLinkedList;

    public LRUCacheDemo(int cacheSize) {
        this.cacheSize = cacheSize;
        this.map = new HashMap<>();
        this.doubleLinkedList = new DoubleLinkedList<>();
    }

    public Object get(Object key) {
        if (!map.containsKey(key)) {
            return -1;
        }
        final Node<Object, Object> node = map.get(key);
        this.doubleLinkedList.removeNode(node);
        this.doubleLinkedList.addHead(node);
        return node.value;
    }

    public void put(Object key, Object value) {
        if (map.containsKey(key)) {
            final Node<Object, Object> node = map.get(key);
            node.value = value;
            map.put(key, node);
            this.doubleLinkedList.removeNode(node);
            this.doubleLinkedList.addHead(node);
        }else{
            if(map.size()==this.cacheSize){
                final Node last = this.doubleLinkedList.getLast();
                map.remove(last.key);
                doubleLinkedList.removeNode(last);
            }
            //新增
            Node<Object,Object> newNode=new Node<>(key,value);
            map.put(key, newNode);
            this.doubleLinkedList.addHead(newNode);
        }
    }

}
```

