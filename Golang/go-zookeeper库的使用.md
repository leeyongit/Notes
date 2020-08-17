go-zookeeper库的使用
---
## go-zookeeper库使用
[samuel/go-zookeeper/zk]是zookeeper的golang客户端，也是个人感觉目前最好用的ZK-golang的客户端了，本文档主要介绍该库的使用方法，并给出自己使用的一个简单事例，至于该库的内部实现，可能会在其他文档中有所阐述。


### 使用事例
```go
package main

import (
    "fmt"
    "github.com/samuel/go-zookeeper/zk"
    "sync"
    "time"
)

// conenct zookeeper cluster
func connect_zk() (*zk.Conn, error) {
    c, _, err := zk.Connect([]string{"127.0.0.1:2182"}, time.Second*1)
    // c, session, err := zk.Connect([]string{"127.0.0.1:2182"}, time.Second*1)
    if err != nil {
        return nil, err
    }
    return c,nil
}

func delete_node(c *zk.Conn, nodePath string) {
    c.Delete(nodePath, -1)
}

func create_node(c *zk.Conn, nodePath string) error {
    if _, err := c.Create(nodePath, []byte{1, 2, 3, 4}, 0, zk.WorldACL(zk.PermAll)); err != nil {
        fmt.Printf("Create returned error: %+v\n", err)
        return err
    }
    return nil
}

func get_node(c *zk.Conn, path string) {
    data, stat, err := c.Get(path)
    if err != nil {
        fmt.Printf("Get returned error: %+v\n", err)
    } else {
        fmt.Printf("Get node, data: %+v, state: %+v\n", data, stat)
    }
}

func watch_node(c *zk.Conn) {
    var wg sync.WaitGroup
    path := "/gozk-test-2"

    delete_node(c, path)

    _, _, childCh, _ := c.ChildrenW("/")
    /*
    if err != nil {
        fmt.Printf("Watch error: %v\n", err)
        return
    }
    */

    wg.Add(1)
    go Watcher(wg, childCh)

    create_node(c, path)

    /*
    if path, err = c.Create("/gozk-test-2", []byte{1, 2, 3, 4}, 0, zk.WorldACL(zk.PermAll)); err != nil {
        fmt.Printf("Creat node: %v error: %v\n", "/gozk-test-2", err)
        return
    } else if path != "/gozk-test-2" {
        fmt.Printf("Create returned different path '%s' != '%s'", path, "/gozk-test-2")
        return
    }
    */

    // add watcher for new added node
    _, _, addCh, _ := c.ChildrenW("/gozk-test-2")
    wg.Add(1)
    go Watcher(wg, addCh)

    delete_node(c, path)

    wg.Wait()
}

func main() {
    path := "/gozk-test"
    // connect zk-server
    c, err := connect_zk()
    if err != nil {
        return
    }

    // create zk-node, delete it if already exist
    delete_node(c, path)
    err = create_node(c, path)
    if err != nil {
        return
    }

    get_node(c, path)

    watch_node(c)
}

func Watcher(wg sync.WaitGroup, childCh <-chan zk.Event) {
    defer wg.Done()
    select {
    case ev := <-childCh:
        if ev.Err != nil {
            fmt.Printf("Child watcher error: %+v\n", ev.Err)
            return
        }
        fmt.Printf("Receive event: %+v\n", ev)
    case _ = <-time.After(time.Second * 2):
        fmt.Printf("Child Watcher timeout")
        return
    }
}
```

### 运行结果

    Get node, data: [1 2 3 4], state: &{Czxid:115 Mzxid:115 Ctime:1406017529127 Mtime:1406017529127 Version:0 Cversion:0 Aversion:0 EphemeralOwner:0 DataLength:4 NumChildren:0 Pzxid:115}
    Receive event: {Type:EventNodeChildrenChanged State:StateSyncConnected Path:/ Err:<nil>}
    Receive event: {Type:EventNodeDeleted State:StateSyncConnected Path:/gozk-test-2 Err:<nil>}

[samuel/go-zookeeper/zk]:https://github.com/samuel/go-zookeeper.git
