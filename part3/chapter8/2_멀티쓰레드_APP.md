# 2. 멀티쓰레드 APP

## 교착상태의 예시

언제 교착상태가 일어나는지 코드를 보자.

```c
pthread_mutex_t mutex1;
pthread_mutex_t mutex2;

// 뮤택스의 초기화 과정
pthread_mutex_init(&mutex1, NULL);
pthread_mutex_init(&mutex2, NULL); 
```

두 개의 뮤택스가 위처럼 초기화된 상황에서, 쓰레드 2개가 만들어져 실행된다고 하자.

```c
// 1st thread는 이 함수를 실행
void *work1()
{
    pthread_mutex_lock(&mutex1);
    pthread_mutex_lock(&mutex2);
    // 업무 처리
    pthread_mutex_unlock(&mutex2);
    pthread_mutex_unlock(&mutex1);

    pthread_exit(0);
}

// 2nd thread는 이 함수를 실행
void *work2()
{
    pthread_mutex_lock(&mutex2);
    pthread_mutex_lock(&mutex1);
    // 업무 처리
    pthread_mutex_unlock(&mutex1);    
    pthread_mutex_unlock(&mutex2);    

    pthread_exit(0);
}
```

해당 코드를 정리해보면 다음과 같다.

|  | 쓰레드 1 | 쓰레드 2 |
| --- | --- | --- |
| 접근 1 | mutex 1  | mutex 2 |
| 접근 2 | mutex 2 | mutex 1 |

접근 1을 시도할 때 교착상태가 날 수 있음.

물론, 쓰레드1이 뮤택스1 & 뮤택스2를 모두 획득하고 방출한 후 쓰레드2가 접근 1, 접근 2를 시도한다면 문제가 없겠지만..

그치만 현실에서는 이러한 상황을 잡는 것이 어렵다.

## 라이브락

또 다른 형태의 라이브니스 장애.

- 공통점 : 교착상태와 마찬가지로 2개 이상의 쓰레드의 행동을 방해함.
- 차이점 : 발생 원인이 서로 다름
    - 데드락 : 모든 각각의 쓰레드가 다른 쓰레드에 의해서만 발생하는 이벤트를 기다릴 때 발생
    - 라이브락 : 쓰레드가 실패한 행동을 계속할 때 발생

### 예시 : 코드

```c
void *work1()
{
    int done = 0;

    while (!done)
    {
        pthread_mutex_lock(&mutex1);
        if (pthread_mutex_trylock(&mutex2))
        {
            // 업무 처리
            pthread_mutex_unlock(&mutex2);
            pthread_mutex_unlock(&mutex1);
            done = 1;
        }
        else
            pthread_mutex_unlock(&mutex1);
    }
    pthread_exit(0);
}

void *work2()
{
    int done = 0;

    while (!done)
    {
        pthread_mutex_lock(&mutex2);
        if (pthread_mutex_trylock(&mutex1))
        {
            // 업무 처리
            pthread_mutex_unlock(&mutex1);
            pthread_mutex_unlock(&mutex2);
            done = 1;
        }
        else
            pthread_mutex_unlock(&mutex2);
    }
    pthread_exit(0);
}
```

`pthread_mutex_unlock` : 봉쇄되지 않고 뮤택스 락 획득 시도

따라서, 다음과 같은 상황이 벌어지면… 

- 쓰레드 1 ⇒ 뮤택스 1 획득 & 쓰레드 2 ⇒ 뮤택스 2 획득
- 이후 if문으로 들어가는 것이 계속 실패.
- else로 가서 락 해제 후 계속 무한반복.

이 상황이 바로 라이브락.

### 해결법

각 쓰레드가 실패한 행동을 재시도하는 시간을 랜덤하게 정하면 됨.

⇒ 이 방식은 네트워크 충돌이 발생할 때 Ethernet 네트워크가 취하는 방식

⇒ 흔히 일어나지는 않지만, 병행 앱을 설계하는 과정에 있어 어려운 문제

⇒ 특정한 스케쥴링 상황에서만 발생할 수 있음.