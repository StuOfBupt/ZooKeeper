## 1. 选举机制

​		所有的candidate连接到 ZooKeeper 上然后创建一个节点用来选举，每个candidate会在该节点下创建一个顺序节点（eg.XXX0000001），从中选出一个节点作为 leader，其余节点作为 follower。当 leader 挂了之后所关联的顺序节点也会被 delete，然后再从剩余的节点中选出一个作为新的 leader

## 2.第三方包

```go
import (
  "github.com/Comcast/go-leaderelection"
	"github.com/samuel/go-zookeeper/zk"
)
```

## 3.Demo

```go
// leader 执行的函数
func doStuff(leaderElector *leaderelection.Election, wg *sync.WaitGroup) {
	defer wg.Done()
	count := 0
	fmt.Println("**DoSutff Start**")
	for {
		select {
		case <- time.After(time.Second * 3):
			fmt.Println("Leader is dong something....")
			count += 1
			if count == 30{
        // leader 工作一段时间主动退出
				fmt.Println("Leader is Resign.....")
				leaderElector.Resign()
				return
			}

		case status, ok := <- leaderElector.Status():
			if !ok {
				fmt.Println("\t\t\tChannel closed while Leading, candidate <", status.CandidateID, "> resigning.")
				leaderElector.Resign()
				return
			}else if status.Err != nil {
				fmt.Println("\t\t\tError detected in while leading, candidate <", status.CandidateID, "> resigning. Error was <",
					status.Err, ">.")
				leaderElector.Resign()
				return
			}else {
				fmt.Println("\t\t\tUnexpected status notification while Leading, candidate <", status.CandidateID, "> resigning.")
				leaderElector.Resign()
				return
			}
		}
	}
}

func main() {
	var hosts = []string{"127.0.0.1:2181"}
	electionPath := "/ElectionNode"
	zkConn, eventCh, err := zk.Connect(hosts, time.Second * 10)
	if err != nil {
		fmt.Println("##Fail to connect zk")
		return
	}
	fmt.Println("Connect Succeed")
	defer zkConn.Close()

	exists, _, _ := zkConn.Exists(electionPath)
	if !exists {
		_, err = zkConn.Create(electionPath,  make([]byte, 0), 0, zk.WorldACL(zk.PermAll))
		if err != nil {
			fmt.Println("##Fail to create election node:", err)
			return
		}
	}

	leaderElector, err := leaderelection.NewElection(zkConn, electionPath, "my_instance")
	if err != nil {
		fmt.Println("##Can't create a new election")
		return
	}
	go leaderElector.ElectLeader()
	wg := sync.WaitGroup{}
	wg.Add(1)
	for {
		select {
		case status, ok := <- leaderElector.Status():
			if !ok {
				fmt.Println("## Election is terminated:")
				leaderElector.Resign()
				return
			}
			if status.Err != nil {
				fmt.Printf("Received election status error: %v for candidate: %v\n", status.Err, status.CandidateID)
				leaderElector.Resign()
				return
			}
			if status.Role == leaderelection.Leader {
				fmt.Println("I'm leader ")
				go doStuff(leaderElector, &wg)
				goto WAIT
			}else if status.Role == leaderelection.Follower {
				fmt.Println("I'm follower ")
			}
		case event, ok := <- eventCh:
			if !ok {
				fmt.Println("eventCh is closed")
				return
			}
			if event.State == zk.StateDisconnected {
        // zk.Conn 自带重连功能，当 zk 挂掉恢复后仍会重新连接
        // 在此期间原有 leader 继续工作
				fmt.Println("****Disconnected****")
			}
		}
	}
	WAIT:
	wg.Wait()
}
```

## 4.API

#### 1.连接 ZooKeeper

```go
zkConn, eventCh, err := zk.Connect(hosts, time.Second * 10)
/**
* zkConn 		连接实例
* eventCh 	事件信道，用于监听 zk 连接状态变化
* err				错误信息
*/
```

#### 2.创建节点

```go
exists, _, _ := zkConn.Exists(electionPath)
if !exists {
		_, err = zkConn.Create(electionPath,  make([]byte, 0), 0, zk.WorldACL(zk.PermAll))
		if err != nil {
			fmt.Println("##Fail to create election node:", err)
			return
		}
}
/**
*	选举前需要创建一个永久节点用于选举
*/
```

#### 3. 发起选举

```go
leaderElector, err := leaderelection.NewElection(zkConn, electionPath, "my_instance")
if err != nil {
		fmt.Println("##Can't create a new election")
		return
}
go leaderElector.ElectLeader()
/**
*	NewElection 		用于初始化一个选举实例，需要一个 zkConn 实例以及选举的节点路径
*									返回Election 实例
*	ElectionLeader	参与者进行选举，并决定参与者的角色（leader/follower），需要放在协程中
*/
```

#### 4.监听状态

```go
for {
  select {
    case status, ok := <- leaderElection.Status():
  }
}
/**
*	status.Role : Leader/Follower
*	status.Err 	:	发生的一些错误信息 
*	ok : 当参与者 Resign 或者leader 终止选举，则关闭 Status 信道
*/
```

#### 5.结束选举

```go
leaderElection.EndElection()
/**
*	成为 leader 后调用，通知本次选举结束
*	将清除当前选举节点的所有资源
* 【不恰当的调用会导致多主现象】
*/
```

## 5.一些坑

#### 1.多主现象的产生

调用了 EndElection，导致一次选举直接结束，然后第一个启动的程序成为 leader，随后启动的程序重新发起选举，因此也成为了“leader”，导致启动的程序都成为了 leader。

期间不应该调用EndElection，Election 节点下的顺序节点不会被清空，直到 leader 挂了之后所在的顺序节点 deleted 然后其他 follower 继任成为新的 leader

#### 2.选举期间 ZK 挂了

zk.Connect有自动重连功能，因此demo 程序在 zk 挂了之后 leader 继续执行 doStuff，然后 zk 恢复之后所有 follower 仍然保持之前的状态，直到 leader 工作结束调用 Resign（或者 leader 挂了），通过 leaderElection.Status 继任为新的 leader 执行 doStuff。