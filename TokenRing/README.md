# Token Ring with Maude
Designing Reliables Distributes Systems, 225p

### Token Ring
In the “token ring” mutual exclusion algorithm, the nodes logically form a “ring” structure, as shown in Figure 13.1 where a node only knows the next node in this ring.

The algorithm works as follows: there is one “token,” and only the node that holds the token may enter its critical section. The node then holds on to the token during its execution in the critical section, and passes the token to the next node in the ring when it exits its critical section. If a node that is not waiting to enter its critical section receives the token, it just passes the token to the next node.

Token Ring을 모델링하고 검증합니다.

<br>

## 검증
### Normal State 검증
아래 명령어는 특정 2개의 node의 state가 normal인 상태를 찾는 명령어입니다.
```
search [1] init =>* C:Configuration
    < O1:Oid : Node | state : normal, next : O1N:Oid >
    < O2:Oid : Node | state : normal, next : O2N:Oid > .
```

실행 결과, Solution이 나옵니다. 즉, 2개의 node가 normal인 state가 존재함을 알 수 있습니다.
```
Solution 1 (state 10)
states: 11  rewrites: 11 in 0ms cpu (1ms real) (~ rewrites/second)
C:Configuration --> < c : Node | state : normal, next : d > < d : Node | state : normal, next : a >
O1:Oid --> b
O1N:Oid --> c
O2:Oid --> a
O2N:Oid --> b
```

<br>

### Mutual Exclusion
아래 명령어는 2개의 node가 acquire인 상태를 찾는 명령어입니다.
```
search [1] init =>* C:Configuration < C:Oid : Node | state : acquire > < C:Oid : Node | state : acquire > .
```

실행 결과, No Solution이 나옵니다. 한 번에 2개 이상의 node가 acquire하는 state가 존재하지 않음을 알 수 있습니다.
```
No solution.
states: 432  rewrites: 865 in 120ms cpu (168ms real) (7208 rewrites/second)
```

<br>

### Critical Section 진입
아래 명령어는 특정 node의 state가 acquire이 되는 상태를 찾는 명령어입니다.
```
search [1] init =>* C:Configuration < C:Oid : Node | state : acquire, next : N:Oid > .
search [1] init =>* C:Configuration < a : Node | state : acquire, next : b > .
search [1] init =>* C:Configuration < b : Node | state : acquire, next : c > .
search [1] init =>* C:Configuration < c : Node | state : acquire, next : d > .
search [1] init =>* C:Configuration < d : Node | state : acquire, next : a > .
```

실행 결과, solution이 나옵니다. 즉, node가 token을 acquire하는 state가 존재함을 알 수 있습니다.
```
Solution 1 (state 10)
states: 11  rewrites: 11 in 0ms cpu (1ms real) (~ rewrites/second)
C:Configuration --> < b : Node | state : normal, next : c > < c : Node | state : normal, next : d > < d : Node | state : normal, next : a >
C:Oid --> a
N:Oid --> b
```