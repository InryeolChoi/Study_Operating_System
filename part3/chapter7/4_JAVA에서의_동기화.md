# 4. JAVA에서의 동기화

# 1. 모니터

```java
public class BoundedBuffer<E>
{
    private static final int BUFFER_SIZE = 5;

    private int count, in, out;
    private E[] buffer;

    public BoundedBuffer() {
        count = 0;
        in = 0;
        out = 0;
        buffer = (E[]) new Object[BUFFER_SIZE];
    }

    // 생산자 전용
    public synchronized void insert() {
        while (count == BUFFER_SIZE) {
            try {
                wait();
            }
            catch (InterruptedException ie) {}
        }
        buffer[in] = item;
        in = (in + 1) % BUFFER_SIZE;
        count++;
        notify();
    }

    // 소비자 전용
    public synchronized E remove() {
        E item;
        
        while (count == 0) {
            try {
                wait();
            }
            catch (InterruptedException ie) { }
        }
        item = buffer[out];
        out = (out + 1) % BUFFER_SIZE;
        count--;
        notify();
        return item;
    }
}
```

자바의 BoundedBuffer 클래스로 모니터를 구현.

- 자바의 메소드가 synchronized로 선언된 경우, 메소드를 호출하려면 반드시 그 객체와 연동된 락을 획득해야 한다.
- 이미 다른 쓰레드가 락을 가지고 있다면, 메소드를 호출하려던 쓰레드는 봉쇄 후 진입 집합에 추가됨.
- 어떠한 쓰레드의 락이 해제되면, JVM이 진입 집합에서 다른 쓰레드를 찾아냄. (대부분의 경우는 FIFO로 작동.)

락을 거는 것 외에도 모든 객체는 대기 집합과 연결됨.

예를 들어, 생산자가 insert()를 호출했는데 버퍼가 가득찬 경우, 쓰레드는 락을 해제하고 계속할 수 있는 조건이 충족될 때까지 기다림.

쓰레드가 wait() 메소드를 호출하면..

1. 쓰레드가 객체의 락을 해제
2. 쓰레드 상태가 봉쇄됨으로 변경
3. 쓰레드는 그 객체의 대기 집합에 넣어짐.

소비자가 생산자에게 진행할 수 있음을 알리는 방법 : `notify()`

`notify()`는 다음과 같이 작동한다.

1. 대기 집합의 쓰레드 리스트에서 임의의 쓰레드 T를 선택한다.
2. 쓰레드 T를 대기 집합에서 진입 집합으로 이동
3. T의 상태를 봉쇄됨에서 실행 가능으로 설정

T는 인제 다른 쓰레드와 락 경쟁을 할 수 있다.

T가 락 제어를 다시 획득하면 wait() 호출에서 복귀 후 count값을 재확인 가능.

# 2. 재진입 락

```java
public class practice2 {
    Lock key = new ReentrantLock();
    
    key.lock();
    try {
        // 임계구역
    }
    finally {
        key.unlock();
    }
}
```

ReentrantLock : Synchronized 제어자처럼 작동

- 단일 쓰레드가 소유, 상호 배타적 액세스 제공
- 오래 기다린 쓰레드에 락을 줄 수 있는 설정도 있음. (= 공정성 매개변수)

ReentrantLock의 작동방식

- lock() 메소드를 이용해 락을 획득
- unlock() 메소드를 이용해 락을 회수 → finally를 이용해 try가 어떻게 끝나든 락을 잘 회수할 수 있게 유도.
- 여러 쓰레드가 공유 데이터를 읽기만 하고 쓰지 않을 때는 너무 보수적인 전략일 수 있다.

# 3. 세마포어

```java
import java.util.concurrent.Semaphore;

public class practice3 {
    Semaphore sem = new Semaphore(1); // 초기값 지정

    try {
        sem.acquire(); // 인터럽트되면, ie 발생
        // 임계구역
    } 
    catch (InterruptedException ie) { }
    finally {
        sem.release();
    }
}
```

# 4. 조건변수

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class practice4 {
    Lock key = new ReentrantLock();
    condition condVar = key.newCondition(); // 조건변수 생성

    public void doWork(int threadNumber) {
        lock.lock();
        try {
            if (threadNumber != turn)
                condVars[threadNumber].await(); // wait() 구현

            //
            //
            turn = (turn + 1) % 5;
            condVars[turn].signal(); // signal() 구현
        } 
        catch (Exception e) { }
        finally {
            lock.unlock();
        }
    }
}
```