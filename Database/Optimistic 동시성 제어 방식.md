# Optimistic 방식
다른 트랜잭션과 충돌하지 않는다고 가정하고 별도의 Locking 없이 자원에 접근하는 것을 말한다.
병행 제어를 위해 아래 세 과정을 수행하며, 각 과정마다 Start(T), Validation(T), Finish(T) 세 가지의 타임 스탬프를 사용한다.

타임 스탬프 : 시스템에서 트랜잭션을 유일하게 식별하기 위해 부여한 식별자로 트랜잭션이 시스템에 들어온 순서대로 부여한다.

<img width="843" alt="스크린샷 2025-02-26 오후 6 21 42" src="https://github.com/user-attachments/assets/b4ad8003-e3e5-4a31-bc04-eddd6a4724b0" />

## 과정
1. 판독 단계(Read Step) : 트랜잭션에 필요한 자료를 DB로 부터 읽어 Local Working Area에 복사한다. 이후에 모든 갱신을 사본을 대상으로 수행한다.
2. 확인 단계(Validation Step) : 직렬 가능성 위반 여부를 검사한다.
3. 기록 단계(Write Step) : 확인 단계를 통과하면 DB에 반영하고, 통과하지 못했다면 롤백한다.
