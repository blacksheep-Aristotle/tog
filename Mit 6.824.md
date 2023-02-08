# Mit 6.824



## lab2：Raft



### A：选举



Raft使用心跳机制来触发领导者选举。当服务器启动时，它们开始是追随者。只要服务器收到来自领导者或候选人的有效RPC，它就一直处于跟随者状态。领导者定期向所有追随者发送心跳（AppendEntries RPCs，不携带任何日志条目），以保持他们的权威。如果追随者在一段被称为选举超时的时间内没有收到任何通信，那么它就认为没有可行的领导者，并开始选举以选择一个新的领导者。



为了开始选举，追随者增加它的当前任期并过渡到候选状态。然后，它为自己投票，并向集群中的每个其他服务器发出RequestVote RPCs。候选者继续处于这种状态，直到发生三种情况之一。(a)它赢得了选举，(b)另一个服务器确立了自己的领导地位，或者(c)一段时间内没有赢家。这些结果将在下面的段落中分别讨论。



如果一个候选人在**同一任期**内获得了整个集群中大多数服务器的投票，那么它就赢得了选举。每台服务器在给定的任期内最多为一名候选人投票，以先来后到为原则（注：第5.4节对投票增加了一个额外的限制）。少数服从多数的原则保证了最多只有一名候选人能够在某一任期内赢得选举（图3中的选举安全属性）。一旦一个候选人在选举中获胜，它就成为领导者。然后，它向所有其他服务器发送心跳信息，以建立其权威并防止新的选举。



在等待投票的过程中，候选人可能会收到另一个服务器的AppendEntries RPC，声称自己是领导者。**如果领导者的任期（包括在其RPC中）至少与候选人的当前任期一样大，那么候选人就会承认领导者是合法的，并返回到跟随者状态。**如果RPC中的任期比候选者当前的任期小，那么候选者拒绝RPC，继续处于候选状态。



第三个可能的结果是，一个候选人既没有赢得选举，也没有失去选举：如果许多追随者同时成为候选人，选票可能被分割，因此没有候选人获得多数。当这种情况发生时，每个候选人都会超时，并通过增加其任期和启动新一轮的请求投票RPC来开始新的选举。然而，如果没有额外的措施，分裂的投票可能会无限期地重复。



  Raft使用随机的选举超时，以确保分裂投票很少发生，并能迅速解决。为了从一开始就防止分裂投票，选举超时是从一个固定的时间间隔中随机选择的（例如，150-300ms）。这就分散了服务器，所以在大多数情况下，只有一个服务器会超时；它赢得了选举，并在任何其他服务器超时之前发送心跳信号。同样的机制被用来处理分裂投票。每个候选人在选举开始时重新启动其随机的选举超时，并等待超时过后再开始下一次选举；这减少了在新的选举中再次出现分裂票的可能性。第9.3节显示，这种方法可以迅速选出一个领导者。



选举是一个例子，说明可理解性是如何指导我们在设计方案之间进行选择的。最初我们计划使用一个排名系统：每个候选人被分配一个独特的排名，用来在竞争的候选人之间进行选择。如果一个候选人发现了另一个排名更高的候选人，它就会回到追随者的状态，这样排名更高的候选人就能更容易地赢得下一次选举。我们发现这种方法在可用性方面产生了一些微妙的问题（如果一个排名较高的服务器失败了，一个排名较低的服务器可能需要超时并再次成为候选人，但如果它过早地这样做，它可能会重置选举领导者的进展）。我们对算法进行了多次调整，但每次调整后都会出现新的边界案例。最终我们得出结论，随机重试的方法更加明显和容易理解。

![image-20221003220622275](Mit 6.824/image-20221003220622275.png)

![image-20221003220633004](Mit 6.824/image-20221003220633004.png)

如果没有选出leader，那么candidate会随机选取150～300ms的时间再发起选举-》增加自己任期号，向其他节点发出投票rpc。。。。

![image-20221003220645664](Mit 6.824/image-20221003220645664.png)

节点通过任期号来确定自己的状态，并拒绝接收低于自己任期号的rpc

**如果投票者的日志比candidate还新，会拒绝该投票请求**

Lastlogindex：

- 如果term相同，则index更长的为新

Laslogterm：

- 如果term不同，term更大的为新

follower会根据任期号和logindex和logterm决定是否投票，**每个follower只有一张票，按照先来先得投票**

每秒发送心跳RPC不超过十次。

测试人员要求您的Raft在旧领导失败后的五秒钟内选出新领导（如果大多数同事仍能沟通）。然而，请记住，在分裂投票的情况下，领导人选举可能需要多轮投票（如果数据包丢失或候选人不幸选择了相同的随机退选时间，则可能发生这种情况）。您必须选择足够短的选举超时（以及心跳间隔），即使需要多轮投票，选举也很可能在5秒内完成。

论文第5.2节提到了150到300毫秒的选举超时。只有当领导者发送心跳的频率远高于每150毫秒一次时，这样的范围才有意义。因为测试仪限制你每秒心跳10次，所以你必须使用大于论文150到300毫秒的选举超时，但不要太大，因为这样你可能无法在5秒内选出一位领导人。

**严格遵守Figure 2**

- 只有在以下三个场景重置定时器：a）你从当前的领导者那里得到一个AppendEntries RPC（即，如果AppendEntries参数中的任期已经过时，你不应该重启你的计时器）；b）你正在开始一个选举；或者c）你授予另一个对等体一个投票。
- 不论你的状态是什么，如果RPC请求或响应包含任期 T > currentTerm：设置currentTerm = T，转换为follower
- 不要将keep-alive和append分开处理！他们的处理逻辑应该是一样的



<img src="Mit 6.824/image-20221007234019580.png" alt="image-20221007234019580" style="zoom: 67%;" />



<img src="Mit 6.824/image-20221008085911265.png" alt="image-20221008085911265" style="zoom:80%;" />



<img src="Mit 6.824/image-20221007234042011.png" alt="image-20221007234042011" style="zoom:67%;" />



```go

//leader维持心跳
func (rf* Raft) Keepalive()  {
	args:=AppendEntriesArgs{Term: rf.term,Leaderid:rf.me}

	for i:=0;i< len(rf.peers);i++{
		if i!=rf.me{
			reply:=AppendEntriesReply{}
			go rf.sendAppendEntry(i,&args,&reply)
		}
	}
}

func (rf *Raft) sendAppendEntry(server int, args *AppendEntriesArgs, reply *AppendEntriesReply) bool {
	ok := rf.peers[server].Call("Raft.RequestApp", args, reply)
	for !ok{
		if rf.killed() {
			return false
		}
		ok=rf.peers[server].Call("Raft.RequestApp", args, reply)
	}
	//对于leader而言，如果reply的term》leader，说明leader已经过期了
	rf.mu.Lock()
	defer rf.mu.Unlock()
	if reply.Term>rf.term{
		rf.statue=follower
		return reply.Success
	}

	return reply.Success
}

//对append的回应：注意：不要将心跳和日志append分开处理！！！！
func (rf *Raft) RequestApp(args *AppendEntriesArgs, reply *AppendEntriesReply)  {

	rf.Log("receive a keep-alive from leader %v which term %v",args.Leaderid,args.Term)

	rf.mu.Lock()
	defer rf.mu.Unlock()
	if args.Term>rf.term{
		rf.Updateterm(args.Term)
	}
	if args.Term>=rf.term{
		rf.statue=follower
		//收到心跳包
		rf.Upelection()

		reply.Success=true
		reply.Term=rf.term
	}else{
		//如果收到term比我低的心跳包，要重置时间吗？应该不用吧。。。
		reply.Term=rf.term
		reply.Success=false
	}

}
```



<img src="Mit 6.824/image-20221008085557088.png" alt="image-20221008085557088" style="zoom:80%;" />

```go
//开始选举，如果每当选，说明都没发送，等待下次ticker
func(rf*Raft)  Leaderelection() {
	rf.statue=candidate
	var  args=RequestVoteArgs{rf.term,rf.me,0,0}

	for i := range rf.peers {
		if i != rf.me {
			var  reply = RequestVoteReply{}
			//应该要并发处理，否则，发完一个就卡住了！！！
			go rf.sendRequestVote(i, &args, &reply)
		}
	}
}
////：a）你从当前的领导者那里得到一个AppendEntries RPC（即，如果AppendEntries参数中的任期已经过时，你不应该重启你的计时器）；
//	//b）你正在开始一个选举；或者c）你授予另一个对等体一个投票。
func(rf*Raft) Upelection() {
	t:=time.Duration(150+rand.Intn(200))* time.Millisecond
	rf.electiontime=time.Now().Add(t)
}
//更新任期
func(rf *Raft) Updateterm(t int)  {
	rf.term=t
	rf.tstatue.Isvote=false
	rf.tstatue.Votenum=0
	//更新时间
	//rf.tstatue.Time
}
func Vote(rf*Raft,reply*RequestVoteReply)  {
	if rf.tstatue.Isvote{
		reply.Vote=false
		//三种情况之一：投出票时重置
	}else{
		rf.Log("vote to ")
		reply.Vote=true
		rf.tstatue.Isvote=true
		rf.Upelection()
	}
}
//
// example RequestVote RPC handler.
//
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
	// Your code here (2A, 2B).
	rf.Log("recieve a vote which come from %v",args.Candidate)
	//如果arg的term大于我的term，reply=args.term，如果arg的term小于我的term，不改直接发
	rf.mu.Lock()
	defer rf.mu.Unlock()

	reply.Term=rf.term
	switch rf.statue {

	case leader:
		//收到大于自己任期号，说明leader落后了,投票然后转为follower
		if args.Term>rf.term{
			rf.statue=follower
			rf.Updateterm(args.Term)
			reply.Term=rf.term

			Vote(rf,reply)
		}else {
			reply.Vote=false
		}
	case candidate:
		//如果candidate收到了比他任期号还大的请求，降级为follower
		if args.Term>rf.term{
			rf.statue=follower
			rf.Updateterm(args.Term)
			reply.Term=rf.term
			Vote(rf,reply)
		}else {
			reply.Vote=false
		}
	case follower:
		//如果任期号相等,是不投票的！
		if args.Term>rf.term{  //说明进入下一次任期了
			//reply.term=rf.term
			rf.Updateterm(args.Term)
			reply.Term=rf.term
			Vote(rf,reply)
		}else {
			reply.Vote=false
		}

	}
}
```





#### 发送投票

以40ms为周期，如果超时了，开始选举，如果是leader，发送心跳包。

```go

//ticker会以心跳为周期不断检查状态。如果当前是Leader就会发送心跳包，而心跳包是靠appendEntries()发送空log
// The ticker go routine starts a new election if this peer hasn't received
// heartsbeats recently.
func (rf *Raft) ticker() {

   for rf.killed() == false {

      time.Sleep(40 * time.Millisecond)

      rf.mu.Lock()

      if rf.statue==leader{
         //rf.Log("start keep-alive")
         rf.Keepalive()  //keep-alive如果有新加的条目，就一次性发送
         //如果现在的时间超出选举时间，说明超时了，开始选举
      }else if time.Now().After(rf.electiontime) {

         rf.statue=candidate
         rf.Updateterm(rf.term+1)
         rf.tstatue.Votenum+=1
         rf.tstatue.Isvote=true
         rf.Upelection()

         rf.Log("start election")

         rf.Leaderelection()
      }

      rf.mu.Unlock()


   }
}
```

将candidate当前的任期号，最后的日志下标和最后的日志任期发送给其他节点。

```go
RequestVoteArgs{rf.term,rf.me,rf.getLastIndex(),rf.getLastTerm()}
```

当前任期号用于实现任期大的优先当选。

日志长度和日志任期用于其他节点判断拥有的日志是否一致，尽量选出日志多的节点当leader。

```go

func (rf *Raft) sendRequestVote(server int, args *RequestVoteArgs, reply *RequestVoteReply) bool {

	ok := rf.peers[server].Call("Raft.RequestVote", args, reply)
	if ok==false||rf.killed() {
		return false
	}


	rf.mu.Lock()
	defer rf.mu.Unlock()

	switch rf.statue {
	//对于leader而言，如果reply的term》leader，说明leader已经过期了
	case leader:
		if reply.Term>rf.term{
			rf.Log("receive a higher reply %v , become follower",reply.Term)
			rf.statue=follower
			rf.Updateterm(reply.Term)
			return reply.Vote
		}
	case candidate:
		//如果reply的任期比我还高，那么candidate转为follower，停止投票
		if reply.Term>rf.term{

			rf.Log("receive a higher reply %v , become follower",reply.Term)
			rf.statue=follower
			//如果收到比我任期还大的，不用重设超时时间
			rf.Updateterm(reply.Term)
			return reply.Vote
		}
		if reply.Vote {

			rf.tstatue.Votenum+=1
			if rf.tstatue.Votenum>=(len(rf.peers)/2+1){

				rf.tstatue.Votenum=0   //防止多次唤醒

				rf.statue=leader

				ll:=rf.getLastIndex()
				for i := range rf.peers  {
					rf.NextIndex[i]=ll+1
				}
				rf.matchIndex[rf.me]=ll

				rf.Log("Become leader!!!! leader lastlog index %v , leader commit %v",rf.getLastIndex(),rf.Commitindex)
				//新leader的next下标应该是最大值，否则从0开始发送，网络负担太大
				//rf.NextIndex=make([]int,len(rf.peers))
				rf.Keepalive()

				//向所有节点发送keep-alive
				return reply.Vote
			}
		}


	}
	return reply.Vote
}
```



#### 收到投票

**只有在以下三个场景重置定时器：a）你从当前的领导者那里得到一个AppendEntries RPC（即，如果AppendEntries参数中的任期已经过时，你不应该重启你的计时器）；b）你正在开始一个选举；或者c）你授予另一个对等体一个投票。**



```go

func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
	// Your code here (2A, 2B).

	//如果arg的term大于我的term，reply=args.term，如果arg的term小于我的term，不改直接发
	rf.mu.Lock()
	defer rf.mu.Unlock()

	rf.Log("recieve a vote which come from %v",args.Candidate)
	reply.Term=rf.term

	if args.Term>rf.term{
		rf.statue=follower
		rf.Updateterm(args.Term)
		rf.Log("a higher vote,Become follower")
	}
	if rf.statue==follower{
		if args.Term>=rf.term {
			//如果candidate的最后日志任期比我的小，拒绝投票
			if args.Lastlogterm<rf.getLastTerm(){
				rf.Log("Vote false on LastTerm who come from %v his loglastterm %v",args.Candidate,args.Lastlogterm)
				reply.Vote=false
				reply.Term=rf.term
				return
            //如果最后的日志任期一样，比谁的日志更多
			}else if args.Lastlogterm==rf.getLastTerm(){
				if args.Lastlogindex<rf.getLastIndex(){
					rf.Log("Vote false on LastIndex who come from %v his Lastlogindex %v",args.Candidate,args.Lastlogindex)
					reply.Vote=false
					reply.Term=rf.term
					return
				}else {
					rf.Log("Vote pass yizhixing")
					reply.Term=rf.term
					Vote(rf,args,reply)
				}
            //如果最后日志任期比我大，不用比下标，直接投票
			}else {
				rf.Log("Vote pass yizhixing")
				reply.Term=rf.term
				Vote(rf,args,reply)

			}
		}
	}

}
```



#### Bug：

git reset --hard

ticker没有响应！！！，打印日志发现超时了，但是却没有下一步动作。并没有进入选举循环

**打印日志发现是超时时间太久了？**

最后发现是醒来之后

```go
		if rf.killed()==false{
			return
		}
```

这行语句使程序直接返回了！

```go
type raft.RequestVoteArgs has no exported fields
```

**原来是结构体元素没有大写开头！！！**



### B：日志和安全性

####  5.3 Log replication 

<img src="Mit 6.824/image-20230124205057815.png" alt="image-20230124205057815" style="zoom:67%;" />

图6：日志是由条目组成的，这些条目按顺序编号。每个条目都包含创建它的任期（每个方框中的数字）和状态机的命令。如果一个条目可以安全地应用于状态机，那么该条目就被认为是承诺的。



一旦一个领导者被选出，它就开始为客户端请求提供服务。每个客户端请求都包含一个要由复制的状态机执行的命令。领导者将命令作为一个新的条目附加到它的日志中，然后它使用AppendEntries RPCs平行于其他每个服务器来复制该条目。当条目被安全复制后（如下所述），领导者将条目应用于其状态机，并将执行结果返回给客户端。如果跟随者崩溃或运行缓慢，或者网络数据包丢失，领导者会无限期地重试Append Entries RPCs（甚至在它响应了客户端之后），直到所有跟随者最终存储所有的日志条目。



日志的组织方式如图6所示。每个日志条目都存储了一个状态机命令，以及领导者收到该条目时的任期编号。**日志条目中的任期编号被用来检测日志之间的不一致**，并确保图3中的一些属性。每个日志条目也有一个整数索引，用于识别它在日志中的位置。



领导决定何时将日志条目应用于状态机是安全的；这样的条目被称为承诺。Raft保证所提交的条目是持久的，最终会被所有可用的状态机执行。一旦创建该条目的领导者将其复制到大多数服务器上，该日志条目就会被提交（例如，图6中的条目7）。这也会提交领导者日志中所有之前的条目，包括之前领导者创建的条目。第5.4节讨论了在领导者变更后应用这一规则时的一些微妙之处，它还表明这种承诺的定义是安全的。领导者会跟踪它所知道的已承诺的最高索引，并且它在未来的AppendEntries RPC（包括心跳）中包括该索引，以便其他服务器最终获知。一旦跟随者得知一个日志条目被提交，它就会将该条目应用到它的本地状态机（按日志顺序）。



我们设计的Raft日志机制在不同服务器上的日志之间保持高度的一致性。这不仅简化了系统的行为，使其更具可预测性，而且是确保安全的重要组成部分。Raft维护以下属性，它们共同构成了图3中的日志匹配属性：

<img src="https://mmbiz.qpic.cn/mmbiz_png/GafMH4lpVp7fa986NDY0Qw6TgX5WtHlfxYXSpX6mm0gPNUuZZ0fbMlRbbIKttoRB6UKRjLbFgFQU6rlwuLHqog/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1" alt="图片" style="zoom:67%;" />

| **Election Safety**: at most one leader can be elected in a given term. §5.2 选举安全：在一个特定的任期内最多可以选出一名领导人。§5.2 |
| ------------------------------------------------------------ |
| **Leader Append-Only**: a leader never overwrites or deletes entries in its log; it only appends new entries. §5.3 Leader Append-Only：领导者从不覆盖或删除其日志中的条目；它只附加新条目。§5.3 |
| **Log Matching**: if two logs contain an entry with the same index and term, then the logs are identical in all entries up through the given index. §5.3 **日志匹配：如果两个日志包含一个具有相同索引和任期的条目，那么这两个日志的所有条目到给定索引都是相同的。**§5.3 |
| **Leader Completeness**: if a log entry is committed in a given term, then that entry will be present in the logs of the leaders for all higher-numbered terms. §5.4 领导者的完整性：如果一个日志条目在某一任期中被承诺，那么该条目将出现在所有更高编号任期的领导者的日志中。§5.4 |
| **State Machine Safety**: if a server has applied a log entry at a given index to its state machine, no other server will ever apply a different log entry for the same index. §5.4.3 状态机安全：如果一个服务器在其状态机上应用了一个给定索引的日志条目，那么其他服务器将永远不会为同一索引应用不同的日志条目。§5.4.3 |

• **If two entries in different logs have the same index and term, then they store the same command.** 

**如果不同日志中的两个条目具有相同的索引和任期，那么它们存储的是同一个命令。**



**• If two entries in different logs have the same index and term, then the logs are identical in all preceding entries.** 

**如果不同日志中的两个条目具有相同的索引和任期，那么日志中的所有前面的条目都是相同的。**



第一个属性来自于这样一个事实，即一个领导者在一个给定的任期中最多创建一个具有给定日志索引的条目，并且日志条目永远不会改变它们在日志中的位置。第二个属性由AppendEntries执行的简单一致性检查来保证。当发送AppendEntries RPC时，领导者包括其日志中紧接新条目之前的条目的索引和任期。如果跟随者在其日志中没有找到具有相同索引和任期的条目，那么它将拒绝新条目。一致性检查作为一个归纳步骤：日志的初始空状态满足了日志匹配属性，并且每当日志被扩展时，一致性检查都会保留日志匹配属性。因此，每当AppendEntries成功返回时，领导者知道跟随者的日志与自己的日志在新条目之前是相同的。



在正常运行期间，领导者和追随者的日志保持一致，所以AppendEntries一致性检查从未失败。然而，领导者崩溃会使日志不一致（老领导者可能没有完全复制其日志中的所有条目）。这些不一致会在一系列领导者和追随者的崩溃中加剧。图7说明了追随者的日志可能与新领导者的日志不同的方式。跟随者可能会丢失领导者的条目，可能会有领导者没有的额外条目，或者两者都有。日志中的缺失和不相干的条目可能跨越多个任期。



在Raft中，领导者通过强迫追随者的日志重复自己的日志来处理不一致的情况。这意味着追随者日志中的冲突条目将被领导者日志中的条目覆盖。第5.4节将表明，如果再加上一个限制，这就是安全的。



![图片](https://mmbiz.qpic.cn/mmbiz_png/GafMH4lpVp7fa986NDY0Qw6TgX5WtHlfXbr7139eosons6gGk8xgQo63DicXf8Ntb9h0ZvCkDFWpy2y6qp3l3KQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



图7：当最上方的领导者掌权时，在追随者的日志中可能会出现（a-f）的任何一种情况。每个盒子代表一个日志条目；盒子里的数字是它的任期。一个追随者可能缺少条目（a-b），可能有额外的未承诺的条目（c-d），或者两者都有（e-f）。例如，场景f会发生在如果该服务器是第2期的领导者，在其日志中增加了几个条目，然后在提交任何条目之前就崩溃了；它很快重新启动，成为第3期的领导者，并在其日志中增加了几个条目；在第2期或第3期的任何条目被提交之前，该服务器再次崩溃，并持续了几个任期。



**为了使追随者的日志与自己的日志保持一致，领导者必须找到两个日志一致的最新日志条目，删除追随者日志中该点之后的任何条目**，并将该点之后领导者的所有条目发送给追随者。所有这些动作都是为了响应AppendEntries RPC所进行的一致性检查而发生的。领导为每个跟随者维护一个nextIndex，它是领导将发送给该跟随者的下一个日志条目的索引。当领导者第一次上台时，它将所有的nextIndex值初始化为其日志中最后一条的索引（图7中的11）。**如果跟随者的日志与领导者的日志不一致，在下一个AppendEntries RPC中，AppendEntries一致性检查将失败。在拒绝之后，领导者会递减nextIndex并重试AppendEntries RPC。最终，nextIndex将达到一个领导者和追随者日志匹配的点。**当这种情况发生时，AppendEntries就会成功，它将删除跟随者日志中任何冲突的条目，并追加领导者日志中的条目（如果有的话）。一旦AppendEntries成功，追随者的日志就与领导者的日志一致了，并且在剩下的任期里，它将保持这种状态。



如果需要，该协议可以被优化以减少被拒绝的AppendEntries RPC的数量。例如，**当拒绝一个AppendEntries请求时，跟随者可以包括冲突条目的任期和它为该任期存储的第一个索引。**有了这些信息，领导者可以递减nextIndex以绕过该任期中的所有冲突条目；每个有冲突条目的任期将需要一个AppendEntries RPC，而不是每个条目一个RPC。在实践中，我们怀疑这种优化是否有必要，因为故障不常发生，而且不太可能有很多不一致的条目。



有了这种机制，领导者在掌权时不需要采取任何特别的行动来恢复日志的一致性。它只是开始正常的操作，而日志会自动收敛以应对Append Entries一致性检查的失败。一个领导者从不覆盖或删除自己日志中的条目（图3中的 "Leader Append-Only"属性）。



这种日志复制机制表现出第2节中所描述的理想的共识属性：只要大多数服务器是正常的，Raft就可以接受、复制和应用新的日志条目；在正常情况下，一个新的条目可以通过单轮RPC复制到集群的大多数；而且一个缓慢的跟随者不会影响性能。



#### 5.4 Safety 



前面的章节描述了Raft如何选举领导和复制日志条目。然而，到目前为止所描述的机制还不足以确保每个状态机以相同的顺序执行完全相同的命令。例如，当领导者提交几个日志条目时，一个跟随者可能无法使用，然后它可能被选为领导者，并用新的条目覆盖这些条目；结果，不同的状态机可能执行不同的命令序列。



本节对Raft算法进行了完善，增加了对哪些服务器可以被选为领导者的限制条件。该限制确保了任何给定任期的领导者都包含了之前任期中承诺的所有条目（图3中的Leader Completeness属性）。考虑到选举限制，我们会使承诺的规则更加精确。最后，我们提出一个Leader Completeness属性的证明草图，并说明它如何导致副本状态机的正确行为。





##### 5.4.1 Election restriction 



在任何基于领导者的共识算法中，领导者最终必须存储所有承诺的日志条目。在一些共识算法中，例如Viewstamped Replication[22]，即使最初不包含所有已承诺的条目，也可以选出一个领导者。这些算法包含额外的机制来识别缺失的条目，并在选举过程中或之后不久将它们传送给新的领导者。不幸的是，这导致了相当多的额外机制和复杂性。Raft使用了一种更简单的方法，它**保证从每一个新的领导者当选的那一刻起，以前的所有承诺条目都存在于其身上，而不需要将这些条目传输给领导者。这意味着日志条目只在一个方向流动，即从领导者到追随者，而且领导者永远不会覆盖他们日志中的现有条目。**



Raft使用投票程序来防止候选人赢得选举，除非其日志包含所有承诺的条目。候选人必须与集群中的大多数人联系才能当选，这意味着每个已承诺的条目必须至少存在于其中一个服务器中。如果候选人的日志至少和该多数人中的任何其他日志一样是最新的（这里的 "最新 "在下面有精确的定义），那么它将包含所有承诺的条目。RequestVote RPC实现了这一限制：RPC包括关于候选人日志的信息，如果投票人自己的日志比候选人的日志更及时，则拒绝投票。



**Raft通过比较日志中最后条目的索引和任期来确定两个日志中哪一个是最新的。如果日志的最后条目有不同的任期，那么任期较晚的日志是最新的。如果日志以相同的任期结束，那么哪一个日志的索引更长，哪一个就更新。**



##### 5.4.2 Committing entries from previous terms 

<img src="Mit 6.824/image-20230124204906553.png" alt="image-20230124204906553" style="zoom:67%;" />



图8：一个时间序列显示了为什么领导者不能使用较早的任期的日志条目来确定承诺。在（a）中，S1是领导者，部分复制了索引2的日志条目。在(b)中，S1崩溃了；S5以S3、S4和它自己的投票当选为第三任期的领导者，并接受了日志索引2的不同条目。在（c）中，S5崩溃了；S1重新启动，被选为领导者，并继续复制。在这一点上，第2项的日志条目已经在大多数服务器上复制，但它没有被提交。如果S1像(d)那样崩溃，S5可以被选为领导者（有S2、S3和S4的投票），并用它自己的第3任期的条目覆盖该条目。然而，如果S1在崩溃前在大多数服务器上复制了其当前任期的条目，如(e)，那么这个条目就被承诺了（S5不能赢得选举）。在这一点上，日志中所有前面的条目也被提交。

![图片](https://mmbiz.qpic.cn/mmbiz_png/GafMH4lpVp7fa986NDY0Qw6TgX5WtHlfmCDwg4W7U0jfQtsGOUvUBYyEXaCkxUibMNDOjFN0UkvmnzpFJVaCesA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图9：如果S1（任期T的领导者）在其任期内提交了一个新的日志条目，而S5被选为后来任期U的领导者，那么至少有一个服务器（S3）接受了该日志条目，并且也为S5投票。



如第5.3节所述，一旦一个条目被存储在大多数服务器上，领导者就知道其当前任期的条目被提交。如果一个领导者在提交条目之前崩溃了，未来的领导者将试图完成对该条目的复制。然而，领导者不能立即得出结论，一旦前一届的条目被存储在大多数服务器上，它就被提交了。图8说明了这样一种情况：一个旧的日志条目被存储在大多数服务器上，但仍然可以被未来的领导者所覆盖。



为了消除类似图8中的问题，**Raft从不通过计算副本数量来提交以前的日志条目。只有领导者当前任期的日志条目是通过计算副本数量来提交的**；一旦当前任期的条目以这种方式被提交，那么由于日志匹配特性，所有之前的条目都被间接提交。在某些情况下，领导者可以安全地断定一个较早的日志条目已被提交（例如，如果该条目存储在每个服务器上），但Raft为了简单起见，采取了更保守的方法。



Raft在承诺规则中产生了这种额外的复杂性，因为当领导者复制以前任期的条目时，日志条目会保留其原始任期编号。在其他共识算法中，如果一个新的领导者重新复制之前 "任期"中的条目，它必须用新的 "任期号 "来做。Raft的方法使得对日志条目的推理更加容易，因为它们在不同的时间和不同的日志中保持着相同的任期编号。此外，与其他算法相比，Raft的新领导从以前的任期中发送较少的日志条目（其他算法必须发送多余的日志条目，在它们被提交之前对其重新编号）。



##### 5.4.3 Safety argument 



基于完整的Raft算法，我们现在可以更精确地论证领导者完备性属性是否成立（这个论证是基于安全证明的，见第9.2节）。我们假设领导者完备性属性不成立，然后我们证明一个矛盾。假设任期T的领导者（leaderT）从其任期中提交了一个日志条目，但是这个日志条目没有被未来某个任期的领导者所存储。考虑最小的任期U > T，其领导者（leaderU）不存储该条目。



\1. The committed entry must have been absent from leaderU’s log at the time of its election (leaders never delete or overwrite entries). 

承诺的条目在leaderU当选时必须它不在的日志中（leader从不删除或改写条目）。



\2. leaderT replicated the entry on a majority of the cluster, and leaderU received votes from a majority of the cluster. Thus, at least one server (“the voter”) both accepted the entry from leaderT and voted for leaderU, as shown in Figure 9. The voter is key to reaching a contradiction. 

leaderT在集群的大多数服务器上复制了该条目，而领导者U从集群的大多数服务器上获得了投票。因此，至少有一台服务器（"投票者"）既接受了leaderT的条目，又投票给leaderU，如图9所示。这个投票者是达成矛盾的关键。



\3. The voter must have accepted the committed entry from leaderT *before* voting for leaderU; otherwise it would have rejected the AppendEntries request from leaderT (its current term would have been higher than T). 

投票者在投票给leaderU之前，必须接受leaderT的承诺条目；否则它就会拒绝leaderT的AppendEntries请求（它的当前任期会比T高）。



\4. The voter still stored the entry when it voted for leaderU, since every intervening leader contained the entry (by assumption), leaders never remove entries, and followers only remove entries if they conflict with the leader. 

投票者在投票给leaderU时仍然储存了该条目，因为每个介入的领导者都包含该条目（根据假设），领导者从不删除条目，而追随者只有在与领导者冲突时才会删除条目。



\5. The voter granted its vote to leaderU, so leaderU’s log must have been as up-to-date as the voter’s. This leads to one of two contradictions. 

投票者将自己的选票投给leaderU，所以leaderU的日志肯定和选民的一样是最新的。这导致了两个矛盾中的一个。



\6. First, if the voter and leaderU shared the same last log term, then leaderU’s log must have been at least as long as the voter’s, so its log contained every entry in the voter’s log. This is a contradiction, since the voter contained the committed entry and leaderU was assumed not to. 

首先，如果投票者和leaderU共享最后一个日志项，那么leaderU的日志至少要和投票者的一样长，所以它的日志包含了投票者日志中的每一个条目。这是一个矛盾，因为投票者包含了已承诺的条目，而leaderU被认为不包含。



\7. Otherwise, leaderU’s last log term must have been larger than the voter’s. Moreover, it was larger than T, since the voter’s last log term was at least T (it contains the committed entry from term T). The earlier leader that created leaderU’s last log entry must have contained the committed entry in its log (by assumption). Then, by the Log Matching Property, leaderU’s log must also contain the committed entry, which is a contradiction. 

否则，leaderU的最后一个日志的任期一定比投票者的大。而且，该任期比T大，因为投票人的最后一个日志任期至少是T（它包含T项的承诺条目）。创建leaderU的最后一个日志的早期leader，其日志中一定包含了已承诺的条目（根据假设）。那么，根据日志匹配属性，leaderU的日志也必须包含已承诺的条目，这是个矛盾。



\8. This completes the contradiction. Thus, the leaders of all terms greater than T must contain all entries from term T that are committed in term T. 

这就完成了矛盾。因此，所有大于T的任期的领导必须包含任期T的所有条目，这些条目在任期T中被承诺。



\9. The Log Matching Property guarantees that future leaders will also contain entries that are committed indirectly, such as index 2 in Figure 8(d). 

日志匹配属性保证了未来的领导也将包含间接承诺的条目，如图8(d)中的索引2。



基于领导者完备性属性，我们可以证明图3中的状态机安全属性，即如果一个服务器在其状态机上应用了一个给定索引的日志条目，那么没有其他服务器会在同一索引上应用一个不同的日志条目。当一个服务器在其状态机上应用一个日志条目时，直到该条目的日志必须与领导者的日志相同，并且该条目必须被提交。现在考虑任何服务器应用一个给定的日志索引的最低任期；日志完整性属性保证所有更高任期的领导者将存储相同的日志条目，因此在以后的任期中应用索引的服务器将应用相同的值。因此，状态机安全属性成立。



  最后，Raft要求服务器按照日志索引顺序应用条目。结合状态机安全属性，这意味着所有服务器将以相同的顺序将相同的日志条目集应用于其状态机。



#### 发送Append



```go

//如果follower没有响应，不断重发。如果follower拒绝，根据reply.Upnextindex恢复follower缺少的日志
//第一条有效日志：term=0
func (rf *Raft) sendAppendEntry(server int, args *AppendEntriesArgs, reply *AppendEntriesReply,num* int) bool {


	ok := rf.peers[server].Call("Raft.RequestApp", args, reply)
	if ok==false||rf.killed() {
		return false
	}

	//对于leader而言，如果reply的term》leader，说明leader已经过期了
	rf.mu.Lock()
	defer rf.mu.Unlock()

	if reply.Term>rf.term{
		rf.statue=follower
		rf.Updateterm(reply.Term)
		rf.Log("Higher append reply , become follower")
		return reply.Success
	}
	if rf.statue!=leader{
		return  false
	}
	if reply.Success{  //如果时reply success，那么所有节点的日志都应该是一样的，因为leader会将冲突到最新的节点一次性发送
		//这里有bug，如果是keeepalive，那么args.Entries=0,所以应该是+而不是=
		rf.matchIndex[server]=args.Previogindex+len(args.Entries)  //更新节点的匹配下标，可能会一次提交多条日志
		rf.NextIndex[server] = rf.matchIndex[server] + 1
		rf.Log("peer %v accept append which waht %v Pre %v",server,rf.NextIndex[server],args.Previogindex)
		*num++
		if *num >= len(rf.peers)/2+1{
			*num=0  //保证不会提交两次！！
			if len(rf.Logs)==0{
				return true
			}
            
			if  rf.matchIndex[server]>rf.matchIndex[rf.me]{

				rf.matchIndex[rf.me]=rf.matchIndex[server]
				upidx:=rf.matchIndex[rf.me]

				if rf.Logs[upidx].Term==rf.term {
					rf.Commitindex=upidx
					rf.Log("Leader Commited %v",rf.Commitindex)
				}else {
					rf.Log("Leader Commited failde beacuse log.term %v are not now term %v",rf.Logs[upidx].Term,rf.term)
				}
			}
			//leader只能提交当前任期的日志！！
			//对于leader而言，只要通过半数就可提交了

		}
	}else {

		rf.NextIndex[server]=reply.Upnextindex
		rf.Log("peer %v refused ,which want %v",server,reply.Upnextindex)
	}

	return reply.Success
}

```



#### 接收Append

有这些动作都是为了响应AppendEntries RPC所进行的一致性检查而发生的。领导为每个跟随者维护一个nextIndex，它是领导将发送给该跟随者的下一个日志条目的索引。当领导者第一次上台时，它将所有的nextIndex值初始化为其日志中最后一条的索引（图7中的11）。**如果跟随者的日志与领导者的日志不一致，在下一个AppendEntries RPC中，AppendEntries一致性检查将失败。在拒绝之后，领导者会递减nextIndex并重试AppendEntries RPC。最终，nextIndex将达到一个领导者和追随者日志匹配的点。**当这种情况发生时，AppendEntries就会成功，它将删除跟随者日志中任何冲突的条目，并追加领导者日志中的条目（如果有的话）。一旦AppendEntries成功，追随者的日志就与领导者的日志一致了，并且在剩下的任期里，它将保持这种状态。

如果需要，该协议可以被优化以减少被拒绝的AppendEntries RPC的数量。例如，**当拒绝一个AppendEntries请求时，跟随者可以包括冲突条目的任期和它为该任期存储的第一个索引。**有了这些信息，领导者可以递减nextIndex以绕过该任期中的所有冲突条目；每个有冲突条目的任期将需要一个AppendEntries RPC，而不是每个条目一个RPC。



```go

//对append的回应：注意：不要将心跳和日志append分开处理！！！！
//
func (rf *Raft) RequestApp(args *AppendEntriesArgs, reply *AppendEntriesReply)  {

	//rf.Log("receive a keep-alive from leader %v which term %v",args.Leaderid,args.Term)


	rf.mu.Lock()
	defer rf.mu.Unlock()

	//是大于等于！！！！
	if args.Term>=rf.term{
		rf.Updateterm(args.Term)
		rf.statue=follower
		rf.Log("Higher append , become follower")
	}
	if rf.statue==leader{
		reply.Success=false
		reply.Term=rf.term
		return
	}
	if args.Term>=rf.term{
		//收到心跳包
		rf.Upelection()

		//如果这个日志在快照中，且term一致。接收日志。
		if (args.Previogindex>=rf.lastIncludeIndex&& args.Previogindex<=rf.getLastIndex()&&
			rf.restoreLogTerm(args.Previogindex)==args.Previogierm){

			// 如果存在日志包那么进行追加
			if args.Entries != nil {

				/*firstIndex := rf.lastIncludeIndex
				//找到第一个冲突日志
				tmpidx :=args.Previogindex+1
				for index,e := range args.Entries{
					if rf.restoreLogTerm(tmpidx)!=e.Term {
						rf.Logs = append(rf.Logs[:args.Previogindex+1-rf.lastIncludeIndex], args.Entries...)
					}
				}
				for index, entry := range args.Entries {
					if tmpidx-firstIndex >= len(rf.Logs) || rf.Logs[entry.Index-firstIndex].Term != entry.Term {
						rf.Logs = shrinkEntriesArray(append(rf.logs[:entry.Index-firstIndex], request.Entries[index:]...))
						break
					}
				}*/
				//+1是因为切片末尾-1
				rf.Logs = append(rf.Logs[:args.Previogindex+1-rf.lastIncludeIndex], args.Entries...)
			}

			rf.Commitindex=Min(rf.getLastIndex(),args.Ladercommit)

			rf.Log("accept the log now lastlog index %v, peer have commit %v",rf.getLastIndex(),rf.Commitindex)
			reply.Success=true
		} else {

            
			conflicidx:=0
			if args.Previogindex>rf.getLastIndex(){
				conflicidx=rf.getLastIndex()+1
			}else if args.Previogindex<rf.lastIncludeIndex {
				conflicidx=rf.lastIncludeIndex+1
			} else{
				tempTerm := rf.restoreLogTerm(args.Previogindex)
				//我在这任期出问题了，返回这个任期内的第一个索引
				for index := args.Previogindex; index > rf.lastIncludeIndex; index-- {
					if rf.restoreLogTerm(index) != tempTerm {
						conflicidx= index+1
						break
					}
				}
			}

			reply.Upnextindex=Max(conflicidx,rf.lastIncludeIndex+1)  //返回的应该是commited了
			reply.Success=false
			rf.Log("log append false, I want %v but accept a log after Previogindex %v",reply.Upnextindex,args.Previogindex)
		}
	}else{
		//如果收到term比我低的心跳包，要重置时间吗？应该不用吧。。。
		rf.Log("accept a lower keep-alive! from %v",args.Leaderid)
		reply.Success=false
	}
	reply.Term=rf.term
}
```



```bash
./loop_to_test_raft.sh 2A 100
```





#### Bug：

![image-20221011001953637](Mit 6.824/image-20221011001953637.png)

第一条日志发送，并且被多数节点复制，但还没提交，第二次keepalive时，日志因没有通过一致性检测被拒收

**第二条日志的preidx和preterm的下标是0，条件没有判断到**

![image-20221011100116132](Mit 6.824/image-20221011100116132.png)

```go
if args.Previogindex>0&& args.Previogindex<=len(rf.Logs)&&rf.Logs[args.Previogindex-1].Term==args.Previogierm{
```



**数据征用**

```go
==================
WARNING: DATA RACE
Write at 0x00c0000a6be8 by goroutine 7:
  6.824/raft.(*Raft).Start()
      /home/blacksheep/Dcs_/6.824/src/raft/raft.go:598 +0x284
  6.824/raft.(*config).one()
      /home/blacksheep/Dcs_/6.824/src/raft/config.go:578 +0x3aa
  6.824/raft.TestBasicAgree2B()
      /home/blacksheep/Dcs_/6.824/src/raft/test_test.go:140 +0x144
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1439 +0x213
  testing.(*T).Run.func1()
      /usr/local/go/src/testing/testing.go:1486 +0x47

Previous read at 0x00c0000a6be8 by goroutine 27:
  
6.824/raft.(*Raft).sendAppendEntry()
      /home/blacksheep/Dcs_/6.824/src/raft/raft.go:419 +0x84
  6.824/raft.(*Raft).Keepalive.func1()
      /home/blacksheep/Dcs_/6.824/src/raft/raft.go:409 +0x71

Goroutine 7 (running) created at:
  testing.(*T).Run()
      /usr/local/go/src/testing/testing.go:1486 +0x724
  testing.runTests.func1()
      /usr/local/go/src/testing/testing.go:1839 +0x99
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1439 +0x213
  testing.runTests()
      /usr/local/go/src/testing/testing.go:1837 +0x7e4
  testing.(*M).Run()
      /usr/local/go/src/testing/testing.go:1719 +0xa71
  main.main()
      _testmain.go:95 +0x2e4

Goroutine 27 (running) created at:
  6.824/raft.(*Raft).Keepalive()
      /home/blacksheep/Dcs_/6.824/src/raft/raft.go:409 +0x336
  6.824/raft.(*Raft).sendRequestVote()
      /home/blacksheep/Dcs_/6.824/src/raft/raft.go:382 +0x5e6
  6.824/raft.(*Raft).Leaderelection.func1()

```



![image-20221013232118732](Mit 6.824/image-20221013232118732.png)

**follower commit后退，leader 在keepalive时数组分片超界**

1.follower commit后退：

- leader的commit没变，引起followercommit变换应该是接受日志时，Preidx减少

2.数组超界：

- 当不正确时，request的nextidx=commit+1，所以当follower和leader日志一致，nextidx=commit+1导致数组超界。写程序时应该正确和错误一起思考



```go
			if len(rf.Logs)!=0 {
				if rf.NextIndex[i]==0 {
					args.Entries=rf.Logs[0:]
				}else {
					args.Entries=rf.Logs[rf.NextIndex[i]:]
				}
			}
```



![image-20221014234743323](Mit 6.824/image-20221014234743323.png)

**分析问题：**

- 收到了投票请求，并且没有拒绝，但是candidate却没有收到回复。
  - 判断日志一致性时少了一个条件。。。



**go不能用+=，得用++？？？？**



**疯狂DDOS，由于之前断连时leader不断轮询发送keepalive，所以导致一档重启连接，节点会被之前过时的keepalive淹没，造成饥饿，根本没法进行正常程序！如果是几百台的大型集群，不断轮询是没有问题的，因为断连是小概率事件，只有小部分节点饥饿，对整个集群运行没有问题，但在本地小规模集群，如果不断轮询，会造成集群中大部分的节点饥饿**





- 当集群只有一条日志，但leader还未提交时宕机。此时节点的comit=0，拒绝append时返回1，而leader再keepalive时。preidx=0， 会无限被拒收！！



![image-20221017230043150](Mit 6.824/image-20221017230043150.png)





![image-20221017230028765](Mit 6.824/image-20221017230028765.png)



**解决方案:**

- 当commit=0时，也视为logs=0

<img src="Mit 6.824/image-20221017231351051.png" alt="image-20221017231351051" style="zoom:80%;" />

### C：持久化

![在这里插入图片描述](Mit 6.824/46075a187c144b44af8a572a6f4537b0.png)



currentTerm、voteFor、log[]三个state。因此只要当raft节点的这三个状态发生改变时，就将他们的状态存储到磁盘上。

用gob的encode库来序列和反序列化文件。

```go

func (rf *Raft) persist() {
	// Your code here (2C).
	// Example:
	// w := new(bytes.Buffer)
	// e := labgob.NewEncoder(w)
	// e.Encode(rf.xxx)
	// e.Encode(rf.yyy)
	// data := w.Bytes()
	// rf.persister.SaveRaftState(data)
	rf.persister.SaveRaftState(rf.persistdate())
}
func (rf *Raft) persistdate() []byte{
	w := new(bytes.Buffer)
	e := labgob.NewEncoder(w)
    //序列化
	e.Encode(rf.term)
	e.Encode(rf.Logs)
	e.Encode(rf.lastIncludeIndex)
	e.Encode(rf.lastIncludeTerm)
	e.Encode(rf.Commitindex)
	e.Encode(rf.LastApplied)
	data := w.Bytes()
	//rf.persister.SaveRaftState(data)
	return data
}
```



#### figure 8

<img src="Mit 6.824/image-20230125161631527.png" alt="image-20230125161631527" style="zoom:67%;" />

图8：一个时间序列显示了为什么领导者不能使用较早的任期的日志条目来确定承诺,**只能通过提交自己任期的日志，从而间接提交之前任期的日志。**。在（a）中，S1是领导者，部分复制了索引2的日志条目。在(b)中，S1崩溃了；S5以S3、S4和它自己的投票当选为第三任期的领导者，并接受了日志索引2的不同条目。在（c）中，S5崩溃了；S1重新启动，被选为领导者，并继续复制。在这一点上，第2项的日志条目已经在大多数服务器上复制，但它没有被提交。如果S1像(d)那样崩溃，S5可以被选为领导者（有S2、S3和S4的投票），并用它自己的第3任期的条目覆盖该条目。然而，如果S1在崩溃前在大多数服务器上复制了其当前任期的条目，如(e)，那么这个条目就被承诺了（S5不能赢得选举）。在这一点上，日志中所有前面的条目也被提交。

**这个问题的原因在于raft的日志在commit之前是可以回滚（回退的）。如何一个节点，包括leader，必须确保某一条日志不会被回滚才能commit它。自己任期的日志一旦复制到了quorum，肯定不会回滚，因为选举流程可以保证这一点。**

而前任的日志，即使已经复制到quorum，也有可能被回滚，因为没有别的机制来保证不会回滚。所以，不能直接commit前任的日志。

**引入 no-op 日志**

no-op 日志即只有 index 和 term 信息，command 信息为空。也是要写到磁盘存储的。

具体流程是在 Leader 刚选举成功的时候，立即追加一条 no-op 日志，并立即复制到其它节点，no-op 日志一经提交，Leader 前面那些未提交的日志全部间接提交，问题就解决了。像上面的 kv 数据库，有了 no-op 日志之后，Leader 就能快速响应客户端查询了。

本质上，no-op 日志使 Leader 隐式地快速提交之前任期未提交的日志，确认当前 `commitIndex`，这样系统才会快速对外正常工作。

```go
//no-op 当上leader后立即提交，而不用等到下次再来,并快速定位各个节点的状态机情况
func (rf* Raft) quaickcommit()  {
	rf.Start(Entry{})
	rf.Keepalive()
}
```



<img src="Mit 6.824/image-20230125190107583.png" alt="image-20230125190107583" style="zoom:67%;" />



#### bug



### D：压缩

就目前情况而言，重启服务器会重播完整的Raft日志，以恢复其状态。然而，对于长期运行的服务来说，永远记住完整的Raft日志是不现实的。相反，您将修改Raft，使其与持续存储其状态“快照”的服务协作，此时Raft将丢弃快照之前的日志条目。结果是持久数据量更小，重启速度更快。然而，现在追随者可能落后太多，以至于领导者已经放弃了需要追赶的日志条目；然后，领导者必须发送快照以及快照时开始的日志。扩展筏文件第7节概述了方案；你必须设计细节。

Raft还在快照中包含了少量的元数据：**最后包含的索引是快照所取代的日志中最后一个条目的索引（状态机应用的最后一个条目），最后包含的任期是这个条目的任期。这些被保留下来是为了支持快照之后的第一个日志条目的AppendEntries一致性检查，因为该条目需要一个先前的日志索引和任期。**为了实现集群成员的变化（第6节），快照还包括日志中的最新配置，即最后包含的索引。一旦服务器完成写入快照，它可以删除所有的日志条目，直到最后包含的索引，以及任何先前的快照。

<img src="Mit 6.824/image-20221018004408498.png" alt="image-20221018004408498" style="zoom:80%;" />

图12：一个服务器用一个新的快照替换了其日志中的**已提交条目**（索引1到5），该快照只存储了当前状态（本例中的变量x和y）。快照最后包含的索引和任期用于定位快照在日志第6条之前的位置。



Raft的日志在正常运行期间不断增长，以纳入更多的客户端请求，但在实际系统中，它不可能无限制地增长。随着日志的增长，它占据了更多的空间，需要更多的时间来重放。如果没有某种机制来丢弃积累在日志中的过时信息，这最终会导致可用性问题。



快照是最简单的压缩方法。在快照中，整个当前系统状态被写入稳定存储的快照中，然后丢弃截至该点的整个日志。快照在Chubby和ZooKeeper中使用，本节的其余部分将介绍Raft中的快照。



递增的压缩方法，如日志清理[36]和日志结构的合并树[30，5]，也是可能的。这些方法一次性对部分数据进行操作，因此它们将压缩的负荷更均匀地分散在时间上。它们首先选择一个积累了许多被删除和被覆盖的对象的数据区域，然后将该区域的活跃对象重写得更紧凑，并释放该区域。与快照处理相比，这需要大量的额外机制和复杂性，因为快照处理总是对整个数据集进行操作，从而简化了问题。虽然日志清理需要对Raft进行修改，但状态机可以使用与快照相同的接口实现LSM树。



图12显示了Raft中快照的基本思路。**每台服务器独立进行快照，只覆盖其日志中已提交的条目。**大部分的工作包括状态机将其当前状态写入快照。**Raft还在快照中包含了少量的元数据：最后包含的索引是快照所取代的日志中最后一个条目的索引（状态机应用的最后一个条目），最后包含的任期是这个条目的任期。**这些被保留下来是为了支持快照之后的第一个日志条目的AppendEntries一致性检查，因为该条目需要一个先前的日志索引和任期。为了实现集群成员的变化（第6节），快照还包括日志中的最新配置，即最后包含的索引。一旦服务器完成写入快照，它可以删除所有的日志条目，直到最后包含的索引，以及任何先前的快照。



尽管服务器通常是独立进行快照，但领导者偶尔必须向落后的跟随者发送快照。这种情况发生在领导者已经丢弃了它需要发送给跟随者的下一个日志条目。幸运的是，这种情况在正常操作中不太可能发生：一个跟上领导者的追随者已经有了这个条目。然而，一个特别慢的跟随者或一个新加入集群的服务器（第6节）就不会有这样的情况。让这样的追随者跟上的方法是，领导者通过网络向它发送一个快照。



领导者使用一个叫做InstallSnapshot的新RPC来向落后于它的跟随者发送快照；见图13。当跟随者收到这个RPC的快照时，它必须决定如何处理其现有的日志条目。通常情况下，快照将包含新的信息，而不是在接收者的日志中。在这种情况下，跟随者会丢弃它的整个日志；它都被快照取代了，而且可能有与快照冲突的未提交的条目。相反，如果跟随者收到描述其日志前缀的快照（由于重传或错误），那么被快照覆盖的日志条目将被删除，但快照之后的条目仍然有效，必须保留。



这种快照方法偏离了Raft的强领导原则，因为跟随者可以在领导不知情的情况下进行快照。然而，我们认为这种背离是合理的。虽然有一个领导者有助于在达成共识时避免冲突的决定，但在快照时已经达成了共识，所以没有决定冲突。数据仍然只从领导者流向追随者，只是追随者现在可以重新组织他们的数据。



我们考虑了另一种基于领导者的方法，即只有领导者会创建一个快照，然后它将这个快照发送给其每个追随者。然而，这有两个缺点。首先，向每个追随者发送快照会浪费网络带宽，并减缓快照过程。每个追随者都已经拥有产生其自身快照所需的信息，而且对于服务器来说，从其本地状态产生快照通常比通过网络发送和接收快照要便宜得多。第二，领导者的实现将更加复杂。例如，领导者需要在向追随者复制新的日志条目的同时向他们发送快照，以便不阻碍新的客户请求。



还有两个影响快照性能的问题。首先，服务器必须决定何时进行快照。如果服务器快照的频率过高，就会浪费磁盘带宽和能源；如果快照的频率过低，就会有耗尽存储容量的风险，而且会增加重启时重放日志的时间。一个简单的策略是，当日志达到一个固定的字节大小时进行快照。如果这个大小被设定为明显大于快照的预期大小，那么快照的磁盘带宽开销就会很小。



第二个性能问题是，写入快照可能需要大量的时间，我们不希望因此而耽误正常的操作。解决方案是使用写时复制技术，这样就可以接受新的更新而不影响正在写入的快照。例如，用功能数据结构构建的状态机自然支持这一点。另外，操作系统的写时拷贝支持（例如Linux上的fork）可以用来创建整个状态机的内存快照（我们的实现采用了这种方法）。





回到raft的日志增量中，其实我们可以发现，commit更新的流程其实是，Leader发送给各个节点进行同步日志，然后返回给leader同步RPC的结果，更新matchIndex。如果超过半数节点已经同步成功后的日志，那么leader会把超过半数，且最新的matchIndex设为commitIndex,然后再由提交ticker进行提交。然后在下一次发送日志心跳时再更新followers的commitIndex下标。

也因此就会可能有半数的节点，又或是网络分区，crash的节点没有更新到已提交的节点，而这段已提交的日志已经被**leader提交而抛弃了**。那么这个时候就需要leader发送自身的快照，安装给这些followers。

可以看出Snapshot会有两个问题：

- 快照设置频繁下带来的浪费磁盘带宽和能源问题。当然快照设置不过与频繁也会导致存储问题。因此合理的安装快照才是正确的解决办法。 Paper中提到的策略就是对Log设置一个合理的大小，超过这个大小就进行日志快照的更新。
- 另外的就是在写入快照时进行的时间开销。因为可能在写入快照时会有竞争，并加锁。paper中提到的也是经典copy-on-write方法，在更新快照时，先临时储存一份log的备份。

<img src="Mit 6.824/45450bd6100f4765b716ff8490839d46.png" alt="在这里插入图片描述" style="zoom:67%;" />



```go
type InstallSnapshotArgs struct {
	Term             int    // 发送请求方的任期
	LeaderId         int    // 请求方的LeaderId
	LastIncludeIndex int    // 快照最后applied的日志下标
	LastIncludeTerm  int    // 快照最后applied时的当前任期
	Data             []byte // 快照区块的原始字节流数据
	//Done bool
}
type Raft struct{
    lastIncludeIndex int  //快照包含的最后下标
	lastIncludeTerm  int  //快照包含的最后任期
}
//几个下标的关系
curindex:从0开始单增,在用户看来代表正确的log下标
lastidx:快照包含的最大下标（已经压入快照的下标）
```

<img src="Mit 6.824/image-20230125201610240.png" alt="image-20230125201610240" style="zoom:67%;" />



#### 更新快照

该服务表示，它已经创建了一个快照，其中包含所有信息，包括索引。这意味着服务不再需要通过（包括）该索引进行日志记录。raft现在应该尽可能地修剪快照。快照只能包含已提交的内容。

```go
//follower主动更新快照。snapshot:传来的快照日志，index：快照日志包含的最后的日志下标。
func (rf *Raft) Snapshot(index int, snapshot []byte) {
	// Your code here (2D).
	if rf.killed() {
		return
	}

	rf.mu.Lock()
	defer rf.mu.Unlock()
	// 如果下标大于自身的提交，说明没被提交不能安装快照，如果自身快照点大于index说明不需要安装
	if rf.lastIncludeIndex >= index || index > rf.Commitindex {
		return
	}
	// 更新日志，只保留index之后的日志。
	sLogs := make([]Entry, 0)
	sLogs = append(sLogs, Entry{})
	for i := index + 1; i <= len(rf.Logs); i++ {
		sLogs = append(sLogs, rf.restoreLog(i))
	}

	// 更新快照任期
	if index == len(rf.Logs)+1 {
		rf.lastIncludeTerm = rf.Logs[len(rf.Logs)-1].Term
	} else {
		rf.lastIncludeTerm = rf.restoreLogTerm(index)
	}

	rf.lastIncludeIndex = index
	rf.Logs = sLogs

	// apply了快照就应该重置commitIndex、lastApplied
	if index > rf.Commitindex {
		rf.Commitindex = index
	}
	if index > rf.LastApplied {
		rf.LastApplied = index
	}

	// 持久化快照信息
	rf.persister.SaveStateAndSnapshot(rf.persistdate(), snapshot)

}
```

**快照下标和原本下标**

**因为存储了快照之后，会将lastindex之前的日志都抛弃（存到磁盘上）**

```go
快照包含的最后的日志的任期和下标
rf.lastIncludeIndex
rf.lastIncludeIndex
当前index在log中的真实位置其实就是cur-last
当index等于lastindex时，就不能再从log中找数据了，因为log直接空了，所以要保存lastindex和lastterm
```



```c++

func (rf* Raft) restoreindex(curIndex int) int {
	return curIndex-rf.lastIncludeIndex
}
// 通过快照偏移还原真实日志条目
func (rf *Raft) restoreLog(curIndex int) Entry {
	return rf.Logs[curIndex-rf.lastIncludeIndex]
}

// 通过快照偏移还原真实日志任期
func (rf *Raft) restoreLogTerm(curIndex int) int {
	// 如果当前index与快照一致/日志为空，直接返回快照/快照初始化信息，否则根据快照计算
	if curIndex==rf.lastIncludeIndex {
		return rf.lastIncludeTerm
	}
	//fmt.Printf("[GET] curIndex:%v,rf.lastIncludeIndex:%v\n", curIndex, rf.lastIncludeIndex)
	return rf.Logs[curIndex-rf.lastIncludeIndex].Term
}

// 获取最后的日志下标
func (rf *Raft) getLastIndex() int {
	return len(rf.Logs) -1 + rf.lastIncludeIndex
}

// 获取最后的任期
func (rf *Raft) getLastTerm() int {
	// 因为初始有填充一个，否则最直接len == 0
	if len(rf.Logs)-1 == 0 {
		return rf.lastIncludeTerm
	} else {
		return rf.Logs[len(rf.Logs)-1].Term
	}
}

// 通过快照偏移还原server的PrevLogInfo
func (rf *Raft) getPrevLogInfo(server int) (int, int) {
	newEntryBeginIndex := Max(rf.NextIndex[server] - 1,0)
    //如果下标超了，相等。
	lastIndex := rf.getLastIndex()
	if newEntryBeginIndex >= lastIndex+1 {
		newEntryBeginIndex = lastIndex
	}
	return newEntryBeginIndex, rf.restoreLogTerm(newEntryBeginIndex)
}

```

当要发送给follower的日志在快照中时，直接发送快照。

```go
//由领导者调用，向跟随者发送快照的分块。领导者总是按顺序发送分块。
func (rf *Raft) leaderSendSnapShot(server int) {

	rf.mu.Lock()

	args := InstallSnapshotArgs{
		rf.term,
		rf.me,
		rf.lastIncludeIndex,
		rf.lastIncludeTerm,
		rf.persister.ReadSnapshot(),
	}
	reply := InstallSnapshotReply{}

	rf.mu.Unlock()

	ok := rf.peers[server].Call("Raft.InstallSnapShot", args, reply)

	if ok == true {
		rf.mu.Lock()
		if rf.statue != leader || rf.term != args.Term {
			rf.mu.Unlock()
			return
		}


		if reply.Term > rf.term {
			rf.statue = follower
			rf.Updateterm(reply.Term)
			//rf.persist()
			rf.Upelection()
			rf.mu.Unlock()
			return
		}
		
		rf.matchIndex[server] = args.LastIncludeIndex
		rf.NextIndex[server] = args.LastIncludeIndex + 1

		rf.mu.Unlock()
		return
	}
}
```



## lab3：基于raft的kv应用层

使用[实验室 2](https://pdos.csail.mit.edu/6.824/labs/lab-raft.html) 中的 Raft 库构建容错键/值存储服务。您的键/值服务将是一个复制的状态机，由多个使用 Raft 进行复制的键/值服务器组成。只要大多数服务器处于活动状态并且可以进行通信，您的键/值服务就应继续处理客户端请求，即使出现其他故障或网络分区也是如此

客户端可以将三个不同的 RPC 发送到键/值服务：Put(key, value)`, `Append(key, arg)， `Get(key)`。该服务维护一个简单的键/值对数据库。键和值是字符串。`Put（键，值）`替换数据库中特定键的值，`append（键，arg）`将参数附加到键的值，`Get（键）`获取键的当前值。不存在的键的 `Get` 应返回空字符串。`附加到`不存在的键应类似于 `Put`。每个客户都通过一名 Clerk 使用Put/append/get 方法与服务人员交谈。Clerk  管理与服务器的 RPC 交互。



**Client**

- 每个client有一个独一的ID（用snow算法生成），每个任务也有一个taskId，服务器通过clientId和taskId来避免任务重复执行





**Server**



<img src="Mit 6.824/image-20221028223837249.png" alt="image-20221028223837249" style="zoom:80%;" />



**日志格式**



```go
//
//日志格式,server传给raft的序列化格式
type Op struct {
	// Your definitions here.
	// Field names must start with capital letters,
	// otherwise RPC will break.
	Key      string
	Value    string
    
	Commid    int
	ClientId int64
    
	Index    int   // raft服务层传来的Index
	OpType   string  //操作类型
}

//server端保存client最后的操作
type ClientOperation struct {
	SequenceNum int
	value string
	Err Err
}
//client端请求的结果
type ChanResult struct {
	value string
	Err Err
}
```

**问题：server通过 单增的commandid和clientid来确定一个日志，那么commadnid会不会乱序呢？？？**

- 对于client来说，它的提交的commanid一定是单增的。

**如何且何时把raft的日志转换到数据库中**

- raft将日志提交到appych时，server端读相应日志，并将其转换成数据库中的内容

<img src="Mit 6.824/image-20221106225644265.png" alt="image-20221106225644265" style="zoom:50%;" />



## La4：终章-高可用的分布式系统



- 使用两阶段提交将跨分片原子事务添加到实验室 4 和/或快照。
- 构建具有异步复制的系统 (like Dynamo or Ficus or Bayou). 也许增加更强的一致性 (as in COPS or Walter or Lynx).
- 使用现代高速 NIC 功能（例如 RDMA 或 DPDK）构建 高速服务，可能具有复制或事务。
- 使用现代快速非易失性存储（例如英特尔傲腾）来 简化容错系统的设计。



lab4 的内容是要在 lab2 的基础上实现一个 multi-raft 的 KV 存储服务，同时也要支持切片在不同 raft 组上的动态迁移而不违背线性一致性，不过其不需要实现集群的动态伸缩。总体来看，lab4 是一个相对贴近生产场景的 lab。

构建一个密钥/值存储系统，它将密钥“分片”或分区到一组副本组上。分片是键/值对的子集；例如，所有以“a”开头的键可能都是一个分片，而所有以“b”开头的按键可能都是另一个，等等。分片的原因是性能。每个复制组只处理几个shard的put和get，并且组并行操作；因此，总系统吞吐量（每单位时间的投入和获得）与组的数量成比例地增加。

分片密钥/值存储将有两个主要组件。首先是一组副本组。每个副本组负责shard的子集。副本由几个服务器组成，这些服务器使用Raft来复制组的shard。第二个组件是“shard控制器”。shard控制器决定哪个副本组应该为每个shard服务；该信息称为配置。配置随时间变化。客户端咨询shard控制器以查找密钥的副本组，副本组咨询控制器以查找要服务的shard。整个系统只有一个shard控制器，使用Raft实现为容错服务。

分片存储系统必须能够在副本组之间切换分片。一个原因是，某些组可能比其他组负载更大，因此需要移动shard以平衡负载。另一个原因是副本组可能会加入和离开系统：可能会添加新的副本组以增加容量，或者可能会将现有副本组脱机以进行修复或退役。





![img](Mit 6.824/77bcfb5c280849f098f30c7da08eb852.png)



### Lab4A：分片

分片（Sharding）将一个数据分成两个或多个较小的块，称为逻辑分片（logical shards）。然后，逻辑分片（logical shards）分布在单独的数据库节点上，称为物理分片（physical shards）。物理分片（physical shards）可以容纳多个逻辑分片（logical shards）。尽管如此，所有分片中保存的数据，共同代表整个逻辑数据集。

数据库分片（Database shards）是无共享架构的一个例子。这意味着分片是自治的：分片间不共享任何相同的数据或服务器资源。但是在某些情况下，将某些表复制到每个分片中作为参考表是有意义的。例如，假设某个应用程序的数据库依赖于重量测量的固定转换率。通过将包含必要转换率数据的表复制到每个分片中，有助于确保查询所需的所有数据都保存在每个分片中。

**Benefits of Sharding 分片的好处**

数据库分片的主要吸引力在于，它可以帮助促进水平扩展（horizontal scaling），也称为向外扩展（scaling out）。水平扩展是将更多的机器添加到现有堆栈中，以分散负载，允许更多的流量和更快的处理。这通常与垂直扩展（vertical scaling）形成对比，垂直扩展也称为向上扩展（scaling up），是指升级现有服务器的硬件，通常是添加更多内存或CPU。

让一个关系数据库在单个机器上运行，并按需升级其服务器资源进行向上扩展是相对简单的。但最终，任何非分布式数据库在存储和计算能力方面都会受到限制，因此可以自由地水平扩展数据库，会使您的架构更加灵活且适应性强。

选择分片数据库架构的另一个原因，是为了加速查询响应的时间。当您对尚未分片的数据库提交查询时，必须先搜索您查询的表中的每一行，然后才能找到您要查找的结果集。对于具有大型单片数据库的应用程序，查询可能变得极其缓慢。但是，通过将一个表分成多个，查询过程会遍历更少的行，并且返回结果集的速度要快得多。

分片还可以通过减少宕机（outage）的影响，使应用程序更稳定可靠。如果您的应用程序或网站依赖于未分片的数据库，则宕机可能会导致整个应用程序不可用。但是，对于分片数据库，宕机可能只会影响单个分片。即使这可能使某些用户无法使用应用程序或网站部分功能，但仍会低于整个数据库崩溃带来的影响。

**Drawbacks of Sharding** **分片的缺点**

虽然对数据库进行分片可以使扩展更容易并提高性能，但它也可能会带来某些限制。在这里，我们将讨论其中的一些限制，以及为什么这些限制会让我们避免对数据库全部分片。

正确实现分片数据库架构，是十分复杂的，所以这是分片遇到的第一个困难。如果操作不正确，则分片过程可能会导致数据丢失或表损坏，这是一个很大的风险。但是，即使正确地进行了分片，也可能对团队的工作流程产生重大影响。与从单个入口点访问和管理数据不同，用户必须跨多个分片位置管理数据，这可能会让某些团队存在工作混乱。

在对数据库进行分片后，用户有时会遇到的一个问题是分片最终会变得不平衡。举例来说，假设您有一个数据库，其中有两个单独的分片，一个用于姓氏以字母A到M开头的客户，另一个用于名字以字母N到Z开头的客户。但是，您的应用程序为姓氏以字母G开头的人提供了过多的服务。因此，A-M分片逐渐累积的数据比N-Z分片要多，这会导致应用程序速度变慢，并对很大一部分用户造成影响。A-M分片已成为所谓的数据热点。在这种情况下，数据库分片的任何好处都被慢速和崩溃抵消了。数据库可能需要修复和重新分片，才能实现更均匀的数据分布。

另一个主要缺点是，一旦对数据库进行了分片，就很难将其恢复到未分片的架构。分片前数据库的备份数据，都无法与分片后写入的数据合并。因此，重建原始的非分片架构，需要将新的分区数据与旧备份合并，或者将分区的数据库转换回单个数据库，这两种方法都是昂贵且耗时的。

要考虑的最后一个缺点是，并不是每个数据库引擎本身都支持分片。例如，尽管可以手动分片PostgreSQL数据库，但PostgreSQL本身并不包括自动分片功能。有许多Postgres分支包括自动分片功能，但这些分支通常落后于最新的PostgreSQL版本，并且缺乏某些其他的功能特性。一些专业的数据库技术——如MySQL Cluster或某些数据库即服务产品（如MongoDB Atlas）确实包含自动分片功能，但这些数据库管理系统的普通版本却并不包含。因此，分片通常需要“自己动手”的方法。这意味着通常很难找到有关分片或故障排除技巧的文档。

现在我们已经介绍了一些分片的缺点和好处，我们将讨论一些分片数据库的不同架构。

一旦你决定对数据库进行分片，接下来你需要弄清楚的是如何进行分片。在运行查询或将传入的数据分发到分片表或数据库时，关键是要将其分配到正确的分片。否则，它可能导致数据丢失或查询速度缓慢。在本节中，我们将介绍一些常见的分片架构，每个架构使用稍微不同的流程来跨分片分发数据。

- 总结：
  - 好处：性能更好，容灾更强
  - 坏处：如何分片考验程序员的功底。当分片不均或某一分片中的某一点请求过热，会导致这一整个分片性能下降。分片不好再现。



**Key Based Sharding 基于键的分片**

哈希函数中输入的值都应该来自同一列。此列称为分片键。

<img src="Mit 6.824/image-20221108235140220.png" alt="image-20221108235140220" style="zoom:67%;" />

- 好处：均匀分片，数据散列效果好。无需再保存数据和其所在位置的映射
- 坏处：数据迁徙时棘手

**Range Based Sharding 基于范围的分片**

基于范围的分片（Range based sharding），基于给定值的范围进行数据分片。为了说明，假设您有一个数据库，用于存储零售商目录中所有产品的信息。您可以创建一些不同的分片，并根据每个产品的价格范围分配每个产品的信息，如下所示：

<img src="Mit 6.824/image-20221108235359509.png" alt="image-20221108235359509" style="zoom:67%;" />

**Directory Based Sharding 基于目录的分片**

要实现基于目录的分片，必须创建并维护一个查找表，该查找表使用分片键来跟踪哪个分片包含哪些数据。简而言之，查找表是一个表，其中包含有关可以找到特定数据的静态信息集。下图显示了基于目录的分片的简单示例：

<img src="Mit 6.824/image-20221108235442889.png" alt="image-20221108235442889" style="zoom:67%;" />

Delivery Zone列被定义为分片键。将来自分片键的数据，连同每一行应该写入的分片写入查找表。这与基于范围的分片类似，但不是确定分片键的数据落入哪个范围，而是将每个键绑定到其自己的特定分片。如果分片键的基数很低，并且分片键存储键的范围没有意义，那么基于目录的分片比基于范围的分片要更好。



**组**

一个分片交给一个组来处理，一个组可以处理多个分片！！

互不相交并且组成完整数据的每一个数据子集称为 Shard。（可以理解为一张表）在同一阶段中，Shard 与集群的对应关系称为 Config，随着时间的推移，增加或减少机器、某个 Shard 中的数据请求过热，Shard 需要在不同集群之中进行迁移。如何在 Config 更新、 Shard 迁移的同时仍能正确对外提供强一致性的服务，是lab4主要挑战。

**lab4A的工作并不是分片，而是管理各组raft集群里的分片**



![image-20221108234147191](Mit 6.824/image-20221108234147191.png)

配置，分片和分组间的关系。

shards的下标是分片，一个分片对应一个gid（分组）.多个下标（分片）的值（gid）可以一样

![image-20230127231126414](Mit 6.824/image-20230127231126414.png)



![在这里插入图片描述](Mit 6.824/20210618222859615.png)



集群支持四个操作：

- Join ：加入一个新的组 ，创建一个最新的配置，加入新的组后需要重新进行**负载均衡**。
- Leave: 将给定的一个组进行撤离，并进行**负载均衡**。
- Move: 为指定的分片，分配指定的组。
- Query: 查询特定的配置版本，如果给定的版本号不存在切片下标中，则直接返回最新配置版本。

操作的时候还要实现**负载均衡**

<img src="Mit 6.824/eeab56fdcce143629272127ca7294fd1.png" alt="在这里插入图片描述" style="zoom:67%;" />

- Balance的作用是使得每个Groups最后管理的Shards数目达到均衡，并且再现有修改后的分配上以代价小的形式来完成。
- Balance要求是一个确定性算法，即Shards[]的一个状态下，多次运行Balance方法，得到的多个Shards[]是相同的。
  - 常见的负载均衡算法：
  - 轮询，随机，哈希，加权，最小连接数。。。

**1.平均分配法：**

先确定分片总数n和分组总数m，取整平均数 n/m .

例：

10个分片分给4个组向上取整后为{3，3，2，2}。

再根据每个分组处理的分片数量，多的给少的。

**2.轮询平滑法：**

通过多次平均地方式来达到这个目的：每次选择一个拥有 shard 数最多的 raft 组和一个拥有 shard 数最少的 raft，将前者管理的一个 shard 分给后者，周而复始，直到它们之前的差值小于等于 1 且 0 raft 组无 shard 为止。

- **Join**

为当前集群添加一组raft服务器。

分片器应该通过创建包含新副本组的新配置来做出反应。新的配置应该在整个组中尽可能均匀地划分shard，并且应该尽可能少地移动shard以实现这一目标。如果GID不是当前配置的一部分，分片器应该允许重复使用GID（即，应该允许GID加入，然后离开，然后再次加入）

**注意：**

这些命令会在一个 raft 组内的所有节点上执行，因此需要保证同一个命令在不同节点上计算出来的新配置一致，而 go 中 map 的遍历是不确定性的，因此需要稍微注意一下确保相同地命令在不同地节点上能够产生相同的配置。

```go
//加入一个新的组 ，创建一个最新的配置，加入新的组后需要重新进行**负载均衡**。
func (sc *ShardCtrler) Join(args *JoinArgs, reply *JoinReply) {
	// Your code here.
	_, ifLeader := sc.rf.GetState()
	if !ifLeader {
		reply.Err = ErrWrongLeader
		return
	}

	// 封装Op传到下层start
	op := Op{
		Optype: Join,
		Servers: args.Servers,
		CommandId: args.CommandId,
		ClientId: args.ClientId,
	}
	//将操作写入日志
	lastIndex, _, _ := sc.rf.Start(op)
	//当这个操作被集群提交时，在调用对应的函数处理操作。
	ch := sc.GetNotifyChan(lastIndex)
	defer func() {
		sc.mu.Lock()
		delete(sc.notifyChans, lastIndex)
		sc.mu.Unlock()
	}()

	// 设置超时ticker
	timer := time.NewTicker(100 * time.Millisecond)
	defer timer.Stop()

	select {
	case reply := <-ch:

		if op.ClientId != reply.Op.ClientId || op.CommandId != reply.Op.CommandId {
			reply.Err = ErrWrongLeader
		} else {
			reply.Err = OK
			return
		}
	case <-timer.C:
		reply.Err = ErrTimeout
	}
}
func (sc* ShardCtrler) JoinHandler(joins map[int][]string ) Config{
	// 取出最后一个config将分组加进去
	lastConfig := sc.configs[len(sc.configs)-1]
	newGroups := make(map[int][]string)

	for gid, serverList := range lastConfig.Groups {
		newGroups[gid] = serverList
	}
	//如果join里有gid
	for gid, serverLists := range joins {
		newGroups[gid] = serverLists
	}
	stog := make(map[int][]int)
	//找到分片最少的分组，并尽量不同的gid对应的index数量差不多
	for index,gid := range lastConfig.Shards {
		stog[gid]=append(stog[gid],index)
	}
	//
	for {
		source, target := sc.Maxserver(stog),sc.Minserver(stog)
		if source != 0 && len(stog[source])-len(stog[target]) <= 1 {
			break
		}
		stog[target] = append(stog[target], stog[source][0])
		stog[source] = stog[source][1:]
	}
	var newShards [NShards]int
	for gid, shards := range stog {
		for _, shard := range shards {
			newShards[shard] = gid
		}
	}
	return &Config{
		Num:    len(sc.configs),
		Shards: newShards,
		Groups: newGroups,
	}
}
```



- leave

```go

//删除一组raft组。如果 Leave 后集群中无 raft 组，则将分片所属 raft 组都置为无效的 0；否则将删除 raft 组的分片均匀地分配给仍然存在的 raft 组。
func (sc* ShardCtrler) LeaveHandler(servers []int) *Config{
	lastConfig := sc.configs[len(sc.configs)-1]
	newGroups := make(map[int][]string)

	for gid, serverList := range lastConfig.Groups {
		newGroups[gid] = serverList
	}
	//stog[i]:这个分组要处理的分片。
	stog := make(map[int][]int)
	for index,gid := range lastConfig.Shards {
		stog[gid]=append(stog[gid],index)
	}
	//要处理的分片
	var LeaveShards []int
	for _,gid := range servers{
		if _,ok := newGroups[gid];ok{
			delete(newGroups,gid)
		}
		//如果这个gid有处理的分片
		if shards, ok := stog[gid]; ok {
			LeaveShards = append(LeaveShards, shards...)
			delete(stog, gid)
		}
	}
	var newShards [NShards]int
	//将分片分给最轻松的分组。
	if len(lastConfig.Groups) != 0 {
		for _, shard := range LeaveShards {
			target := sc.Minserver(stog)
			stog[target] = append(stog[target], shard)
		}
		for gid, shards := range stog {
			for _, shard := range shards {
				newShards[shard] = gid
			}
		}
	}
	return &Config{
		Num:    len(sc.configs),
		Shards: newShards,
		Groups: newGroups,
	}
}
```



### lab4B：高可用性的数据库

shard化存储系统必须能够在副本组之间移动shard。一个原因是某些组可能比其他组负载更大，因此需要移动shard来平衡负载。另一个原因是副本组可能会加入或离开系统：可能会添加新的副本组以增加容量，或者可能会使现有副本组脱机以进行修复或退役。



这个实验室的主要挑战是处理重新配置——改变shard分配给组的方式。在单个副本组中，所有组成员必须就何时发生与客户机放置/附加/获取请求相关的重新配置达成一致。**例如，Put可能在重新配置的同时到达，这会导致副本组停止对持有Put密钥的shard负责。组中的所有复制副本必须就Put是在重新配置之前还是之后发生达成一致。**如果在之前，Put应该生效，shard的新所有者将看到它的效果；如果之后，put操作没有起效，客户必须在新所有者处重新尝试。推荐的方法是让每个副本组使用Raft不仅记录Put、Appends和Get的序列，还记录重新配置的序列。您需要确保在任何时候最多有一个副本组为每个shard提供请求。



重新配置还需要副本组之间的交互。例如，在配置10中，组G1可以负责碎片S1。在配置11中，组G2可以负责碎片S1。在从10到11的重新配置期间，G1和G2必须使用RPC将碎片S1（键/值对）的内容从G1移动到G2。



只有RPC可以用于客户端和服务器之间的交互。例如，不允许服务器的不同实例共享Go变量或文件。



这个实验室使用“配置”来指代将shard分配给副本组。这与Raft集群成员身份更改不同。您不必实现Raft集群成员身份更改。

multi-raft就是用多组raft组来搭建分布式存储系统，lab3中是当raft组的，lab4相当于在各个lab3的单raft组中协调。

- 如何将数据分片到各个raft组上。

- 如何处理读写跨raft组的数据，您需要为跨shard移动的客户端请求提供最多一次语义（重复检测）。
- 客户端操作和负载均衡操作同时到达，该如何处理。
- CONFIG变化后，如何转移SHARD（如果一个REPLICA GROUP A得到一个SHARD 1，对应B 失去一个SHARD 1）

- 想想shardkv客户端和服务器应该如何处理ErrWrongGroup。如果客户端收到ErrWrongGroup，是否应更改序列号？如果在执行Get/Put请求时返回ErrWrongGroup，服务器是否应该更新客户端状态？

- 在服务器移动到新配置后，它可以继续存储不再拥有的shard（尽管这在真实系统中是令人遗憾的）。这可能有助于简化服务器实现。

- 当组G1在配置更改期间需要G2的shard时，G2在处理日志条目的过程中的什么时候将shard发送给G1是否重要？

- 如果您的一个RPC处理程序在其回复中包含一个映射（例如，键/值映射），该映射是服务器状态的一部分，那么您可能会由于竞争而出现错误。RPC系统必须读取映射才能将其发送给调用者，但它没有持有覆盖映射的锁。但是，当RPC系统读取同一映射时，服务器可能会继续修改该映射。解决方案是RPC处理程序在回复中包含该映射的副本。

- 如果您在Raft日志条目中放置了一个映射或切片，并且您的key/value服务器随后在applyCh上看到了该条目，并在key/value服务器的状态中保存了对映射/切片的引用，那么您可能会发生竞争。制作映射/切片的副本，并将副本存储在键/值服务器的状态中。竞争是你的键/值服务器修改映射/切片和Raft在持久化日志的同时读取它。

- 在配置更改期间，一对组可能需要在它们之间的两个方向上移动shard。如果您看到死锁，这可能是一个原因。

一开始系统会创建一个 shardctrler 组来负责配置更新，分片分配等任务，接着系统会创建多个 raft 组来承载所有分片的读写任务。此外，raft 组增删，节点宕机，节点重启，网络分区等各种情况都可能会出现。

对于集群内部，我们需要保证所有分片能够较为均匀的分配在所有 raft 组上，还需要能够支持动态迁移和容错。

对于集群外部，我们需要向用户保证整个集群表现的像一个永远不会挂的单节点 KV 服务一样，即具有线性一致性。

lab4b 的基本测试要求了上述属性，challenge1 要求及时清理不再属于本分片的数据，challenge2 不仅要求分片迁移时不影响未迁移分片的读写服务，还要求不同地分片数据能够独立迁移，即如果一个配置导致当前 raft 组需要向其他两个 raft 组拉取数据时，即使一个被拉取数据的 raft 组全挂了，另一个正常的被拉取的raft组也能正常提供服务 

(假设某个复制组G3在转换到C时需要G1的碎片S1和G2的碎片S2。我们确实希望G3在收到必要的状态后立即开始为碎片提供服务，即使它仍在等待其他碎片。例如，如果G1停机，G3一旦从G2接收到适当的数据，就应该开始为S2的请求提供服务，尽管到C的转换尚未完成。) 。

#### Client端

每个client有一个独一的ID（用snow算法生成），每个任务也有一个taskId，服务器通过clientId和taskId来避免任务重复执行

```go
//这个操作是否重复，因为commandid从客户端还是服务端都能保证它是递增的，所以只要和最后的commid相比就好了。
func (kv *KVServer) Isrepeat(Clinet int64,commid int)  bool{

	if value, ok := kv.lastClientOperation[Clinet]; ok {
		if value.SequenceNum <= commid {
			return true
		}
	}
	return false
}
```

- 缓存每个分片的 leader。
- rpc 请求成功且服务端返回 OK 或 ErrNoKey，则可直接返回。
- rpc 请求成功且**服务端返回 ErrWrongGroup，则需要重新获取最新 config 并再次发送请求。**
- rpc 请求失败一次，需要继续遍历该 raft 组的其他节点。
- rpc 请求失败副本数次，此时需要重新获取最新 config 并再次发送请求。

```go
func (ck *Clerk) Get(key string) string {
	ck.CommandId++
	args := GetArgs{
		ClientId: ck.ClientId,
		CommandId: ck.CommandId,
		Key: key,
	}

	for {
		//根据分片找到对应的的raft组
		shard := key2shard(key)
		gid := ck.config.Shards[shard]
		//尝试对这个raft组找到它的leader
		if servers, ok := ck.config.Groups[gid]; ok {
			// try each server for the shard.
				srv := ck.make_end(servers[ck.G2L[gid]])
				var reply GetReply
				ok := srv.Call("ShardKV.Get", &args, &reply)
            	if !ok{
                	break;
            	}
				if reply.Err == OK || reply.Err == ErrNoKey {
					return reply.Value
				}
                //如果找到错误分组，推出循环，去找最新的配置。
				if reply.Err == ErrWrongGroup {
					break
				}
            	if reply.Err==ErrWrongLeader{
                	ck.G2L[gid]=reply.leaderId
            	}
                //如果分组正在迁徙，等待一会重试
				// ... not ok, or ErrWrongLeader

		}
		time.Sleep(100 * time.Millisecond)
		// ask controler for the latest configuration.
		ck.config = ck.sm.Query(-1)
	}

	return ""
}

```



#### Server端

说到底，分布式的最终目的还是实现线性，就像jvm的happen before一样，不论处理器，编译器怎么优化，怎么改变运行顺序，最终的结果就与串行运行的结果一致。

- 如何保证各个集群配置的线性。

  配置的更新只能一步一步递增，且必须等到数据迁徙完成才能更新配置信息。如果当前节点获取到的配置信息比其他集群新，必须阻塞，等到所有集群的配置信息一致。

- 如何保证数据迁徙的线性。

  数据迁徙的方向只能是单向，要么从需要方流向多余方，要么从多余方流向需要方。

  - 为了保证在数据迁徙时尽可能的提供服务，数据迁徙的粒度就必须以分片为单位。所以各个分片的配置序列可能会在一段时间内不一致，但是配置序列的不一致又会阻塞配置的更新，所以最后集群各个分片的状态一定会达成一致。

- 如何保证集群内部数据的线性。

  不论是集群配置还是数据迁徙，都要通过raft组写日志来达成本集群的共识。

<img src="Mit 6.824/image-20230207154245836.png" alt="image-20230207154245836" style="zoom:80%;" />



服务端数据结构

我们的kv系统要求配置的更新和分片的状态变化彼此独立，所以很显然地一个答案就是我们不仅不能在配置更新时同步阻塞的去拉取数据，也不能异步的去拉取所有数据并当做一条 raft 日志提交，而是应该将不同 raft 组所属的分片数据独立起来，分别提交多条 raft 日志来维护状态。因此，ShardKV 应该对每个分片额外维护其它的一些状态变量。

处理客户端或其他集群的操作时，先判断配置信息是否一致，如果一致，继续判断该分片的配置信息是否一致，如果一致，返回结果。

```go
//读写的粒度是以分片为单位，分片可用的时机就是分片的配置序列和节点的配置序列一致。
type MemoryKV struct {
	Shardmap []Shard
}
type Shard struct {
	KvMap     map[string]string
	ConfigNum int 
}

func NewMemoryKV(NShards int) *MemoryKV {

	return &MemoryKV{ make([]Shard, NShards)}
}
```

**如果leader在更新配置的过程挂掉了，如何保证其余节点不会重复数据迁徙**

配置信息都保存了，分片信息还未保存。这时新leader的配置协程会检查是否有分片未处理，如果有，再去处理数据迁徙。

配置信息保存，一部分片信息保存了，新leader的配置协程同样会处理剩余未能达成一致性的分片信息。

最后更新分片时，再次检查分片的日志序号，防止重复处理。

```go
//不断的判断更新配置---有新配置---raft集群达成共识---去拉取相应的
func (kv*ShardKV) UpdateCfg() {
	for kv.Killed()==false {
		time.Sleep(Upcfgtime)
		//只有leader能查看配置，因为只有leader能将新配置发给其他节点。
		if kv.rf.statue!=leader || !kv.CheckFinish() {
			continue
		}

		currentcfg,flag := kv.Checkcfg(kv.currentConfig.num+1)
		if !flag {
			continue
		}
		kv.lastConfig=kv.currentConfig
		kv.currentConfig=currentcfg

        
        //如果配置和分片有一个不一致，就不能再继续更新配置，必须阻塞直到配置和分片趋于一致。
		//配置有更新。先在raft集群中达成共识
		kv.RaftUpcfg()
		//然后再做数据迁徙，以分片为粒度。
		kv.CheckFinish()

	}

}
```

节点再查询新配置时，一定要先完成之前的配置更新。

```go
//配置是否更新完,是否发送完，是否接收完。底层是否已经更新了配置。
func (kv* ShardKV)CheckFinish()bool  {
	//如果当前的Config
	var flag bool
	for shard, gid := range kv.lastConfig.Shards {

		// 判断切片是否都发送了
		if gid == kv.gid && kv.Config.Shards[shard] != kv.gid {
            //如果还没发送，发送数据。
			if  kv.shardsPersist[shard].ConfigNum < kv.Config.Num{

				sendDate := kv.cloneShard(kv.Config.Num, kv.shardsPersist[shardId].KvMap)
                //bug：：这里注意要将我们保存的各个客户的最后Command也一并发送过去，因为对上层而言数据迁徙是透明的，所以客户的最后command应该是集群间共享的！！
				args := SendShardArg{
					LastAppliedRequestId: kv.lastCommand,
					ShardId:              shardId,
					Shard:                sendDate,
					ClientId:             int64(gid),
                    //如果接收方的任期序列比我们还低它会返回false，我们不断尝试向他发送直到它配置追上来。
					RequestId:            kv.Config.Num,
				}
				kv.SendShard(kv.Config.Shards[shard],args)
				flag = false
			}
		}
	}

	if !flag {
		time.Sleep(Upcfgtime)
		return false
	}
	for shard, gid := range kv.lastConfig.Shards {

		// 判断切片是否都收到了
		if gid != kv.gid && kv.Config.Shards[shard] == kv.gid  {
			if  kv.shardsPersist[shard].ConfigNum < kv.Config.Num{
				return false
			}
		}
	}

	return true

}
```

向集群发送分片

```go
//向这个集群发送分片,如果该集群的配置信息比我们落后怎么办？那么该集群会返回错误，因为他
func (kv *ShardKV)SendShard(gid int,args * SendShardArg)  {


	// shardId -> gid -> server names
	serversList := kv.Config.Groups[kv.Config.Shards[gid]]
	servers := make([]*labrpc.ClientEnd, len(serversList))
	for i, name := range serversList {
		servers[i] = kv.make_end(name)
	}
	go func(servers []*labrpc.ClientEnd, args *SendShardArg) {

		index := 0
		start := time.Now()
		for {
			var reply AddShardReply
			// 对自己的共识组内进行add
			ok := servers[index].Call("ShardKV.AddShard", args, &reply)

			// 如果给予切片成,抛弃自己的切片
			if ok && reply.Err == OK || time.Now().Sub(start) >= 2*time.Second {

				// 如果成功发送，更新集群删除该分片。
				//如果发送成功，而leader挂掉了，怎么处理。如果leader此时挂掉了，其他节点并未删除分片，所以会再次发送分片，
				//如果接收方收到了已经存在的分片，返回true。
				kv.mu.Lock()
				command := Op{
					Optype:   DeleteShard,
					ClientId: int64(kv.gid),
					CommandId:    kv.Config.Num,
					ShardId:  args.ShardId,
				}
				kv.mu.Unlock()
				kv.Command(command, RemoveShardsTimeout)
				break
			}
			index = (index + 1) % len(servers)
			if index == 0 {
				time.Sleep(UpConfigLoopInterval)
			}
		}
	}(servers, args)
}
```



```go

func (kv *ShardKV) AddShard(args *SendShardArg, reply *AddShardReply) {
	command := Op{
		OpType:   AddShardType,
		ClientId: args.ClientId,
		SeqId:    args.RequestId,
		ShardId:  args.ShardId,
		Shard:    args.Shard,
		SeqMap:   args.LastAppliedRequestId,
	}
	reply.Err = kv.Command(command, AddShardsTimeout)
	return
}
//接收方
func (kv *ShardKV) addShardHandler(op Op) {
    //如果我已经接收过该分片了，或者你的配置已经过时了，返回true。
	if op.Shard.ConfigNum <= kv.Db[op.ShardId].ConfigNum || op.Shard.ConfigNum < kv.Config.Num {
		return true
	}

	kv.shardsPersist[op.ShardId] = kv.cloneShard(op.Shard.ConfigNum, op.Shard.KvMap)
	//更新这个分片客户端的信息。
	for clientId, seqId := range op.SeqMap {
		if r, ok := kv.SeqMap[clientId]; !ok || r < seqId {
			kv.SeqMap[clientId] = seqId
		}
	}
}
```





服务端不断先在raft集群达成共识，再调用回调函数处理。

```go
func (kv *ShardKV)Command(command Op,tiemout time.Millisecond) Err {
	_, ifLeader := kv.rf.GetState()
	if !ifLeader {
		return ErrWrongLeaderr
	}

	//将操作写入日志
	lastIndex, _, _ := kv.rf.Start(command)
	//当这个操作被集群提交时，在调用对应的函数处理操作。
	ch := kv.GetNotifyChan(lastIndex)
	defer func() {
		kv.mu.Lock()
		delete(kv.notifyChans, lastIndex)
		kv.mu.Unlock()
	}()

	// 设置超时ticker
	timer := time.NewTicker(100 * time.Millisecond)
	defer timer.Stop()

	select {
	case reply := <-ch:
		//可能在处理的过程中leader挂掉了，该日志并未提交。另一个leader上位覆盖掉了该日志，虽然他们的日志下标相同，但是日志代表的内容不同。
		if command.ClientId != reply.Op.ClientId || command.CommandId != reply.Op.CommandId {
			return  ErrWrongLeader
		} else {
			return reply.Err
		}
	case <-timer.C:
		return ErrTimeout
	}
}
```





```go
//下层的raft层提交数据上来。
func (kv* ShardKV) applier()  {
	for kv.killed() == false {
		select {
		case message := <-kv.applyCh:

			if message.CommandValid {
				kv.mu.Lock()
				if message.CommandIndex <= kv.lastApplied {
					kv.mu.Unlock()
					continue
				}
				kv.lastApplied = message.CommandIndex

				var response CommandReply
				command := message.Command.(Op)
				switch command.Optype {
				case Upcfg:
					response = kv.UpdateCfg(&command)
				case DeleteShard:
					response = kv.DeleteShard(&command)
				case AddShard:
					response=kv.AddShard(&command)
				case Get:
					response = kv.GetValue(&command)
				case Put:
					response = kv.PutValue(&command)
				case Append:
					response = kv.AppendValue(&command)
				}

				// 如果需要snapshot，且超出其stateSize,告诉raft层开始压缩。
				if kv.maxraftstate != -1 && kv.rf.GetRaftStateSize() > kv.maxraftstate {
					snapshot := kv.PersistSnapShot()
					kv.rf.Snapshot(message.CommandIndex, snapshot)
				}
				kv.mu.Unlock()
				//如果发来的是快照。
			} else if message.SnapshotValid {
				kv.mu.Lock()
				if kv.rf.CondInstallSnapshot(message.SnapshotTerm, message.SnapshotIndex, message.Snapshot) {
					kv.DecodeSnapShot(message.Snapshot)
					kv.lastApplied = message.SnapshotIndex
				}
				kv.mu.Unlock()
			} else {
				panic(fmt.Sprintf("unexpected Message %v", message))
			}
		}
	}
}
```

可以看到，如果当前 raft 组在当前 config 下负责管理此分片，则只要分片的配置序列号相同，本 raft 组就可以为该分片提供读写服务，否则返回 ErrWrongGroup 让客户端重新 fecth 最新的 config 并重试即可。

读写操作的基本逻辑和 lab3 一致，可以在向 raft 提交前和 apply 时都检测一遍是否重复以保证线性化语义。

```c++

func (kv *ShardKV) Get(args *GetArgs, reply *GetReply) {
	// Your code here.
	shardId := key2shard(args.Key)
	kv.mu.Lock()
	if kv.Config.Shards[shardId] != kv.gid {
		reply.Err = ErrWrongGroup
	} else if kv.Db.SGetShard(shardId).KvMap == nil {
		reply.Err = ShardNotArrived
	}
	kv.mu.Unlock()
	if reply.Err == ErrWrongGroup || reply.Err == ShardNotArrived {
		return
	}
	command := Op{
		Optype:   Get,
		ClientId: args.ClientId,
		CommandId:args.CommandId,
		Key:      args.Key,
	}
	commandreply := kv.Command(command, GetTimeout)
	if err != OK {
		reply.Err = commandreply.Err
		return
	}
	kv.mu.Lock()
	
    reply.Err=commandreply.value
	kv.mu.Unlock()
	return

}

func (kv *ShardKV) PutAppend(args *PutAppendArgs, reply *PutAppendReply) {
	// Your code here.
	shardId := key2shard(args.Key)
	kv.mu.Lock()
	if kv.Config.Shards[shardId] != kv.gid {
		reply.Err = ErrWrongGroup
	} else if kv.Db.SGetShard(shardId).KvMap == nil {
		reply.Err = ShardNotArrived
	}
	kv.mu.Unlock()
	if reply.Err == ErrWrongGroup || reply.Err == ShardNotArrived {
		return
	}
	command := Op{
		Optype:   optype(args.Op),
		ClientId: args.ClientId,
		CommandId:args.CommandId,
		Key:      args.Key,
		Value:    args.Value,
	}
	reply.Err = kv.Command(command, AppOrPutTimeout).Err
	return
}
```

