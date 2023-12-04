# RAFT with Maude


## confluent RAFT 모델링

[oraft-confluent.maude](./oraft-confluent.maude) 파일은 교수님의 피드백 후 수정한 파일입니다.

```
load oraft-confluent-maude .

--- 실행 확인
rew [3] test3Node .
```
위 명령어로 load하고, 실행 여부를 확인할 수 있습니다.


교수님께서 주신 피드백은 크게 2가지가 있었습니다.
1. omod로 리팩토링하기
    - 이 때 class를 바꾸지 않는 방식으로 모델링할 것.
2. 현재 추상화 레벨에서 confluent하게 모델링하기
    - token ring을 사용하지 않고, 모든 rule을 non-deterministic하게 변경하기
    - node의 kill, revive, timeout도 모델링에 포함시켜야 한다.
3. log contains에 owise 넣기

### 1. omod로 리팩토링
기존 module로 정의한 것을 omod로 변경했습니다.

```
class ONode | type : NodeType, nodeTerm : Nat, log : Log, committedLog : Log,
    neighbors : OidSet, isWaiting : Bool,
    quorum : Nat, numNeighbors : Nat, numPros : Nat,
    numResponses : Nat .

sort NodeType .
ops FollowerNode LeaderNode OfflineNode CandidateNode : -> NodeType [ctor] .
```

class는 위와 같이 정의했습니다. NodeType은 제가 별도로 선언한 subsort입니다.


```
--- 기존 코드
crl [candidate-defeat-vote] :
    < leader : CandidateNode | nodeTerm : currentTerm, nextNeighbor : nextNode, quorum : quorum, numPros : numPros, numNeighbors : numNeighbors, numResponses : numNeighbors, AS >
    =>
    < leader : FollowerNode | nodeTerm : currentTerm, nextNeighbor : nextNode, quorum : quorum, numPros : numPros, numNeighbors : numNeighbors, numResponses : numNeighbors, AS >
    (msg timeout(currentTerm + 1) from leader to nextNode)
    if numPros < quorum .

--- 변경 후 코드
crl [candidate-defeat-vote] :
    < Candidate : ONode | type : CandidateNode, nodeTerm : CurrentTerm, quorum : Quorum, numPros : NumPros, numNeighbors : NumNeighbors, numResponses : NumNeighbors >
    =>
    < Candidate : ONode | type : FollowerNode >
    if NumPros < Quorum .
```

한편, 이에 따라 교수님이 언급해 주신 것처럼 새로운 class가 생기는 것이 아니라, class 내부에 지정한 type이 변경되게 변경했습니다. 예시는 위 코드와 같습니다.

또한 convention도 수정했습니다. 교수님께서 언급해 주신 것과 같이 var로 선언된 것들은 대문자로 시작하게 변경했습니다.

### 2. confluent 모델링
기존에 제가 제시했던 코드는 초기화 단계에서 직접 kill, revive, timeout을 지정하는 방식이었습니다. 때문에 모델 내부에서 kill, revive, timeout 동작이 수행될 수 없는 deterministic한 방식이었습니다.

교수님께서 피드백해주신 내용대로 해당 부분을 삭제하고, 각 node가 nondeterministic하게 kill(offline node로 변경), revive(offline node가 follower node로 변경), timeout(follower node가 candidate node로 변경)하는 rewriting logic을 추가했습니다.

```
  --- leader가 죽는 경우
  rl [leader-become-offline] :
    < CurrentNode : ONode | type : LeaderNode >
    =>
    < CurrentNode : ONode | type : OfflineNode > .

  --- follower를 죽이는 경우
  rl [follower-become-offline] :
    < CurrentNode : ONode | type : FollowerNode >
    =>
    < CurrentNode : ONode | type : OfflineNode > .

  --- candidate를 죽이는 경우
  rl [die] :
    < CurrentNode : ONode | type : CandidateNode >
    =>
    < CurrentNode : ONode | type : OfflineNode > .

  --- offline이 다시 online으로 변경
  rl [come-online] :
    < CurrentNode : ONode | type : OfflineNode >
    =>
    < CurrentNode : ONode | type : FollowerNode > .
```

추가한 로직들은 위와 같습니다.

또한 피드백해주신 log contains에 owise를 추가하는 피드백도 반영했습니다.

```
--- 기존 코드
op contains : Log Entry -> Bool .
eq contains(Arr1 ++ E ++ Arr2, E) = true .
eq contains(Arr1, E) = false [owise] .

--- 변경 후 코드
op contains : Log Entry -> Bool .
eq contains(arr1 ++ entry ++ arr2, entry) = true .
eq contains(arr1, entry) = false . --- owise
```

### 실행 결과
그렇게 실행한 결과, test3Node에서 search 명령어에 대해 depth를 지정했을 때 state 개수는 다음과 같습니다.

교수님이 말씀해 주신 대로, nondeterministic하게 설정하면 state의 개수가 급격하게 증가하는 것을 확인할 수 있었습니다.
```
search [1, 5] test3Node =>! C:Configuration .
depth 5: states: 1326  rewrites: 11971 in 28ms cpu (26ms real) (427535 rewrites/second)

search [1, 6] test3Node =>! C:Configuration .
depth 6: states: 3529  rewrites: 31278 in 63ms cpu (63ms real) (489590 rewrites/second)

search [1, 7] test3Node =>! C:Configuration .
depth 7: states: 9153  rewrites: 80878 in 211ms cpu (236ms real) (381535 rewrites/second)

search [1, 8] test3Node =>! C:Configuration .
depth 8: states: 23228  rewrites: 204540 in 5478ms cpu (5582ms real) (37332 rewrites/second)

search [1, 9] test3Node =>! C:Configuration .
depth 9: states: 57904  rewrites: 506788 in 9233ms cpu (9370ms real) (54883 rewrites/second)

search [1, 10] test3Node =>! C:Configuration .
depth 10: states: 142304  rewrites: 1244049 in 25047ms cpu (25330ms real) (49667 rewrites/second)

search [1, 11] test3Node =>! C:Configuration .
depth 11: states: 345930  rewrites: 3046704 in 86698ms cpu (87949ms real) (35141 rewrites/second)
```

depth가 11 이상인 경우에는 너무 오랜 시간이 걸렸기 때문에, 해당 depth 내에서만 검증을 진행했습니다.

#### leader state가 존재하는지
```
search [1, 10] test3Node =>* C:Configuration
< O1:Oid : ONode | type : LeaderNode, AS1:AttributeSet > .

Solution 1 (state 861)
states: 862  rewrites: 7296 in 10ms cpu (17ms real) (729600 rewrites/second)
C:Configuration -->
< client : Client | none >
< onode(1) : ONode | type : FollowerNode, nodeTerm : 1, log : entry(0, 0, "startup"), committedLog : entry(0, 0, "startup"), neighbors : (onode(0),onode(2)), isWaiting : false, quorum : 2, numNeighbors : 2, numPros : 1, numResponses : 0 >
< onode(2) : ONode | type : FollowerNode, nodeTerm : 1, log : entry(0, 0, "startup"), committedLog : entry(0, 0, "startup"), neighbors : (onode(0),onode(1)), isWaiting : false, quorum : 2, numNeighbors : 2, numPros : 1, numResponses : 0 >
(msg clientRequest("client-request") from client to onode(0))
(msg clientRequest("client-request2") from client to onode(0))
(msg requestAppendEntry(1, entry(0, 0, "startup")) from onode(0) to onode(1))
msg requestAppendEntry(1, entry(0, 0, "startup")) from onode(0) to onode(2)

O1:Oid --> onode(0)
AS1:AttributeSet --> nodeTerm : 1, log : entry(0, 0, "startup"), committedLog : entry(0, 0, "startup"), neighbors : (onode(1),onode(2)), isWaiting : true, quorum : 2, numNeighbors : 2, numPros : 1, numResponses : 0
```

leader로 가는 state가 존재함을 알 수 있습니다.

#### leader safety
leader safety는 leader가 2개 존재하지 않는지 여부입니다.
```
search [1, 10] test3Node =>* C:Configuration
    < O1:Oid : ONode | type : LeaderNode, AS1:AttributeSet >
    < O2:Oid : ONode | type : LeaderNode, AS2:AttributeSet > .

No solution.
states: 97087  rewrites: 824356 in 33910ms cpu (33907ms real) (24310 rewrites/second)
```

depth 10 내에서 leader가 2개인 state는 존재하지 않음을 알 수 있습니다.

#### log matching
log matching은 같은 log일 때 다른 내용의 log 존재 여부입니다.
```
search [1, 10] test3Node =>* C:Configuration
    < O1:Oid : ONode | log : L11:Log ++ entry(ind:Nat, term:Nat, C1:Command) ++ L12:Log, AS1:AttributeSet >
    < O2:Oid : ONode | log : L21:Log ++ entry(ind:Nat, term:Nat, C2:Command) ++ L22:Log, AS2:AttributeSet >
    such that C1:Command =/= C2:Command or L11:Log =/= L21:Log .

No solution.
states: 97087  rewrites: 4319488 in 64380ms cpu (64388ms real) (67093 rewrites/second)
```
depth 10 내에서 log가 동일할 때 command가 다른 log는 존재하지 않음을 알 수 있습니다.

#### state machine safety
state machine safety는 같은 committed log일 때 다른 내용의 committed log 존재 여부입니다.
```
search [1, 10] test3Node =>* C:Configuration
    < O1:Oid : ONode | committedLog : L11:Log ++ entry(ind:Nat, term:Nat, C1:Command) ++ L12:Log, AS1:AttributeSet >
    < O2:Oid : ONode | committedLog : L21:Log ++ entry(ind:Nat, term:Nat, C2:Command) ++ L22:Log, AS2:AttributeSet >
    such that C1:Command =/= C2:Command or L11:Log =/= L21:Log .

No solution.
states: 97087  rewrites: 4319488 in 61150ms cpu (61143ms real) (70637 rewrites/second)
```
depth 10 내에서 committed log가 동일할 때 command가 다른 committed log는 존재하지 않음을 알 수 있습니다.



## token ring을 사용한 RAFT 모델링

[raft-token-ring.maude](./raft-token-ring.maude) 파일은 데모 때 보여드린 코드입니다.

[oraft-confluent.maude](./oraft-confluent.maude)에서는 state가 3개일 때 depth 11 이상으로만 넘어가도 연산에 많은 시간이 걸리기 때문에 이를 개선하기 위해 아래와 같은 제약들을 추가했습니다.

### 위 버전과 차이점
1. kill, revive, timeout을 초기화 시 직접 관리합니다.
    - node에 문제가 생겨 offline으로 변화하는 것을 rewriting logic으로 작성할 수 있지만, kill과 revive를 반복하면서 state가 계속 추가되기 때문에 이를 줄이는 효과가 있습니다.
    - 그러나 kill, revive의 경우에는 초기화 시 사용한다는 점 때문에 모델링에 제약을 걸게 됩니다.
2. token ring 방식을 사용합니다.
    - 원본 RAFT에서는 heartbeat를 사용한 random timeout을 사용해 각 node의 timeout을 관리합니다. 이 방식은 한 번에 하나의 node만 timeout이 될 수 있게 확률적으로 보장하는 특성이기 때문에, 한 번에 하나의 node만 timeout이 발생했다고 볼 수 있습니다. 따라서 token ring 방식을 차용해 한 번에 하나의 node만 timeout이 발생한다는 것을 보장할 수 있습니다.
    - 한 번에 하나의 node만 timeout이 발생하기 때문에 대부분의 로직이 deterministic하게 동작합니다. 때문에 state의 수를 줄일 수 있습니다.
    - 실행 도중에는 candidate가 2개 이상 발생할 수 없지만, 초기화를 통해 leader election에 있어 경쟁 상태를 표현할 수 있습니다.

### 실행 결과
#### test3Node
test3Node는 node가 3개일 때 leader election을 검증할 수 있는 모듈입니다.
```
search [1] test3Node =>! C:Configuration .

Solution 1 (state 541)
states: 543  rewrites: 14785 in 30ms cpu (27ms real) (492833 rewrites/second)
C:Configuration -->
< client : Client | none >
< node(0) : LeaderNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request") ++ entry(2, 1, "client-request2")),
committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request") ++ entry(2, 1, "client-request2")), neighbors : (node(1),node(2)), isWaiting : false, nextNeighbor : node(1), quorum : 2, numNeighbors : 2, numPros : 3, numResponses : 2 >
< node(1) : FollowerNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request") ++ entry(2, 1, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request") ++ entry(2, 1, "client-request2")), neighbors : (node(0),node(2)), isWaiting : false, nextNeighbor : node(2), quorum : 2, numNeighbors : 2, numPros : 1, numResponses : 0 >
< node(2) : FollowerNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request") ++ entry(2, 1, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request") ++ entry(2, 1, "client-request2")), neighbors : (node(0),node(1)), isWaiting : false, nextNeighbor : node(0), quorum : 2, numNeighbors : 2, numPros : 1, numResponses : 0 >
```
위 코드는 실행을 확인하는 코드입니다.

```
--- 2개의 leader 존재 여부 검증 (leader safety)
search [1] test3Node =>* C:Configuration
    < O1:Oid : LeaderNode | AS1:AttributeSet >
    < O2:Oid : LeaderNode | AS2:AttributeSet > .

No solution.
states: 543  rewrites: 14785 in 40ms cpu (35ms real) (369625 rewrites/second)

--- 같은 log일 때 다른 log 존재 여부 검증 (log matching)
search [1] test3Node =>* C:Configuration
    < O1:Oid : C1:Cid | log : L11:Log ++ entry(ind:Nat, term:Nat, C1:Command) ++ L12:Log, AS1:AttributeSet >
    < O2:Oid : C2:Cid | log : L21:Log ++ entry(ind:Nat, term:Nat, C2:Command) ++ L22:Log, AS2:AttributeSet >
    such that C1:Command =/= C2:Command or L11:Log =/= L21:Log .

No solution.
states: 543  rewrites: 62749 in 2890ms cpu (2892ms real) (21712 rewrites/second)

--- 같은 committed log일 때 다른 log 존재 여부 검증 (state machine safety)
search [1] test3Node =>* C:Configuration
    < O1:Oid : C1:Cid | committedLog : L11:Log ++ entry(ind:Nat, term:Nat, C1:Command) ++ L12:Log, AS1:AttributeSet >
    < O2:Oid : C2:Cid | committedLog : L21:Log ++ entry(ind:Nat, term:Nat, C2:Command) ++ L22:Log, AS2:AttributeSet >
    such that C1:Command =/= C2:Command or L11:Log =/= L21:Log .

No solution.
states: 543  rewrites: 44221 in 540ms cpu (541ms real) (81890 rewrites/second)
```
이후 leader satefy, log matching, state machine safety 3가지에 대해 검증했고, no solution이 나온 것으로 보아 모두 보장됩니다.

#### testKill
testKill은 node가 3개일 때 kill + leader election을 검증할 수 있는 모듈입니다.
```
search [1] testKill =>! C:Configuration .

Solution 1 (state 2885)
states: 2899  rewrites: 55190 in 1110ms cpu (1111ms real) (49720 rewrites/second)
C:Configuration -->
< client : Client | none >
< node(0) : LeaderNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request") ++ entry(2, 1, "client-request2")),
committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request") ++ entry(2, 1, "client-request2")), neighbors : (node(1),node(2)), isWaiting : false, nextNeighbor : node(1), quorum : 2, numNeighbors : 2, numPros : 2, numResponses : 2 >
< node(1) : FollowerNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request") ++ entry(2, 1, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request") ++ entry(2, 1, "client-request2")), neighbors : (node(0),node(2)), isWaiting : false, nextNeighbor : node(2), quorum : 2, numNeighbors : 2, numPros : 1, numResponses : 0 >
< node(2) : OfflineNode | nodeTerm : 1, log : entry(0, 0, "startup"), committedLog : entry(0, 0, "startup"), neighbors : (node(0),node(1)), isWaiting : false, nextNeighbor : node(0), quorum : 2, numNeighbors : 2, numPros : 1, numResponses : 0 >
```
위 코드는 실행을 확인하는 코드입니다. `node(2) : OfflineNode`인 것을 보아 kill이 잘 수행된 것을 확인할 수 있습니다.

```
--- 2개의 leader 존재 여부 검증 (leader safety)
search [1] testKill =>* C:Configuration
    < O1:Oid : LeaderNode | AS1:AttributeSet >
    < O2:Oid : LeaderNode | AS2:AttributeSet > .

No solution.
states: 2899  rewrites: 55190 in 1230ms cpu (1231ms real) (44869 rewrites/second)

--- 같은 log일 때 다른 log 존재 여부 검증 (log matching)
search [1] testKill =>* C:Configuration
    < O1:Oid : C1:Cid | log : L11:Log ++ entry(ind:Nat, term:Nat, C1:Command) ++ L12:Log, AS1:AttributeSet >
    < O2:Oid : C2:Cid | log : L21:Log ++ entry(ind:Nat, term:Nat, C2:Command) ++ L22:Log, AS2:AttributeSet >
    such that C1:Command =/= C2:Command or L11:Log =/= L21:Log .

No solution.
states: 2899  rewrites: 275714 in 4980ms cpu (4973ms real) (55364 rewrites/second)

--- 같은 committed log일 때 다른 log 존재 여부 검증 (state machine safety)
search [1] testKill =>* C:Configuration
    < O1:Oid : C1:Cid | committedLog : L11:Log ++ entry(ind:Nat, term:Nat, C1:Command) ++ L12:Log, AS1:AttributeSet >
    < O2:Oid : C2:Cid | committedLog : L21:Log ++ entry(ind:Nat, term:Nat, C2:Command) ++ L22:Log, AS2:AttributeSet >
    such that C1:Command =/= C2:Command or L11:Log =/= L21:Log .

No solution.
states: 2899  rewrites: 198242 in 4500ms cpu (4504ms real) (44053 rewrites/second)
```
이후 leader satefy, log matching, state machine safety 3가지에 대해 검증했고, no solution이 나온 것으로 보아 모두 보장됩니다.

#### testRevive
testRevive는 node가 3개일 때 revive를 검증할 수 있는 모듈입니다.
```
search [1] testRevive =>! C:Configuration .

Solution 1 (state 414)
states: 418  rewrites: 6318 in 150ms cpu (143ms real) (42120 rewrites/second)
C:Configuration -->
< client : Client | none >
< node(0) : LeaderNode | nodeTerm : 0, log : (entry(0, 0, "startup") ++ entry(1, 0, "client-request") ++ entry(2, 0, "client-request2")),
committedLog : (entry(0, 0, "startup") ++ entry(1, 0, "client-request") ++ entry(2, 0, "client-request2")), neighbors : (node(1),node(2)), isWaiting : false, nextNeighbor : node(1), quorum : 2, numNeighbors : 2, numPros : 3, numResponses : 2 >
< node(1) : FollowerNode | nodeTerm : 0, log : (entry(0, 0, "startup") ++ entry(1, 0, "client-request") ++ entry(2, 0, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 0, "client-request") ++ entry(2, 0, "client-request2")), neighbors : (node(0),node(2)), isWaiting : false, nextNeighbor : node(2), quorum : 2, numNeighbors : 2, numPros : 1, numResponses : 0 >
< node(2) : FollowerNode | nodeTerm : 0, log : (entry(0, 0, "startup") ++ entry(1, 0, "client-request") ++ entry(2, 0, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 0, "client-request") ++ entry(2, 0, "client-request2")), neighbors : (node(0),node(1)), isWaiting : false, nextNeighbor : node(0), quorum : 2, numNeighbors : 2, numPros : 1, numResponses : 0 >
```
위 코드는 실행을 확인하는 코드입니다. `node(2) : FollowerNode`인 것을 보아 revive가 잘 수행된 것을 확인할 수 있습니다.

```
--- 2개의 leader 존재 여부 검증 (leader safety)
search [1] testRevive =>* C:Configuration
    < O1:Oid : LeaderNode | AS1:AttributeSet >
    < O2:Oid : LeaderNode | AS2:AttributeSet > .

No solution.
states: 418  rewrites: 6318 in 180ms cpu (181ms real) (35100 rewrites/second)

--- 같은 log일 때 다른 log 존재 여부 검증 (log matching)
search [1] testRevive =>* C:Configuration
    < O1:Oid : C1:Cid | log : L11:Log ++ entry(ind:Nat, term:Nat, C1:Command) ++ L12:Log, AS1:AttributeSet >
    < O2:Oid : C2:Cid | log : L21:Log ++ entry(ind:Nat, term:Nat, C2:Command) ++ L22:Log, AS2:AttributeSet >
    such that C1:Command =/= C2:Command or L11:Log =/= L21:Log .

No solution.
states: 418  rewrites: 35430 in 430ms cpu (438ms real) (82395 rewrites/second)

--- 같은 committed log일 때 다른 log 존재 여부 검증 (state machine safety)
search [1] testRevive =>* C:Configuration
    < O1:Oid : C1:Cid | committedLog : L11:Log ++ entry(ind:Nat, term:Nat, C1:Command) ++ L12:Log, AS1:AttributeSet >
    < O2:Oid : C2:Cid | committedLog : L21:Log ++ entry(ind:Nat, term:Nat, C2:Command) ++ L22:Log, AS2:AttributeSet >
    such that C1:Command =/= C2:Command or L11:Log =/= L21:Log .

No solution.
states: 418  rewrites: 26022 in 390ms cpu (385ms real) (66723 rewrites/second)
```
이후 leader satefy, log matching, state machine safety 3가지에 대해 검증했고, no solution이 나온 것으로 보아 모두 보장됩니다.


#### test4Node
test4Node는 node가 4개일 때 초기화하는 모듈입니다.
```
search [1] test4Node =>! C:Configuration .

Solution 1 (state 5101)
states: 5103  rewrites: 152842 in 3180ms cpu (3184ms real) (48063 rewrites/second)
C:Configuration -->
< client : Client | none >
< node(0) : LeaderNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")),
committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), neighbors : (node(1),node(2),node(3)), isWaiting : false, nextNeighbor : node(1), quorum : 3, numNeighbors : 3, numPros : 3, numResponses : 3 >
< node(1) : FollowerNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), neighbors : (node(0),node(2),node(3)), isWaiting : false, nextNeighbor : node(2), quorum : 3, numNeighbors : 3, numPros : 1, numResponses : 0 >
< node(2) : FollowerNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), neighbors : (node(0), node(1),node(3)), isWaiting : false, nextNeighbor : node(3), quorum : 3, numNeighbors : 3, numPros : 1, numResponses : 0 >
< node(3) : OfflineNode | nodeTerm : 0, log : entry(0, 0, "startup"), committedLog : entry(0, 0, "startup"), neighbors : (node(0),node(1),node(2)), isWaiting : false, nextNeighbor : node(0), quorum : 3, numNeighbors : 3, numPros : 1, numResponses : 0 >
```

#### test4NodeCompetition
test4NodeCompetition는 node가 4개 + candidate가 2개일 때, 경쟁 상태에서 leader가 어떻게 나오는지 검증하는 모듈입니다.
```
--- 2개의 leader 존재 여부 검증 (leader safety)
search [1] test4NodeCompetition =>* C:Configuration
    < O1:Oid : LeaderNode | AS1:AttributeSet >
    < O2:Oid : LeaderNode | AS2:AttributeSet > .

No solution.
states: 952  rewrites: 17635 in 350ms cpu (348ms real) (50385 rewrites/second)

--- node(0)이 leader가 되는지 여부 검증
search [1] test4NodeCompetition =>* C:Configuration
    < node(0) : LeaderNode | AS1:AttributeSet > .

Solution 1 (state 328)
states: 329  rewrites: 6752 in 140ms cpu (134ms real) (48228 rewrites/second)
C:Configuration -->
< client : Client | none >
< node(1) : FollowerNode | nodeTerm : 1, log : entry(0, 0, "startup"), committedLog : entry(0, 0, "startup"), neighbors : (node(0),node(2),node( 3)), isWaiting : false, nextNeighbor : node(2), quorum : 3, numNeighbors : 3, numPros : 1, numResponses : 0 >
< node(2) : FollowerNode | nodeTerm : 1, log : entry(0, 0, "startup"), committedLog : entry(0, 0, "startup"), neighbors : (node(0),node(1),node(3)), isWaiting : false, nextNeighbor : node(3), quorum : 3, numNeighbors : 3, numPros : 1, numResponses : 0 >
< node(3) : FollowerNode | nodeTerm : 1, log : entry(0, 0, "startup"), committedLog : entry(0, 0, "startup"), neighbors : (node(0),node(1),node(2)), isWaiting : false, nextNeighbor : node(0), quorum : 3, numNeighbors : 3, numPros : 1, numResponses : 0 >
(msg timeout(2) from client to node(1))
(msg requestAppendEntry(1, entry(0, 0, "startup")) from node(0) to node(1))
(msg requestAppendEntry(1, entry(0, 0, "startup")) from node(0) to node(2))
msg requestAppendEntry(1, entry(0, 0, "startup")) from node(0) to node(3)

AS1:AttributeSet --> nodeTerm : 1, log : entry(0, 0, "startup"), committedLog : entry(0, 0, "startup"), neighbors : (node(1),node(2),node(3)), isWaiting : true, nextNeighbor : node(1), quorum : 3, numNeighbors : 3, numPros : 1, numResponses : 0

--- node(1)이 leader가 되는지 여부 검증
search [1] test4NodeCompetition =>* C:Configuration
    < node(1) : LeaderNode | AS1:AttributeSet > .

Solution 1 (state 364)
states: 365  rewrites: 7548 in 130ms cpu (132ms real) (58061 rewrites/second)
C:Configuration --> < client : Client | none > < node(0) : FollowerNode | nodeTerm : 2, log : entry(0, 0, "startup"), committedLog : entry(0, 0, "startup"), neighbors : (node(1),node(2),node(
3)), isWaiting : false, nextNeighbor : node(1), quorum : 3, numNeighbors : 3, numPros : 1, numResponses : 0 >
< node(2) : FollowerNode | nodeTerm : 2, log : entry(0, 0, "startup"),
committedLog : entry(0, 0, "startup"), neighbors : (node(0),node(1),node(3)), isWaiting : false, nextNeighbor : node(3), quorum : 3, numNeighbors : 3, numPros : 1, numResponses : 0 >
< node(3) : FollowerNode | nodeTerm : 2, log : entry(0, 0, "startup"), committedLog : entry(0, 0, "startup"), neighbors : (node(0),node(1),node(2)), isWaiting : false, nextNeighbor : node(0), quorum : 3, numNeighbors : 3, numPros : 1, numResponses : 0 >
(msg timeout(1) from client to node(0))
(msg requestAppendEntry(2, entry(0, 0, "startup")) from node(1) to node(0))
(msg requestAppendEntry(2, entry(0, 0, "startup")) from node(1) to node(2))
msg requestAppendEntry(2, entry(0, 0, "startup")) from node(1) to node(3)

AS1:AttributeSet --> nodeTerm : 2, log : entry(0, 0, "startup"), committedLog : entry(0, 0, "startup"), neighbors : (node(0),node(2),node(3)), isWaiting : true, nextNeighbor : node(2), quorum : 3, numNeighbors : 3, numPros : 1, numResponses : 0
```
node(0), node(1)이 각각 leader가 될 수 있고, leader safety 또한 검증했습니다.

#### test4NodeLeaderKillWithQuorum
test4NodeLeaderKillWithQuorum는 node가 4개일 때 leader kill + 정족수를 만족할 때를 검증할 수 있는 모듈입니다.
```
search [1] test4NodeLeaderKillWithQuorum =>! C:Configuration .

Solution 1 (state 5102)
states: 5104  rewrites: 152844 in 5140ms cpu (5147ms real) (29736 rewrites/second)
C:Configuration -->
< client : Client | none >
< node(0) : OfflineNode | nodeTerm : 0, log : entry(0, 0, "startup"), committedLog : entry(0, 0, "startup"), neighbors : (node(1),node(2),node( 3)), isWaiting : false, nextNeighbor : node(1), quorum : 3, numNeighbors : 3, numPros : 1, numResponses : 0 >
< node(1) : LeaderNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry( 1, 1, "client-request1") ++ entry(2, 1, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), neighbors : (node( 0),node(2),node(3)), isWaiting : false, nextNeighbor : node(2), quorum : 3, numNeighbors : 3, numPros : 3, numResponses : 3 >
< node(2) : FollowerNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), neighbors : (node(0),node(1),node(3)), isWaiting : false, nextNeighbor : node(3), quorum : 3, numNeighbors : 3, numPros : 1, numResponses : 0 >
< node(3) : FollowerNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), neighbors : (node(0),node(1),node(2)), isWaiting : false, nextNeighbor : node(0), quorum : 3, numNeighbors : 3, numPros : 1, numResponses : 0 >
```

node(0)이 OfflineNode가 되었고, node(1)이 LeaderNode로 선정된 것을 볼 수 있습니다. 마찬가지로 log matching, leader safety, state machine safety를 검증할 수 있습니다.

#### test4NodeLeaderKillWithoutQuorum
test4NodeLeaderKillWithoutQuorum는 node가 4개일 때 leader kill + 정족수를 만족하지 않을 때를 검증하는 모듈입니다.
```
search [1] test4NodeLeaderKillWithoutQuorum =>! C:Configuration .
```
이 경우에는 계속 실행됩니다. `set trace on .`으로 돌아가고 있는 것을 보면, termNumber는 계속 올라가지만 CandidateNode가 충분한 수의 투표를 받지 못해 계속 새로운 투표를 요청하는 것을 검증할 수 있습니다.

#### test5Node
test5Node는 node가 5개일 때 검증하는 모듈입니다. 실행은 약 90초 정도가 걸립니다.

```
search [1] test5Node =>! C:Configuration .

Solution 1 (state 52945)
states: 52947  rewrites: 2918921 in 85520ms cpu (85527ms real) (34131 rewrites/second)
C:Configuration -->
< client : Client | none >
< node(0) : LeaderNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), neighbors : (node(1),node(2),node(3),node(4)), isWaiting : false, nextNeighbor : node(1), quorum : 3, numNeighbors : 4, numPros : 5, numResponses : 4 >
< node(1) : FollowerNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), neighbors : (node(0),node(2),node(3),node(4)), isWaiting : false, nextNeighbor : node(2), quorum : 3, numNeighbors : 4, numPros : 1, numResponses : 0 >
< node(2) : FollowerNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), neighbors : (node(0), node(1),node(3),node(4)), isWaiting : false, nextNeighbor : node(3), quorum : 3, numNeighbors : 4, numPros : 1, numResponses : 0 >
< node(3) : FollowerNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), neighbors : (node(0),node(1),node(2),node(4)), isWaiting : false, nextNeighbor : node(4), quorum : 3, numNeighbors : 4, numPros : 1, numResponses : 0 >
< node(4) : FollowerNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), neighbors : (node(0),node(1),node(2),node(3)), isWaiting : false, nextNeighbor : node(0), quorum : 3, numNeighbors : 4, numPros : 1, numResponses : 0 >
```
마찬가지로 log matching, leader safety, state machine safety를 검증할 수 있습니다.

#### test6Node
test6Node는 node가 6개일 때 검증하는 모듈입니다. 실행은 약 20분 정도가 걸립니다.
```
search [1] test6Node =>! C:Configuration .

Solution 1 (state 581317)
states: 581319  rewrites: 39999037 in 1217550ms cpu (1217611ms real) (32852 rewrites/second)
C:Configuration -->
< client : Client | none >
< node(0) : LeaderNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), neighbors : (node(1),node(2),node(3),node(4),node(5)), isWaiting : false, nextNeighbor : node(1), quorum : 4, numNeighbors : 5, numPros : 6, numResponses : 5 >
< node(1) : FollowerNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), neighbors : (node(0), node(2),node(3),node(4),node(5)), isWaiting : false, nextNeighbor : node(2), quorum : 4, numNeighbors : 5, numPros : 1, numResponses : 0 >
< node(2) : FollowerNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), neighbors : (node(0),node(1),node(3),node(4),node(5)), isWaiting : false, nextNeighbor : node(3), quorum : 4, numNeighbors : 5, numPros : 1, numResponses : 0 >
< node(3) : FollowerNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), neighbors : (node(0),node(1),node(2),node(4),node(5)), isWaiting : false, nextNeighbor : node(4), quorum : 4, numNeighbors : 5, numPros : 1, numResponses : 0 >
< node(4) : FollowerNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), neighbors : (node(0),node(1),node(2),node(3),node(5)), isWaiting : false, nextNeighbor : node(0), quorum : 4, numNeighbors : 5, numPros : 1, numResponses : 0 >
< node(5) : FollowerNode | nodeTerm : 1, log : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), committedLog : (entry(0, 0, "startup") ++ entry(1, 1, "client-request1") ++ entry(2, 1, "client-request2")), neighbors : (node(0),node(1),node(2),node(3),node(4)), isWaiting : false, nextNeighbor : node(0), quorum : 4, numNeighbors : 5, numPros : 1, numResponses : 0 >
```
마찬가지로 log matching, leader safety, state machine safety를 검증할 수 있습니다.