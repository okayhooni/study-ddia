## Chapter 5: Replication

### 멀티 리더 레플리케이션(Multi-Leader Replication)
- 싱글 리더 레플리케이션의 단점: 모든 쓰기(write) 작업은 반드시 "단일" 리더를 거쳐야 하므로, 싱글 리더가 SPOF 가 될 수 있음
- 이러한 제약에 대한 대안으로서, 멀티 리더 레플리케이션 전략이 쓰일 수 있음
- multi-leader configuration = "master-master" replication = "active/active" replication
- 각 리더는, 리더인 동시에 다른 리더들에 대한 팔로어 역할도 함

#### 멀티-리더 레플리케이션의 유즈케이스
- Multi-datacenter operation
  - 데이터센터를 여러 곳에 분산으로 둘 경우, 기존의 싱글 리더 전략을 취할 시에 여러 데이터 센터 중에 한 곳에 리더를 두어야 해서, 모든 write 작업이 해당 데이터센터로 집중됨, 따라서 각 데이터 센터마다 별도의 리더를 두는 전략을 취할 수 있음
  - 각 데이터 센터 간의 싱크 이슈가 더해져 컨플릭트 해결(Conflict resolution)이 복잡해지는 단점이 있음
  - 여러 이슈(autoincrementing keys, triggers, integrity constraints)가 발생 가능하여 멀티-리더 레플리케이션은 가급적 지양되는 편
- Clients with offline operation
  - 캘린더 앱이 예시
    - 캘린더 앱을 이용하는 각각의 디바이스는 온/오프라인에 상관없이 일정을 읽고/쓸수 있어야 함
    - 오프라인 상태에서의 업데이트 내역도, 해당 디바이스가 온라인이 되면 싱크되어야 함
  - 이 경우, 각각의 디바이스는 쓰기(write)가 가능한 리더 역할을 수행하는 로컬 데이스베이스를 가져야 함
  - 캘린더 앱이 설치된 모든 디바이스 사이에 비동기(asynchronous) 멀티 리더 레플리케이션 프로세스(싱크)가 필요함
  - 이때 replication lag은 인터넷 접근 주기에 따라 수시간에서 수일이 될 수도 있음
- Collaborative editing
  - Google Docs와 같은 Real-time collaborative editing application이 예시
  - lock과 transaction에 기반한 싱글 리더 전략으로 구현시 반응성이 좋지 않음
  - 각 사용자의 브라우저 또는 클라이언트 앱이 리더로 동작하되, 각 변화의 단위를 작게(single keystroke) 하고 lock을 사용하지 않으면서, conflict resolution을 수행하는 방식으로 구현됨
    
#### 쓰기 충돌의 해결
- Synchronous versus asynchronous conflict detection
- Conflict avoidance! 최대한 피하자!
- Converging toward a consistent state (Eventually Consistency)
  - Give each write a unique ID / LWW(last write wins) : 데드락을 회피하기 위해 락 사이의 global order 를 두는 것과 유사
  - Give each replica a unique ID
  - merge/concatenate values together
  - record conflicts and handle it on application code level (Custom conflict resolution logic)
- What is a conflict?
