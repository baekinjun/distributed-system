# 🎮 분산 시스템 패턴 Week 2 - 구체적 설명

## 실전 예시: 게임 서버 클러스터 시스템

---

# 📚 3장: 쓰기 전 로그 (Write-Ahead Log, WAL)

## 💡 핵심 개념

**문제:** 서버가 갑자기 죽으면 메모리에만 있던 데이터가 날아간다!

**해결:** 모든 변경사항을 먼저 디스크에 기록하자!

---

## 🔍 구체적인 시나리오

### **상황: 게임 아이템 거래**

```
유저 "철수"가 "전설의 검" 구매:

❌ 쓰기 전 로그 없이:
1. 메모리에 "철수 돈 -10,000원" 저장
2. 메모리에 "철수 아이템 +전설의 검" 저장
3. 💥 서버 크래시!
4. 재시작 → 메모리 날아감 → 철수 돈만 없어짐!

✅ 쓰기 전 로그 사용:
1. 디스크 로그에 기록: "철수가 전설의 검 구매 시작"
2. 메모리 갱신
3. 💥 서버 크래시!
4. 재시작 → 로그 읽음 → "아! 미완료 거래가 있네?" → 재실행!
```

---

## 💻 실제 구현 예시

```java
public class WriteAheadLog {
    private File logFile;
    private FileWriter writer;
    
    // 1. 로그 엔트리 작성
    public long append(LogEntry entry) {
        // 먼저 로그 파일에 쓰기!
        long entryId = generateId();
        
        String logLine = String.format(
            "[%d] START %s userId=%s itemId=%s price=%d\n",
            entryId,
            entry.getOperation(),
            entry.getUserId(),
            entry.getItemId(),
            entry.getPrice()
        );
        
        writer.write(logLine);
        writer.flush(); // ← 중요! 실제로 디스크에 쓰기
        
        return entryId;
    }
    
    // 2. 실제 작업 수행
    public void execute(long entryId, LogEntry entry) {
        try {
            // 실제 비즈니스 로직 실행
            User user = db.getUser(entry.getUserId());
            user.subtractMoney(entry.getPrice());
            user.addItem(entry.getItemId());
            db.save(user);
            
            // 성공! 로그에 표시
            markComplete(entryId);
            
        } catch (Exception e) {
            // 실패! 로그에 표시
            markFailed(entryId);
        }
    }
    
    // 3. 서버 재시작 시 복구
    public void recover() {
        List<LogEntry> unfinished = findUnfinishedEntries();
        
        for (LogEntry entry : unfinished) {
            System.out.println("미완료 작업 발견! 재실행: " + entry);
            execute(entry.getId(), entry);
        }
    }
}
```

**실제 로그 파일:**
```
[1001] START BUY_ITEM userId=철수 itemId=전설의검 price=10000
[1001] COMPLETE
[1002] START BUY_ITEM userId=영희 itemId=마법방패 price=5000
💥 크래시!

--- 재시작 후 ---
[RECOVER] 1002번 미완료 발견, 재실행 시작
[1002] COMPLETE
```

---

## ⚙️ 구현시 고려사항

### **1. 플러시 (Flush) 성능 문제**

```java
// ❌ 나쁜 예: 매번 플러시 → 너무 느림
public void append(LogEntry entry) {
    writer.write(entry.toString());
    writer.flush(); // 100ms
}
// 초당 10개만 처리 가능!

// ✅ 좋은 예: 배치 플러시
public void appendBatch(List<LogEntry> entries) {
    for (LogEntry entry : entries) {
        writer.write(entry.toString());
    }
    writer.flush(); // 한 번만!
}
// 초당 1000개 처리 가능!
```

### **2. 데이터 무결성 검사 (CRC)**

```java
public class LogEntry {
    private String data;
    private long crc; // 체크섬
    
    public void write() {
        // CRC 계산
        this.crc = calculateCRC(data);
        
        // 로그 형식: [데이터][CRC]
        String logLine = data + "|CRC:" + crc;
        writeToFile(logLine);
    }
    
    public boolean read() {
        String[] parts = readFromFile().split("\\|CRC:");
        String data = parts[0];
        long savedCrc = Long.parseLong(parts[1]);
        
        // 데이터 손상 확인
        long calculatedCrc = calculateCRC(data);
        if (calculatedCrc != savedCrc) {
            System.err.println("로그 손상 감지!");
            return false;
        }
        
        return true;
    }
}
```

**로그 손상 예시:**
```
정상: [1001] BUY_ITEM user=철수 |CRC:4567890123
손상: [1001] BUY_��EM user=철수 |CRC:4567890123
      ↓
CRC 불일치 → 해당 엔트리 무시
```

### **3. 중복 방지**

```java
public class DeduplicationManager {
    private Set<String> processedRequests = new HashSet<>();
    
    public void processLog(LogEntry entry) {
        String requestId = entry.getRequestId();
        
        // 이미 처리했나?
        if (processedRequests.contains(requestId)) {
            System.out.println("중복 로그 무시: " + requestId);
            return;
        }
        
        // 실제 처리
        execute(entry);
        processedRequests.add(requestId);
    }
}
```

---

# 📚 4장: 분할 로그 (Segmented Log)

## 💡 핵심 개념

**문제:** 로그 파일이 100GB가 넘어가면?
- 서버 재시작 시 100GB 읽는데 10분 걸림!
- 오래된 로그 지우기도 어려움

**해결:** 로그를 여러 파일로 쪼개자!

---

## 🔍 구체적인 시나리오

```
단일 로그 파일 (나쁜 예):
game_log.log (100GB)
├─ [0-9,999,999] 모든 로그
└─ 너무 커서 관리 불가!

분할 로그 (좋은 예):
game_log_0000000.log (100MB) ← 가장 오래됨
game_log_0100000.log (100MB)
game_log_0200000.log (100MB)
...
game_log_9900000.log (100MB) ← 최신

장점:
✅ 오래된 파일만 삭제 가능
✅ 빠른 검색 (필요한 파일만 읽기)
✅ 병렬 처리 가능
```

---

## 💻 실제 구현

```java
public class SegmentedLog {
    private static final long SEGMENT_SIZE = 100 * 1024 * 1024; // 100MB
    private List<LogSegment> segments = new ArrayList<>();
    private LogSegment activeSegment;
    
    // 1. 로그 추가 (자동 롤링)
    public void append(LogEntry entry) {
        // 현재 세그먼트가 꽉 찼나?
        if (activeSegment.size() >= SEGMENT_SIZE) {
            rollover(); // 새 세그먼트 생성!
        }
        
        activeSegment.append(entry);
    }
    
    // 2. 새 세그먼트로 전환
    private void rollover() {
        // 현재 세그먼트 닫기
        activeSegment.close();
        
        // 새 세그먼트 시작
        long baseOffset = activeSegment.getLastOffset() + 1;
        String filename = String.format("game_log_%010d.log", baseOffset);
        
        activeSegment = new LogSegment(filename, baseOffset);
        segments.add(activeSegment);
        
        System.out.println("새 세그먼트 생성: " + filename);
    }
    
    // 3. 특정 오프셋 읽기
    public LogEntry read(long offset) {
        // 어느 세그먼트에 있을까?
        LogSegment segment = findSegment(offset);
        
        if (segment == null) {
            throw new IllegalArgumentException("오프셋 없음: " + offset);
        }
        
        return segment.read(offset);
    }
    
    // 4. 세그먼트 찾기
    private LogSegment findSegment(long offset) {
        for (LogSegment segment : segments) {
            if (segment.getBaseOffset() <= offset && 
                offset <= segment.getLastOffset()) {
                return segment;
            }
        }
        return null;
    }
}

public class LogSegment {
    private String filename;
    private long baseOffset; // 이 세그먼트의 시작 오프셋
    private long currentOffset;
    
    public LogSegment(String filename, long baseOffset) {
        this.filename = filename;
        this.baseOffset = baseOffset;
        this.currentOffset = baseOffset;
    }
    
    public void append(LogEntry entry) {
        entry.setOffset(currentOffset++);
        writeToFile(entry);
    }
}
```

**파일 구조 예시:**
```
/var/game-server/logs/
├── game_log_0000000.log (100MB) [offset 0 ~ 99,999]
├── game_log_0100000.log (100MB) [offset 100,000 ~ 199,999]
├── game_log_0200000.log (100MB) [offset 200,000 ~ 299,999]
└── game_log_0300000.log (50MB)  [offset 300,000 ~ 350,000] ← 현재 활성

오프셋 150,000 읽기 요청:
→ game_log_0100000.log 파일만 열면 됨!
```

**성능 비교:**
```
단일 파일:
- 오프셋 150,000 찾기 → 100GB 파일 스캔 → 10초

분할 파일:
- 오프셋 150,000 찾기 → game_log_0100000.log (100MB)만 스캔 → 0.01초
```

---

# 📚 5장: 로우 워터마크 (Low Water Mark)

## 💡 핵심 개념

**문제:** 로그가 무한히 쌓이면 디스크가 가득 참!

**해결:** "여기까지는 안전하게 지워도 돼!" 표시를 하자!

---

## 🔍 구체적인 시나리오

### **스냅샷 기반 로우 워터마크**

```
게임 서버 상태:
- 유저 데이터
- 아이템 소유 현황
- 길드 정보

로그:
[0] 철수 회원가입
[1] 철수 돈 +1000
[2] 철수 아이템 구매
[3] 영희 회원가입
...
[10000] 철수 레벨업
[10001] 스냅샷 생성! ← 현재 상태 저장

로우 워터마크 = 10001

이제 0~10000번 로그는 필요 없음!
왜? 스냅샷에 최종 상태가 있으니까!
```

---

## 💻 실제 구현

### **1. 스냅샷 기반 (Raft, Zookeeper 방식)**

```java
public class SnapshotBasedLogCleaner {
    private SegmentedLog log;
    private StateStorage stateStorage;
    private long lowWaterMark = 0;
    
    // 1. 주기적으로 스냅샷 생성
    @Scheduled(fixedRate = 3600000) // 1시간마다
    public void createSnapshot() {
        // 현재 로그 인덱스
        long currentIndex = log.getLastOffset();
        
        // 현재 상태를 디스크에 저장
        Snapshot snapshot = new Snapshot();
        snapshot.setData(stateStorage.exportAll());
        snapshot.setLastIncludedIndex(currentIndex);
        snapshot.save();
        
        System.out.println("스냅샷 생성: " + currentIndex);
        
        // 로우 워터마크 업데이트
        updateLowWaterMark(currentIndex);
    }
    
    // 2. 오래된 로그 삭제
    private void updateLowWaterMark(long newMark) {
        this.lowWaterMark = newMark;
        
        // 로우 워터마크 이전 로그 파일 삭제
        List<LogSegment> oldSegments = 
            log.getSegmentsBefore(lowWaterMark);
            
        for (LogSegment segment : oldSegments) {
            System.out.println("삭제: " + segment.getFilename());
            segment.delete();
        }
    }
    
    // 3. 서버 재시작 시 복구
    public void recover() {
        // 최신 스냅샷 로드
        Snapshot snapshot = Snapshot.loadLatest();
        stateStorage.importAll(snapshot.getData());
        
        // 스냅샷 이후의 로그만 재생
        long startOffset = snapshot.getLastIncludedIndex() + 1;
        log.replayFrom(startOffset);
    }
}
```

**타임라인 예시:**
```
T=0: 서버 시작
[0-10000] 로그 누적 (10GB)

T=1시간: 스냅샷 생성
스냅샷: 현재 상태 저장
로우 워터마크 = 10000
[0-10000] 로그 삭제 ✅

[10001-20000] 새 로그 누적

T=2시간: 스냅샷 생성
스냅샷: 현재 상태 저장
로우 워터마크 = 20000
[10001-20000] 로그 삭제 ✅

결과: 항상 최근 로그만 유지!
```

### **2. 시간 기반 (Kafka 방식)**

```java
public class TimeBasedLogCleaner {
    private SegmentedLog log;
    private static final long RETENTION_MS = 7 * 24 * 3600 * 1000; // 7일
    
    @Scheduled(fixedRate = 3600000) // 1시간마다
    public void cleanOldLogs() {
        long now = System.currentTimeMillis();
        long cutoffTime = now - RETENTION_MS;
        
        List<LogSegment> segments = log.getAllSegments();
        
        for (LogSegment segment : segments) {
            // 세그먼트의 최신 엔트리 시간
            long lastModified = segment.getLastModifiedTime();
            
            if (lastModified < cutoffTime) {
                System.out.println(
                    "7일 지난 로그 삭제: " + segment.getFilename()
                );
                segment.delete();
            }
        }
    }
}
```

**시간 기반 예시:**
```
11월 1일: game_log_0000000.log 생성
11월 2일: game_log_0100000.log 생성
11월 3일: game_log_0200000.log 생성
...
11월 8일: 정리 작업 실행
          ↓
11월 1일 파일 (7일 지남) → 삭제 ✅
11월 2일 파일 (6일) → 유지
11월 3일 파일 (5일) → 유지
```

---

# 📚 6장: 리더-팔로워 (Leader-Follower)

## 💡 핵심 개념

**문제:** 서버 3대가 각자 다른 데이터를 가지면 혼란!

**해결:** 리더 1명만 결정하고, 나머지는 복사만 하자!

---

## 🔍 구체적인 시나리오

```
❌ 리더 없이 (혼란):
서버A: 철수 돈 = 5000원
서버B: 철수 돈 = 7000원
서버C: 철수 돈 = 3000원
→ 뭐가 맞는 거야?!

✅ 리더-팔로워:
리더(A): 철수 돈 = 5000원 (유일한 진실!)
    ↓ 복제
팔로워B: 철수 돈 = 5000원 (복사본)
팔로워C: 철수 돈 = 5000원 (복사본)
→ 모두 동일!
```

---

## 💻 실제 구현

```java
public class ClusterNode {
    private enum State {
        FOLLOWER,    // 팔로워
        CANDIDATE,   // 후보자 (선거 중)
        LEADER       // 리더
    }
    
    private State state = State.FOLLOWER;
    private String leaderId = null;
    private int currentTerm = 0; // 세대 번호
    private ReplicationLog log;
    private List<ClusterNode> peers;
    
    // 1. 클라이언트 요청 처리
    public Response handleRequest(Request request) {
        if (state == State.LEADER) {
            // 리더만 쓰기 처리
            return processWrite(request);
        } else {
            // 팔로워는 리더로 redirect
            return redirectToLeader(request);
        }
    }
    
    // 2. 리더의 쓰기 처리
    private Response processWrite(Request request) {
        // 로그에 추가
        LogEntry entry = new LogEntry(
            term: currentTerm,
            data: request.getData()
        );
        log.append(entry);
        
        // 팔로워들에게 복제
        int ackCount = 1; // 나 자신
        for (ClusterNode follower : peers) {
            if (follower.replicate(entry)) {
                ackCount++;
            }
        }
        
        // 과반수 확인
        int quorum = (peers.size() + 1) / 2 + 1;
        if (ackCount >= quorum) {
            entry.setCommitted(true);
            return Response.success();
        }
        
        return Response.fail("정족수 미달");
    }
    
    // 3. 팔로워의 복제 처리
    public boolean replicate(LogEntry entry) {
        // 세대 확인
        if (entry.getTerm() < currentTerm) {
            return false; // 옛날 리더의 요청
        }
        
        // 로그에 추가
        log.append(entry);
        return true;
    }
}
```

---

## 🗳️ 리더 선출 과정

```java
public class LeaderElection {
    private int votesReceived = 0;
    private int currentTerm = 0;
    
    // 1. 하트비트 타임아웃 → 선거 시작
    public void onHeartbeatTimeout() {
        startElection();
    }
    
    // 2. 선거 시작
    private void startElection() {
        // 상태 변경
        state = State.CANDIDATE;
        currentTerm++; // 세대 증가
        votesReceived = 1; // 나 자신에게 투표
        
        System.out.println("선거 시작! Term: " + currentTerm);
        
        // 다른 서버들에게 투표 요청
        for (ClusterNode peer : peers) {
            VoteRequest request = new VoteRequest(
                term: currentTerm,
                candidateId: myId,
                lastLogIndex: log.getLastIndex(),
                lastLogTerm: log.getLastTerm()
            );
            
            if (peer.requestVote(request)) {
                votesReceived++;
            }
        }
        
        // 과반수 득표?
        int quorum = (peers.size() + 1) / 2 + 1;
        if (votesReceived >= quorum) {
            becomeLeader();
        }
    }
    
    // 3. 투표 요청 받기
    public boolean requestVote(VoteRequest request) {
        // 세대 확인
        if (request.term < currentTerm) {
            return false; // 낮은 세대
        }
        
        // 이미 이 세대에서 투표했나?
        if (votedFor != null && votedFor != request.candidateId) {
            return false;
        }
        
        // 후보자의 로그가 최신인가?
        if (request.lastLogTerm < log.getLastTerm() ||
            (request.lastLogTerm == log.getLastTerm() && 
             request.lastLogIndex < log.getLastIndex())) {
            return false; // 로그가 뒤처짐
        }
        
        // 투표!
        votedFor = request.candidateId;
        return true;
    }
    
    // 4. 리더가 되기
    private void becomeLeader() {
        state = State.LEADER;
        System.out.println("나는 리더다! Term: " + currentTerm);
        
        // 모든 팔로워에게 하트비트 시작
        startSendingHeartbeats();
    }
}
```

**실제 선거 시나리오:**
```
서버 3대: A, B, C

T=0: 모두 팔로워, 리더 A
A → B,C: 하트비트
A → B,C: 하트비트
A → B,C: 하트비트

T=10: A 다운! 💥

T=13: B의 하트비트 타임아웃
B: "리더 죽었나? 선거 시작!"
B → C: "나 뽑아줘! Term=2"
C: "OK, 투표!"

B: "과반수 득표! (B+C = 2/2)"
B: "나는 새 리더! Term=2"

T=14: B → C: 하트비트 (리더로서)
C: "B가 새 리더구나"

T=20: A 복구
A: "나 리더인데..." (Term=1)
B → A: "안돼, 나 리더야" (Term=2)
A: "아, 세대가 높네... 팔로워로 전환"
```

---

# 📚 7장: 하트비트 (Heartbeat)

## 💡 핵심 개념

**문제:** 서버가 죽었는지 어떻게 알아?

**해결:** "나 살아있어!" 신호를 주기적으로 보내자!

---

## 🔍 구체적인 시나리오

```
리더: "나 살아있어!" (1초마다)
팔로워: "OK, 리더 살아있음"

리더: "나 살아있어!"
팔로워: "OK"

리더: "나 살아있어!"
팔로워: "OK"

💥 리더 다운!

팔로워: "1초... 2초... 3초..."
팔로워: "3초 동안 신호 없음!"
팔로워: "리더 죽은 것 같아, 선거 시작!"
```

---

## 💻 실제 구현

```java
public class HeartbeatManager {
    private static final long HEARTBEAT_INTERVAL = 1000; // 1초
    private static final long TIMEOUT = 3000; // 3초
    
    private long lastHeartbeatTime;
    private ScheduledExecutorService scheduler;
    
    // 1. 리더: 하트비트 전송
    public void startSendingHeartbeats() {
        scheduler.scheduleAtFixedRate(() -> {
            HeartbeatMessage msg = new HeartbeatMessage(
                leaderId: myId,
                term: currentTerm,
                commitIndex: log.getCommitIndex()
            );
            
            for (ClusterNode follower : followers) {
                follower.receiveHeartbeat(msg);
            }
            
            System.out.println("[" + now() + "] 하트비트 전송");
            
        }, 0, HEARTBEAT_INTERVAL, TimeUnit.MILLISECONDS);
    }
    
    // 2. 팔로워: 하트비트 수신
    public void receiveHeartbeat(HeartbeatMessage msg) {
        // 세대 확인
        if (msg.term < currentTerm) {
            System.out.println("옛날 리더의 하트비트, 무시");
            return;
        }
        
        // 시간 갱신
        lastHeartbeatTime = System.currentTimeMillis();
        
        // 리더 정보 업데이트
        this.leaderId = msg.leaderId;
        this.currentTerm = msg.term;
        
        System.out.println("[" + now() + "] 하트비트 수신 from " + 
                          msg.leaderId);
    }
    
    // 3. 팔로워: 타임아웃 확인
    public void checkTimeout() {
        scheduler.scheduleAtFixedRate(() -> {
            long now = System.currentTimeMillis();
            long elapsed = now - lastHeartbeatTime;
            
            if (elapsed > TIMEOUT) {
                System.out.println("[" + now() + "] 타임아웃! " + 
                                  elapsed + "ms 경과");
                onHeartbeatTimeout();
            }
            
        }, 0, 100, TimeUnit.MILLISECONDS);
    }
    
    // 4. 타임아웃 처리
    private void onHeartbeatTimeout() {
        System.out.println("리더 죽은 듯, 선거 시작!");
        startElection();
    }
}
```

**실제 로그:**
```
[10:00:00.000] 리더A: 하트비트 전송
[10:00:00.001] 팔로워B: 하트비트 수신 from A
[10:00:00.001] 팔로워C: 하트비트 수신 from A

[10:00:01.000] 리더A: 하트비트 전송
[10:00:01.001] 팔로워B: 하트비트 수신 from A
[10:00:01.001] 팔로워C: 하트비트 수신 from A

[10:00:02.000] 💥 리더A 크래시!

[10:00:03.000] (하트비트 없음)
[10:00:03.100] 팔로워B: 경과 시간 2100ms
[10:00:03.200] 팔로워B: 경과 시간 2200ms
...
[10:00:05.000] 팔로워B: 경과 시간 3000ms
[10:00:05.001] 팔로워B: 타임아웃! 선거 시작!
[10:00:05.002] 팔로워C: 타임아웃! 선거 시작!
```

---

## ⚠️ HOL 블로킹 문제

```java
// ❌ 나쁜 예: 단일 큐
public class BadServer {
    private BlockingQueue<Task> queue;
    
    public void processQueue() {
        while (true) {
            Task task = queue.take();
            
            if (task instanceof WriteTask) {
                // 느린 디스크 쓰기 (100ms)
                writeToDisk(task);
            } else if (task instanceof HeartbeatTask) {
                // 빠른 하트비트 (1ms)
                sendHeartbeat(task);
            }
        }
    }
}

// 문제:
// WriteTask가 큐 앞에 있으면
// HeartbeatTask가 100ms 동안 대기!
// → 하트비트 지연 → 가짜 타임아웃!
```

```java
// ✅ 좋은 예: 별도 스레드
public class GoodServer {
    private BlockingQueue<WriteTask> writeQueue;
    private Thread heartbeatThread;
    
    public void start() {
        // 쓰기 작업 스레드
        Thread writeThread = new Thread(() -> {
            while (true) {
                WriteTask task = writeQueue.take();
                writeToDisk(task); // 느림
            }
        });
        writeThread.start();
        
        // 하트비트 전용 스레드
        heartbeatThread = new Thread(() -> {
            while (true) {
                sendHeartbeat(); // 빠름
                Thread.sleep(1000);
            }
        });
        heartbeatThread.setPriority(Thread.MAX_PRIORITY);
        heartbeatThread.start();
    }
}
```

---

# 📚 8장: 과반수 정족수 (Quorum)

## 💡 핵심 개념

**문제:** 몇 대의 서버에 복제되면 "성공"이라고 말할까?
- 1대? → 그 서버 죽으면 데이터 유실
- 5대 전부? → 1대만 느려도 전체가 느려짐

**해결:** 과반수! (절반 이상)

---

## 🔍 구체적인 시나리오

```
서버 5대: A(리더), B, C, D, E

철수가 아이템 구매:

A: 로그 기록 완료 (1/5)
A → B: 복제 → 성공 (2/5)
A → C: 복제 → 성공 (3/5) ← 과반수!
A → D: 복제 → 느림... (응답 없음)
A → E: 복제 → 실패 (서버 다운)

과반수 = (5/2) + 1 = 3
3대 성공 → 커밋! ✅

철수: "구매 완료!" 메시지 받음
```

---

## 💻 실제 구현

```java
public class QuorumManager {
    private List<ClusterNode> cluster;
    private int quorumSize;
    
    public QuorumManager(List<ClusterNode> cluster) {
        this.cluster = cluster;
        // 과반수 계산
        this.quorumSize = (cluster.size() / 2) + 1;
    }
    
    // 1. 로그 복제 (과반수 대기)
    public boolean replicateWithQuorum(LogEntry entry) {
        // 리더 자신 = 1표
        AtomicInteger ackCount = new AtomicInteger(1);
        CountDownLatch latch = new CountDownLatch(quorumSize - 1);
        
        // 병렬로 복제 요청
        for (ClusterNode follower : followers) {
            executor.submit(() -> {
                try {
                    boolean success = follower.replicate(entry);
                    if (success) {
                        int count = ackCount.incrementAndGet();
                        System.out.println("복제 성공 (" + count + 
                                         "/" + cluster.size() + ")");
                        latch.countDown();
                    }
                } catch (Exception e) {
                    System.err.println("복제 실패: " + follower.getId());
                }
            });
        }
        
        try {
            // 과반수 대기 (최대 1초)
            boolean reached = latch.await(1, TimeUnit.SECONDS);
            
            if (reached) {
                System.out.println("과반수 달성! 커밋!");
                entry.setCommitted(true);
                return true;
            } else {
                System.err.println("과반수 실패: " + ackCount.get() + 
                                 "/" + quorumSize);
                return false;
            }
            
        } catch (InterruptedException e) {
            return false;
        }
    }
}
```

**타임라인:**
```
T=0.000: 리더A: 로그 추가 완료
T=0.001: A → B,C,D,E: 복제 요청 전송 (병렬)
T=0.050: B: 복제 완료 → A: ACK (1/4)
T=0.051: C: 복제 완료 → A: ACK (2/4)
T=0.052: A: 과반수 달성! (A+B+C=3/5)
T=0.052: 클라이언트: "성공!" 응답 ✅
T=0.500: D: 복제 완료 → A: ACK (3/4, 늦음)
T=1.000: E: 타임아웃 (응답 없음)

결과: D,E 느려도 OK! 이미 커밋됨!
```

---

## 🎯 클러스터 크기 결정

```
1대: 정족수 1 → 실패 허용 0대 ❌
2대: 정족수 2 → 실패 허용 0대 ❌ (의미 없음)
3대: 정족수 2 → 실패 허용 1대 ✅
4대: 정족수 3 → 실패 허용 1대 (비효율)
5대: 정족수 3 → 실패 허용 2대 ✅
6대: 정족수 4 → 실패 허용 2대 (비효율)
7대: 정족수 4 → 실패 허용 3대 ✅

결론: 대부분 3대 또는 5대 사용!
```

**왜 짝수는 비효율?**
```
4대 vs 5대:
- 둘 다 1대 실패 허용
- 4대: 정족수 3 (75% 응답 필요)
- 5대: 정족수 3 (60% 응답 필요)
→ 5대가 더 유리!

6대 vs 7대:
- 둘 다 2대 실패 허용
- 6대: 정족수 4 (67% 응답 필요)
- 7대: 정족수 4 (57% 응답 필요)
→ 7대가 더 유리!
```

---

# 📚 9장: 세대 시계 (Generation Clock / Term)

## 💡 핵심 개념

**문제:** 옛날 리더가 살아나서 명령하면?

**해결:** 각 리더에게 세대 번호를 붙이자!

---

## 🔍 구체적인 시나리오

```
T=0: 서버A가 1대 리더
A: "철수 돈 = 10000원" (세대 1)

T=1: A 네트워크 끊김 💥
B가 2대 리더로 선출

T=2: B: "철수 돈 = 8000원" (세대 2)

T=3: A 네트워크 복구!
A: "나 리더인데? 철수 돈 = 10000원" (세대 1)
B: "안돼, 세대 1은 옛날 거야! 지금은 세대 2!"

A: "아... 내가 뒤처졌구나. 팔로워로 전환"
```

---

## 💻 실제 구현

```java
public class GenerationClock {
    private int currentTerm = 0; // 현재 세대
    
    // 1. 서버 시작 시 세대 복구
    public void initialize() {
        // 로그에서 최신 세대 읽기
        this.currentTerm = log.getLatestTerm();
        System.out.println("복구된 세대: " + currentTerm);
    }
    
    // 2. 리더 선출 시 세대 증가
    public void onLeaderElection() {
        currentTerm++;
        log.persistTerm(currentTerm);
        System.out.println("새 세대 시작: " + currentTerm);
    }
    
    // 3. 요청 처리 (세대 검증)
    public Response handleRequest(Request request) {
        int requestTerm = request.getTerm();
        
        if (requestTerm < currentTerm) {
            // 옛날 리더의 요청
            return Response.fail("낮은 세대: " + requestTerm + 
                               " (현재: " + currentTerm + ")");
        }
        
        if (requestTerm > currentTerm) {
            // 내가 뒤처짐!
            System.out.println("더 높은 세대 발견: " + requestTerm);
            currentTerm = requestTerm;
            stepDown(); // 리더 그만둠
        }
        
        // 같은 세대 → 처리
        return processRequest(request);
    }
    
    // 4. 로그 엔트리에 세대 포함
    public void appendLog(String data) {
        LogEntry entry = new LogEntry(
            term: currentTerm, // ← 세대 포함!
            data: data
        );
        log.append(entry);
    }
    
    // 5. 충돌 해결
    public void resolveConflict() {
        LogEntry myEntry = log.get(100);
        LogEntry leaderEntry = leader.getLogEntry(100);
        
        if (myEntry.getTerm() < leaderEntry.getTerm()) {
            // 내 엔트리가 낮은 세대 → 삭제
            System.out.println("충돌 엔트리 삭제: " + myEntry);
            log.truncateFrom(100);
        }
    }
}
```

**실제 시나리오:**
```
초기 상태:
서버A: 리더 (세대 1)
서버B: 팔로워 (세대 1)
서버C: 팔로워 (세대 1)

T=10:00:
A: "철수 아이템 추가" (세대 1)
→ B: 복제 OK
→ C: 복제 OK

T=10:01: A 네트워크 끊김!
B,C: "A 죽은 듯..."

T=10:02: B가 선거 시작
B: 세대 증가 → 2
B: "나 뽑아줘!" (세대 2)
C: "투표!" (세대 2)
B: 2대 리더 됨

T=10:03:
B: "영희 아이템 추가" (세대 2)
→ C: 복제 OK

T=10:04: A 네트워크 복구
A: "나 리더야! 철수 레벨업!" (세대 1)
→ B: "안돼, 세대 낮음"
→ C: "안돼, 세대 낮음"

A: 하트비트 받음 (세대 2, 리더=B)
A: "세대 2? 나보다 높네..."
A: 팔로워로 전환
A: 로그 동기화 (B의 로그 받아옴)
```

---

# 📚 10장: 하이 워터마크 (High Water Mark)

## 💡 핵심 개념

**문제:** 팔로워는 언제 커밋해야 할까?

**해결:** 리더가 "여기까지 확정!" 표시를 알려주자!

---

## 🔍 구체적인 시나리오

```
리더A: [1] [2] [3] [4] [5] HWM=5
       └─ 과반수 복제 완료! ─┘

팔로워B: [1] [2] [3] [4] [5] commit=?
팔로워C: [1] [2] [3] commit=?
팔로워D: [1] commit=?

리더: "HWM=5야!" (하트비트)

팔로워B: "5까지 커밋!" [1][2][3][4][5] ✅
팔로워C: "3까지만 있네... 3까지 커밋" [1][2][3] ✅
팔로워D: "1까지만... 1 커밋" [1] ✅
```

---

## 💻 실제 구현

```java
public class HighWaterMarkManager {
    private int highWaterMark = 0;
    private Map<String, Integer> followerProgress = new HashMap<>();
    
    // 1. 리더: HWM 계산
    public void updateHighWaterMark() {
        // 각 팔로워가 어디까지 받았는지 수집
        List<Integer> indexes = new ArrayList<>();
        indexes.add(log.getLastIndex()); // 리더 자신
        
        for (String followerId : followers.keySet()) {
            int index = followerProgress.get(followerId);
            indexes.add(index);
        }
        
        // 정렬
        Collections.sort(indexes);
        
        // 과반수 위치 = HWM
        int quorumPosition = indexes.size() / 2;
        int newHWM = indexes.get(quorumPosition);
        
        if (newHWM > highWaterMark) {
            System.out.println("HWM 업데이트: " + highWaterMark + 
                             " → " + newHWM);
            highWaterMark = newHWM;
        }
    }
    
    // 2. 리더: 복제 응답 처리
    public void onReplicationAck(String followerId, int lastIndex) {
        // 팔로워 진행 상황 업데이트
        followerProgress.put(followerId, lastIndex);
        
        // HWM 재계산
        updateHighWaterMark();
    }
    
    // 3. 리더: 하트비트에 HWM 포함
    public void sendHeartbeat() {
        HeartbeatMessage msg = new HeartbeatMessage(
            term: currentTerm,
            leaderId: myId,
            highWaterMark: this.highWaterMark // ← 포함!
        );
        
        for (ClusterNode follower : followers) {
            follower.receiveHeartbeat(msg);
        }
    }
    
    // 4. 팔로워: HWM 수신 → 커밋
    public void onHeartbeat(HeartbeatMessage msg) {
        int leaderHWM = msg.highWaterMark;
        
        System.out.println("HWM 수신: " + leaderHWM);
        
        // HWM까지 커밋
        for (int i = commitIndex + 1; i <= leaderHWM; i++) {
            LogEntry entry = log.get(i);
            if (entry != null) {
                applyToStateMachine(entry);
                commitIndex = i;
                System.out.println("커밋: " + i);
            } else {
                // 로그가 없으면 리더에게 요청
                System.out.println("로그 " + i + " 없음, 요청");
                requestLog(i);
                break;
            }
        }
    }
    
    // 5. 클라이언트 읽기
    public LogEntry read(int index) {
        if (index > highWaterMark) {
            throw new IllegalArgumentException(
                "읽을 수 없음: " + index + " > HWM(" + 
                highWaterMark + ")"
            );
        }
        return log.get(index);
    }
}
```

**구체적인 예시:**
```
초기:
리더:   [1][2][3][4][5] commit=5 HWM=5
팔A:    [1][2][3][4][5] commit=2
팔B:    [1][2][3]       commit=2
팔C:    [1][2][3][4]    commit=2

T=1: 리더 → 하트비트 (HWM=5)

팔A 수신:
"HWM=5? 나는 5까지 있어!"
[3][4][5] 커밋 실행
commit=5 ✅

팔B 수신:
"HWM=5? 나는 3까지만 있어..."
[3] 커밋 실행
"4,5번 로그 요청!"
리더 → 팔B: [4][5] 전송

팔B:
[4][5] 수신
[4][5] 커밋 실행
commit=5 ✅

팔C 수신:
"HWM=5? 나는 4까지 있어"
[3][4] 커밋 실행
"5번 로그 요청!"
리더 → 팔C: [5] 전송
commit=5 ✅
```

---

## 🔧 로그 절단 (Truncation)

```java
public class LogTruncation {
    
    // 서버 재시작 후 재합류
    public void rejoinCluster() {
        // 1. 최신 리더 찾기
        ClusterNode leader = findLeader();
        
        // 2. 리더의 HWM 요청
        int leaderHWM = leader.getHighWaterMark();
        int leaderTerm = leader.getCurrentTerm();
        
        System.out.println("리더 HWM: " + leaderHWM);
        
        // 3. 내 로그 확인
        for (int i = log.getLastIndex(); i > leaderHWM; i--) {
            LogEntry entry = log.get(i);
            
            if (entry.getTerm() < leaderTerm) {
                // 낮은 세대 → 충돌!
                System.out.println("충돌 엔트리 발견: " + i);
                log.truncateFrom(i); // 삭제
            }
        }
        
        // 4. 누락된 로그 받아오기
        for (int i = log.getLastIndex() + 1; i <= leaderHWM; i++) {
            LogEntry entry = leader.getLogEntry(i);
            log.append(entry);
            System.out.println("로그 복구: " + i);
        }
        
        System.out.println("재합류 완료!");
    }
}
```

**시나리오:**
```
T=0: 서버A (리더, 세대 2)
로그: [1,세대1][2,세대1][3,세대2][4,세대2][5,세대2]
HWM=5

T=1: 서버B 다운
로그: [1,세대1][2,세대1][3,세대2]

T=2: A 다운, C가 3대 리더 (세대 3)
로그: [1][2][3][4,세대3][5,세대3][6,세대3]
HWM=6

T=3: B 재시작, 재합류
B의 로그: [1][2][3] ← 뒤처짐

B: "리더 C의 HWM은 6이네"
B: "나는 3까지만 있어"
C → B: [4,세대3][5,세대3][6,세대3] 전송
B: 로그 추가
B 로그: [1][2][3][4][5][6] ✅
```

**충돌 시나리오:**
```
T=0: A가 2대 리더
로그: [1][2][3,세대2][4,세대2]
HWM=4

T=1: A 다운 직전, B에게만 전송
B: [1][2][3,세대2][4,세대2]
C: [1][2]

T=2: C가 3대 리더
C: [1][2][3,세대3]
HWM=3

T=3: B 재합류
B: [1][2][3,세대2][4,세대2] ← 충돌!
     
리더: "HWM=3이야"
B: "4번은? ... 세대2네, 리더는 세대3"
B: "4번 삭제!" (충돌)
B: [1][2][3,세대2] ← 절단
B: "3번도 세대2... 리더는 세대3"
B: "3번 삭제!"
B: [1][2]
리더 → B: [3,세대3] 전송
B: [1][2][3,세대3] ✅
```

---

# 📚 핵심 패턴 총정리

## 🎯 전체 흐름

```
1. 쓰기 전 로그 (WAL)
   └─ 모든 변경사항을 먼저 디스크에!
   
2. 분할 로그
   └─ 로그를 작은 파일로 쪼개기
   
3. 로우 워터마크
   └─ 오래된 로그 안전하게 삭제
   
4. 리더-팔로워
   └─ 한 명만 결정, 나머지는 복사
   
5. 하트비트
   └─ "나 살아있어!" 신호
   
6. 과반수 정족수
   └─ 절반 이상 동의하면 확정
   
7. 세대 시계
   └─ 옛날 리더 구분하기
   
8. 하이 워터마크
   └─ "여기까지 확정!" 표시
```

---

## 🔄 실제 동작 예시

```
게임 서버 클러스터 (3대):

철수가 "전설의 검" 구매 요청:

1. 리더A: WAL에 기록
   [1001] BUY_ITEM user=철수 item=전설의검
   
2. 리더A → B,C: 복제
   B: OK (1/2)
   C: OK (2/2)
   
3. 과반수 도달 (A+B+C=3/3)
   
4. 하이 워터마크 업데이트
   HWM = 1001
   
5. 리더 → B,C: 하트비트 (HWM=1001)
   
6. B,C: HWM=1001까지 커밋
   
7. 철수: "구매 완료!" ✅

만약 B가 다운되면:
1. C: "B 하트비트 안와... 3초..."
2. 과반수는 A+C=2/3 ← 여전히 OK!
3. 계속 작동 ✅

만약 A(리더)가 다운되면:
1. B,C: "A 하트비트 안와..."
2. B,C: 선거 시작 (세대 증가)
3. B가 새 리더
4. 계속 작동 ✅
```

---

이 패턴들은 Kafka, Cassandra, MongoDB, Raft, Zookeeper 등
실제 분산 시스템에서 모두 사용됩니다! 🚀