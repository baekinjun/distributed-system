# 🎮 분산 시스템 패턴 - 구체적 설명

## 실전 예시: 게임 아이템 거래 시스템

롤(LOL) 같은 게임의 아이템 거래 시스템을 만든다고 가정

---

## 1️⃣ 단일 서버의 한계 (1.1)

### **초기 상황**
```
서버 1대:
- CPU: 요청 처리
- 메모리: 현재 접속자 정보
- 디스크: 아이템 데이터 저장
- 네트워크: 유저와 통신

유저 1000명 → OK ✅
유저 100만명 → 서버 터짐 💥
```

### **구체적인 문제**
```java
// 서버 1대가 처리하는 상황
public void buyItem(String userId, String itemId) {
    // 1. DB에서 유저 정보 읽기 (디스크 I/O)
    User user = database.getUser(userId);
    
    // 2. 돈이 충분한지 확인 (CPU 계산)
    if (user.getMoney() >= item.getPrice()) {
        // 3. 돈 차감 (메모리 + 디스크)
        user.setMoney(user.getMoney() - item.getPrice());
        
        // 4. 아이템 지급
        user.addItem(item);
        
        // 5. DB에 저장
        database.save(user);
    }
}

// 문제:
// 동시에 10만명이 아이템 구매 → 큐잉 발생
// CPU 100% → 응답 느림
// 메모리 부족 → 서버 다운
```

---

## 2️⃣ 비즈니스 로직과 데이터 분리 (1.2)

### **해결: 2개로 분리**

```
[웹 서버 5대] ← 비저장 (유저 요청만 받음)
       ↓
[데이터베이스 서버 1대] ← 상태저장 (진짜 데이터)
```

```java
// 웹 서버 (비저장 컴포넌트)
@RestController
public class ItemController {
    @Autowired
    private ItemService itemService;
    
    @PostMapping("/buy")
    public ResponseEntity<?> buyItem(@RequestBody BuyRequest req) {
        // 여기는 계산만! 저장은 안함
        return itemService.buyItem(req.getUserId(), req.getItemId());
    }
}

// 데이터베이스 (상태저장 컴포넌트)
// - 진짜 돈, 아이템 정보 저장
// - 하지만 얘도 병목!
```

**문제: DB 서버가 병목! 📉**

---

## 3️⃣ 쓰기 전 로그 (Write-Ahead Log) (2.1)

### **구체적인 문제 상황**

```
유저 "철수"가 아이템 구매:
1. 돈 100,000원 → 90,000원 (차감)
2. 아이템 "전설의 검" 지급
3. 저장 중... 💥 서버 크래시!

결과:
- 돈은 차감됨 (메모리에는 있었음)
- 아이템은 안받음 (디스크 저장 전)
- 철수: "내 돈 어디갔어?!" 😡
```

### **해결: WAL (Write-Ahead Log)**

```java
public class ItemService {
    private WriteAheadLog wal;
    private Database db;
    
    public void buyItem(String userId, String itemId) {
        // 1. 먼저 로그에 기록! (디스크에 안전하게)
        LogEntry entry = new LogEntry(
            "BUY_ITEM",
            userId: "철수",
            itemId: "전설의 검",
            price: 10000,
            timestamp: now()
        );
        wal.append(entry); // ← 여기서 디스크에 기록!
        
        // 2. 실제 처리
        try {
            User user = db.getUser(userId);
            user.setMoney(user.getMoney() - 10000);
            user.addItem(itemId);
            db.save(user);
            
            // 3. 로그에 "완료" 표시
            wal.markComplete(entry.getId());
            
        } catch (Exception e) {
            // 서버 죽어도 로그는 남아있음!
            wal.markFailed(entry.getId());
        }
    }
    
    // 서버 재시작 시
    public void recover() {
        List<LogEntry> unfinished = wal.getUnfinishedEntries();
        for (LogEntry entry : unfinished) {
            // 미완료 작업 재실행!
            retry(entry);
        }
    }
}
```

**로그 파일 예시:**
```
[2024-11-02 14:00:01] START BUY_ITEM userId=철수 itemId=전설의검 price=10000
[2024-11-02 14:00:02] COMPLETE BUY_ITEM id=12345
[2024-11-02 14:00:03] START BUY_ITEM userId=영희 itemId=마법의방패 price=5000
💥 서버 크래시!

// 재시작 후
[2024-11-02 14:01:00] RECOVER: 영희 거래 재실행...
[2024-11-02 14:01:01] COMPLETE BUY_ITEM id=12346
```

---

## 4️⃣ 리더-팔로워 패턴 (2.2)

### **구체적인 동시성 문제**

```java
// 철수가 돈 10,000원 가지고 있음

// 서버1에서 (동시)
철수가 "전설의 검" 구매 (10,000원)

// 서버2에서 (동시)  
철수가 "마법의 방패" 구매 (10,000원)

// 둘 다 성공?! 
// 철수: 돈 10,000원으로 20,000원어치 구매?!
```

### **해결: 리더만 쓰기 가능**

```
[리더 서버] ← 모든 쓰기 요청 (구매, 판매)
    ↓ 복제
[팔로워1] ← 읽기만 (내 아이템 보기)
[팔로워2] ← 읽기만 (내 아이템 보기)
[팔로워3] ← 읽기만 (내 아이템 보기)
```

```java
public class ClusterManager {
    private Server leader;
    private List<Server> followers;
    
    public void buyItem(String userId, String itemId) {
        // 무조건 리더로만 보냄!
        if (this.isLeader()) {
            // 1. 리더가 처리
            processTransaction(userId, itemId);
            
            // 2. 팔로워들에게 복제
            LogEntry entry = createLogEntry(userId, itemId);
            for (Server follower : followers) {
                follower.replicate(entry);
            }
        } else {
            // 팔로워면 리더로 redirect
            redirectToLeader(userId, itemId);
        }
    }
    
    public User getUser(String userId) {
        // 읽기는 아무 서버나 OK
        return database.getUser(userId);
    }
}
```

---

## 5️⃣ 하트비트 & 리더 선출 (2.3)

### **구체적인 시나리오**

```
서버 구성:
- 서버A (리더)
- 서버B (팔로워)
- 서버C (팔로워)

14:00:00 - 서버A: "나 살아있어!" → B, C에게 전송
14:00:01 - 서버A: "나 살아있어!"
14:00:02 - 서버A: "나 살아있어!"
14:00:03 - 💥 서버A 다운!
14:00:04 - 서버B: "어? 하트비트 안와..."
14:00:05 - 서버B: "아직도 안와... 리더 죽었나?"
14:00:06 - 서버B & C: "선거 시작!"
```

```java
public class HeartbeatManager {
    private static final long HEARTBEAT_INTERVAL = 1000; // 1초
    private static final long TIMEOUT = 3000; // 3초
    
    private long lastHeartbeatTime;
    private boolean isLeader;
    
    // 리더의 하트비트 전송
    @Scheduled(fixedRate = HEARTBEAT_INTERVAL)
    public void sendHeartbeat() {
        if (isLeader) {
            HeartbeatMessage msg = new HeartbeatMessage(
                serverId: "서버A",
                term: 5, // 5대째 리더
                timestamp: System.currentTimeMillis()
            );
            
            for (Server follower : followers) {
                follower.receiveHeartbeat(msg);
            }
        }
    }
    
    // 팔로워의 하트비트 수신
    public void receiveHeartbeat(HeartbeatMessage msg) {
        this.lastHeartbeatTime = System.currentTimeMillis();
        System.out.println("리더 살아있음: " + msg.serverId);
    }
    
    // 타임아웃 체크
    @Scheduled(fixedRate = 1000)
    public void checkTimeout() {
        long now = System.currentTimeMillis();
        if (now - lastHeartbeatTime > TIMEOUT) {
            System.out.println("리더 죽은듯! 선거 시작!");
            startElection();
        }
    }
    
    // 선거
    private void startElection() {
        // 1. 내 표 던지기
        int votesReceived = 1; // 나 자신
        
        // 2. 다른 서버들에게 투표 요청
        for (Server peer : peers) {
            VoteRequest req = new VoteRequest(
                candidateId: myId,
                lastLogIndex: myLog.size()
            );
            
            if (peer.requestVote(req)) {
                votesReceived++;
            }
        }
        
        // 3. 과반수 얻으면 리더!
        int majority = (peers.size() + 1) / 2 + 1;
        if (votesReceived >= majority) {
            becomeLeader();
        }
    }
}
```

**실제 로그:**
```
[14:00:00] 서버A(리더): Heartbeat sent to B, C
[14:00:01] 서버B: Heartbeat received from A
[14:00:01] 서버C: Heartbeat received from A
[14:00:02] 서버A(리더): Heartbeat sent to B, C
[14:00:03] 💥 서버A 크래시
[14:00:04] 서버B: WARNING - No heartbeat for 2s
[14:00:05] 서버B: TIMEOUT - Starting election
[14:00:05] 서버C: TIMEOUT - Starting election
[14:00:06] 서버B: Requesting votes...
[14:00:06] 서버C: Voting for B
[14:00:07] 서버B: Elected as new leader! (2/2 votes)
[14:00:07] 서버B(리더): Sending first heartbeat
```

---

## 6️⃣ 세대 시계 (Generation Clock) (2.4)

### **진짜 문제 상황**

```
타임라인:

10:00 - 서버A(1대 리더): 철수 돈 = 50,000원
10:01 - 💥 서버A 네트워크 끊김!
10:02 - 서버B가 2대 리더로 선출
10:03 - 서버B(2대 리더): 철수가 10,000원 사용 → 40,000원
10:04 - 서버A 네트워크 복구! (아직 자기가 리더인줄 앎)
10:05 - 서버A(1대 리더): 철수가 20,000원 사용 → 30,000원

철수 돈이 30,000원? 40,000원? 🤔
```

```java
public class GenerationClock {
    private int generation; // 세대 번호
    
    // 리더 선출될 때마다 증가
    public void electNewLeader() {
        this.generation++;
        System.out.println("새 리더 선출! 세대: " + generation);
    }
    
    // 모든 요청에 세대 번호 포함
    public class Request {
        int generation;
        String operation;
        Object data;
    }
    
    // 요청 처리
    public void handleRequest(Request req) {
        if (req.generation < this.generation) {
            // 옛날 리더의 요청 → 거절!
            System.out.println("거절: 낮은 세대 " + req.generation);
            return;
        }
        
        if (req.generation > this.generation) {
            // 내가 뒤처짐! → 내 세대 업데이트
            this.generation = req.generation;
            stepDown(); // 리더 그만둠
        }
        
        // 같은 세대면 처리
        process(req);
    }
}
```

**구체적 예시:**
```java
// 10:00 - 서버A (1대 리더)
Request req1 = new Request(
    generation: 1,
    operation: "UPDATE",
    data: "철수 돈 = 50,000원"
);
서버A.handle(req1); ✅

// 10:03 - 서버B (2대 리더)  
Request req2 = new Request(
    generation: 2,
    operation: "UPDATE", 
    data: "철수 돈 = 40,000원"
);
서버B.handle(req2); ✅

// 10:05 - 서버A (아직 1대라고 생각)
Request req3 = new Request(
    generation: 1, // ← 낮음!
    operation: "UPDATE",
    data: "철수 돈 = 30,000원" 
);
서버B.handle(req3); ❌ 거절!
// "세대 1은 옛날 거야! 지금은 세대 2야!"

// 서버A가 깨달음
서버A.generation = 2; // 업데이트
서버A.stepDown(); // "나 리더 아니구나..."
```

---

## 7️⃣ 과반수 정족수 (Quorum) (2.5)

### **왜 필요한가?**

```
서버 5대:
A(리더), B, C, D, E

철수가 아이템 구매:
A: "돈 차감! 로그에 기록!"
A → B: "이거 복제해!" (OK)
A → C: "이거 복제해!" (OK)  
A → D: "이거 복제해!" (네트워크 느림...)
A → E: "이거 복제해!" (응답 없음...)

A는 언제 철수에게 "구매 완료!"라고 말해야 할까?
- 5대 모두 응답? → 너무 오래 걸림
- 1대만 응답? → 나중에 데이터 유실 위험

답: 과반수 (3대)!
```

```java
public class QuorumManager {
    private List<Server> cluster;
    private int quorumSize;
    
    public QuorumManager(List<Server> cluster) {
        this.cluster = cluster;
        // 과반수 = (전체 / 2) + 1
        this.quorumSize = (cluster.size() / 2) + 1;
    }
    
    public boolean replicateLog(LogEntry entry) {
        // 1. 리더 자신은 이미 기록 (1표)
        int ackCount = 1;
        
        // 2. 팔로워들에게 복제 요청
        List<Future<Boolean>> futures = new ArrayList<>();
        for (Server follower : followers) {
            Future<Boolean> future = executor.submit(() -> {
                return follower.appendLog(entry);
            });
            futures.add(future);
        }
        
        // 3. 응답 기다리기 (타임아웃 1초)
        for (Future<Boolean> future : futures) {
            try {
                if (future.get(1, TimeUnit.SECONDS)) {
                    ackCount++;
                }
            } catch (TimeoutException e) {
                // 응답 안옴
            }
            
            // 과반수 달성하면 바로 리턴!
            if (ackCount >= quorumSize) {
                entry.setCommitted(true);
                return true; // ← 철수에게 "성공!" 응답
            }
        }
        
        // 과반수 실패
        return false;
    }
}
```

**실시간 로그:**
```
[14:00:00.000] Leader A: 철수 구매 요청 받음
[14:00:00.001] Leader A: 로그 추가 (index=100)
[14:00:00.002] Leader A: 팔로워들에게 복제 요청...
[14:00:00.050] Follower B: 로그 복제 완료 (1/3)
[14:00:00.051] Follower C: 로그 복제 완료 (2/3)
[14:00:00.052] 과반수 달성! (A+B+C = 3/5)
[14:00:00.053] Leader A → 철수: "구매 완료!" ✅
[14:00:00.500] Follower D: 로그 복제 완료 (늦음)
[14:00:01.000] Follower E: 타임아웃 (응답 없음)

// 결과: D, E 늦어도 OK! 이미 커밋됨!
```

---

## 8️⃣ 하이 워터마크 (High-Water Mark) (2.6)

### **팔로워는 언제 커밋?**

```
상황:
Leader A: [log1] [log2] [log3] [log4] [log5]
          커밋됨 -----> HWM(5)

Follower B: [log1] [log2] [log3] [log4] [?]
            어디까지 커밋해야 하지?

Follower C: [log1] [log2] [?] [?] [?]
            나는 2까지만 받았는데...
```

```java
public class HighWaterMarkManager {
    private int highWaterMark = 0; // 확정된 로그 인덱스
    private List<LogEntry> log = new ArrayList<>();
    
    // 리더: 과반수 복제되면 HWM 업데이트
    public void updateHighWaterMark() {
        // 각 팔로워가 어디까지 받았는지 확인
        List<Integer> replicatedIndexes = new ArrayList<>();
        for (Follower f : followers) {
            replicatedIndexes.add(f.getLastLogIndex());
        }
        Collections.sort(replicatedIndexes);
        
        // 과반수 위치 = HWM
        int quorumIndex = replicatedIndexes.get(quorumSize - 1);
        
        if (quorumIndex > highWaterMark) {
            highWaterMark = quorumIndex;
            System.out.println("HWM 업데이트: " + highWaterMark);
        }
    }
    
    // 하트비트에 HWM 포함
    public void sendHeartbeat() {
        HeartbeatMessage msg = new HeartbeatMessage(
            leaderId: myId,
            highWaterMark: this.highWaterMark // ← 여기!
        );
        
        for (Follower f : followers) {
            f.receiveHeartbeat(msg);
        }
    }
    
    // 팔로워: HWM까지 커밋
    public void onHeartbeat(HeartbeatMessage msg) {
        int leaderHWM = msg.highWaterMark;
        
        // 리더의 HWM까지 모든 로그 커밋
        for (int i = commitIndex + 1; i <= leaderHWM; i++) {
            if (i < log.size()) {
                commitLog(log.get(i));
                commitIndex = i;
            }
        }
    }
}
```

**구체적 시나리오:**
```
T=0: 
Leader: [1][2][3][4][5] HWM=5 (과반수 확인됨)
FolA:   [1][2][3][4][5] commit=2 (아직 커밋 안함)
FolB:   [1][2][3]       commit=2
FolC:   [1][2][3][4]    commit=2

T=1: Leader가 하트비트 전송 (HWM=5 포함)
Leader → All: "HWM은 5야!"

T=2: 팔로워들 HWM 받음
FolA: "오! 5까지 커밋해야겠다"
      [1][2][3][4][5] commit=5 ✅
      
FolB: "어? 나 3까지밖에 없는데..."
      [1][2][3] commit=3 (있는 것까지만)
      "4, 5 달라고 요청해야지"
      
FolC: "4까지 커밋!"
      [1][2][3][4] commit=4 ✅
      "5번 달라고 요청!"
```

---

## 9️⃣ 단일 갱신 큐 (2.7)

### **동시성 문제**

```java
// 문제 상황
public class SlowItemService {
    public void buyItem(String userId, String itemId) {
        // 1. 로그에 쓰기 (느림! 디스크 I/O)
        writeToLog(entry); // 100ms
        
        // 2. 다른 서버에 복제 (느림! 네트워크)
        replicateToFollowers(entry); // 50ms
        
        // 3. 과반수 대기 (느림!)
        waitForQuorum(); // 50ms
        
        // 총 200ms...
        // 초당 5개밖에 못처리!
    }
}

// 유저 1000명이 동시에 구매하면?
// 1000 * 200ms = 200초 = 3분 이상! 😱
```

### **해결: 비동기 큐**

```java
public class AsyncItemService {
    private BlockingQueue<PendingRequest> requestQueue;
    private Map<String, CompletableFuture<Response>> pendingRequests;
    
    // 1. 요청 받으면 바로 응답 (빠름!)
    public CompletableFuture<Response> buyItem(String userId, String itemId) {
        String requestId = UUID.randomUUID().toString();
        
        // 콜백 등록
        CompletableFuture<Response> future = new CompletableFuture<>();
        pendingRequests.put(requestId, future);
        
        // 큐에 넣기
        PendingRequest req = new PendingRequest(
            id: requestId,
            userId: userId,
            itemId: itemId
        );
        requestQueue.offer(req);
        
        return future; // ← 바로 리턴! (1ms)
    }
    
    // 2. 백그라운드 워커가 처리
    @Async
    public void processQueue() {
        while (true) {
            PendingRequest req = requestQueue.take();
            
            try {
                // 실제 처리 (느림)
                LogEntry entry = createLogEntry(req);
                writeToLog(entry);
                replicateToFollowers(entry);
                waitForQuorum();
                
                // 완료! 콜백 호출
                CompletableFuture<Response> future = 
                    pendingRequests.remove(req.id);
                future.complete(new Response("SUCCESS"));
                
            } catch (Exception e) {
                future.completeExceptionally(e);
            }
        }
    }
}

// 사용
CompletableFuture<Response> future = service.buyItem("철수", "전설의검");
// 여기서 바로 다른 일 할 수 있음!

future.thenAccept(response -> {
    System.out.println("구매 완료!");
});
```

**성능 비교:**
```
동기 방식:
요청1 → 처리 (200ms) → 응답
요청2 → 처리 (200ms) → 응답  
요청3 → 처리 (200ms) → 응답
총 600ms

비동기 방식:
요청1 → 큐에 넣기 (1ms) → 응답
요청2 → 큐에 넣기 (1ms) → 응답
요청3 → 큐에 넣기 (1ms) → 응답
총 3ms! (백그라운드에서 처리)
```

---

## 🔟 멱등 수신자 (Idempotent Receiver) (2.7)

### **중복 요청 문제**

```
철수가 아이템 구매:
1. 클라이언트 → 서버: "아이템 사줘!"
2. 서버 처리 완료
3. 서버 → 클라이언트: "완료!" (응답 전송)
4. 💥 네트워크 끊김! 클라이언트가 응답 못받음
5. 클라이언트: "응답 안와... 재시도!"
6. 클라이언트 → 서버: "아이템 사줘!" (똑같은 요청)
7. 서버: "또 샀네!" → 아이템 2개 줌! 😱
```

```java
public class IdempotentItemService {
    // 클라이언트별 처리된 요청 저장
    private Map<String, Map<Long, Response>> processedRequests;
    
    public Response buyItem(BuyRequest request) {
        String clientId = request.getClientId();
        long requestId = request.getRequestId();
        
        // 1. 이미 처리한 요청인지 확인
        if (processedRequests.containsKey(clientId)) {
            Response cached = processedRequests
                .get(clientId)
                .get(requestId);
                
            if (cached != null) {
                System.out.println("중복 요청! 캐시된 응답 반환");
                return cached; // ← 다시 처리 안함!
            }
        }
        
        // 2. 첫 요청이면 처리
        Response response = actuallyBuyItem(request);
        
        // 3. 응답 저장
        processedRequests
            .computeIfAbsent(clientId, k -> new HashMap<>())
            .put(requestId, response);
            
        return response;
    }
}

// 클라이언트 측
public class ItemClient {
    private String clientId = UUID.randomUUID().toString();
    private AtomicLong requestCounter = new AtomicLong(0);
    
    public void buyItem(String itemId) {
        long requestId = requestCounter.incrementAndGet();
        
        BuyRequest req = new BuyRequest(
            clientId: this.clientId,    // ← 클라이언트 ID
            requestId: requestId,        // ← 요청 번호
            itemId: itemId
        );
        
        // 재시도 로직
        int maxRetries = 3;
        for (int i = 0; i < maxRetries; i++) {
            try {
                Response res = server.buyItem(req);
                return; // 성공!
            } catch (TimeoutException e) {
                // 재시도 (같은 requestId!)
            }
        }
    }
}
```

**실제 로그:**
```
[14:00:00] 클라이언트 "client-123" 요청 #1 전송
[14:00:00] 서버: 요청 #1 처리 시작
[14:00:01] 서버: 아이템 지급 완료
[14:00:01] 서버: 응답 저장 (client-123, req#1 → "SUCCESS")
[14:00:01] 서버 → 클라이언트: "SUCCESS"
[14:00:02] 💥 네트워크 에러! 클라이언트 응답 못받음
[14:00:03] 클라이언트: 타임아웃! 재시도!
[14:00:03] 클라이언트 → 서버: 요청 #1 재전송 (같은 번호!)
[14:00:03] 서버: "어? #1은 이미 처리했는데?"
[14:00:03] 서버: 캐시에서 응답 찾음 → "SUCCESS"
[14:00:03] 서버 → 클라이언트: "SUCCESS" (다시 처리 안함!)
[14:00:03] 클라이언트: 응답 받음! ✅
```

---

## 📚 전체 흐름 요약

```
1. 서버 1대 → 한계 도달
   └─ 해결: 여러 대로 분산

2. 분산 시 문제들:
   ├─ 서버 죽음 
   │  └─ 해결: 쓰기 전 로그 (WAL)
   │
   ├─ 동시 갱신 충돌
   │  └─ 해결: 리더-팔로워 패턴
   │
   ├─ 리더 죽음
   │  └─ 해결: 하트비트 + 선거
   │
   ├─ 여러 리더 혼란
   │  └─ 해결: 세대 시계
   │
   ├─ 데이터 확정 시점
   │  └─ 해결: 과반수 정족수
   │
   ├─ 팔로워 동기화
   │  └─ 해결: 하이 워터마크
   │
   ├─ 성능 저하
   │  └─ 해결: 비동기 큐
   │
   └─ 중복 요청
      └─ 해결: 멱등 수신자
```

---

## 🎯 핵심 원칙

1. **내구성 우선**: 먼저 로그에 기록하라
2. **단일 진실 공급원**: 리더만 쓰기
3. **장애 감지**: 하트비트로 살아있음 확인
4. **버전 관리**: 세대로 충돌 해결
5. **과반수 합의**: 절반 이상 동의하면 확정
6. **명시적 동기화**: 하이 워터마크로 커밋 시점 전달
7. **비동기 처리**: 빠른 응답 + 백그라운드 작업
8. **멱등성**: 같은 요청 여러 번 처리해도 같은 결과

---

이 패턴들은 Kafka, Cassandra, MongoDB, Raft 등 실제 분산 시스템에서 모두 사용됩니다! 🚀