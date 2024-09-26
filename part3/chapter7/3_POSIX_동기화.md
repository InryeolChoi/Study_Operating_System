# 3. POSIX 동기화

사용자 수준에서 프로그래머가 사용할 수 있고, 특정 운영체제의 일부가 아님.

물론, 호스트 운영체제가 사용하는 도구를 사용해 구현해야 함.

# POSIX : 뮤택스 락

## 특징

- Pthreads에서 사용할 수 있는 기본적인 동기화기법.
- 코드의 임계구역을 보호하기 위해 사용
- 쓰레드는 임계구역 진입 전 락을 획득하고, 나갈 때 락을 방출.

## 코드

Pthread는 뮤택스 락의 데이터 형으로 pthread_mutex_t를 사용.

뮤택스는 pthread_mutex_init() 함수를 호출하여 생성.

- 1번째 매개변수는 뮤택스를 가르키는 포인터.
- 2번째 매개변수는 초기화 변수.

뮤택스는 옆의 두 함수를 이용해 획득 & 방출됨.

뮤택스 락을 못 획득 시 획득을 요청한 쓰레드는 락을 가지고 있는 쓰레드가 락을 방출할 때까지 봉쇄.

모든 뮤택스는 연산 성공시 0 반환

```c
#include <pthread.h>
pthread_mutex_t mutex;

pthread_mutex_init(&mutex, NULL); // 뮤택스 호출

pthread_mutex_lock(&mutex); // 뮤택스 락 걸기

/* 임계구역 */

pthread_mutex_unlock(&mutex); // 뮤택스 락 해제
```

# POSIX : 세마포어

## 특징

- Pthreads를 구현하는 많은 시스템은 세마포어도 함께 제공.
- POSIX는 **named(기명)**과 **unnamed(무명)**, 즉 2개 유형의 세마포어를 명기.
- 둘은 유사하지만 프로세스 간의 생성 및 공유 방식이 다름.

## 기명 세마포어

가명 세마포어는…

- 여러 관련 없는 프로세스가 세마포어 이름만 참조해 동기화 기법으로 공통 세마포어를 동기화 기법으로 쉽게 사용가능.

세마포어를 생성하고 열 때 `sem_open()`을 사용

- 세마포어를 SEM이라고 부르고 있음.
- `O_CREAT` : 세마포어가 없으면 생성
- `0666` : 타 프로세스에 읽기 및 쓰기 권한 부여
- `1` : 1로 초기화 됨.

생성 시 기존 세마포어의 디스크립터는 반환.

앞에서 설명한 대로 wait()와 post()를 각각 sem_wait()와 sem_post()로 구현하고 있음.

```c
#include <semaphore.h>
sem_t *sem;

sem = sem_open("SEM", O_CREAT, 0666, 1);

sem_wait(&sem);

/* 임계구역 */

sem_post(&sem);
```

## 무명 세마포어

`sem_init()` 함수로 초기화

- `&sem` : 세마포어를 가리키는 포인터
- `0` : 공유 수준을 나타내는 플래그
- `1` : 세마포어의 초기값.

앞에서 설명한 대로 wait()와 post()를 각각 sem_wait()와 sem_post()로 구현하고 있음.

이는 기명 세마포어와 동일함.

```c
#include <semaphore.h>
sem_t *sem;

sem_init(&sem, 0, 1);

sem_wait(&sem);

/* 임계구역 */

sem_post(&sem);
```

뮤택스 락과 마찬가지로 모든 세마포어 함수는 성공하면 0, 실패하면 기타 값을 반환.

# POSIX : 조건변수

Pthread는 일반적으로 C에서 사용되는데, C에는 모니터가 없으므로 조건변수를 뮤택스 락과 연결하여 락킹을 제공한다.

**조건변수**

- `pthread_cond_t` 라는 데이터 유형을 사용
- `cond_init()` 이라는 함수를 사용하여 초기화.

`pthread_cond_wait()` ⇒ 조건변수를 기다리는데 사용

- 조건 변수와 연관된 뮤택스 락은 `pthread_cond_wait()` 함수가 호출되기 전에 획득되어야 한다.
- 이는 가능한 경쟁 조건으로부터 조건 절의 데이터를 보호하는데 사용되기 때문이다.

공유 데이터를 변경하는 쓰레드는 `pthread_cond_signal()` 함수를 호출하여 조건 변수를 기다리는 하나의 쓰레드에 신호할 수 있다.

`pthread_cond_signal()` 호출은 뮤텍스 락을 해제하지 않음.

이를 해제하는 것은 `pthread_mutex_unlock()`

해제 이후, 쓰레드는 뮤택스 락의 소유자가 되며 실행 재개.

```c
pthread_mutex_t mutex;
pthread_cond_t cond_var;

pthread_mutex_init(&mutex, NULL);
pthread_cond_init(&cond_var, NULL);

pthread_mutex_lock(&mutex);

// true가 아니면 이 안에서 계속 맴돈다.
while (a != b) {
	pthread_cond_wait(&cond_var, &mutex);
}
pthread_mutex_unlock(&mutex);
```

```c
pthread_mutex_lock(&mutex);
a = b;

pthread_cond_signal(&cond_var);
pthread_mutex_unlock(&mutex);
```