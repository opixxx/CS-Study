# HTTP
## HTTP란?
HTTP 는 Hyper Text Transfer Protocol 의 약자이며,  처음에는 서버와 브라우저간에 데이터를 주고 받기 위해 설계된 프로토콜이다. 지금은 서버와 서버간의 통신할 때도 많이 이용한다.

## HTTP 특징
### Stateless
클라이언트의 `state`를 유지하지 않는다.
- 장점 : 서버 확장(Scale out)의 용이하다.
- 단점 : 클라이언트가 추가 데이터를 전송해야 한다.
  - 쿠키, 세션, 토큰 등을 이용해서 `state`를 유지한다.
### Connectionless
**Connectionless** 

<img width="515" alt="image 2" src="https://github.com/user-attachments/assets/552ea289-252b-4320-aa16-0293672ffb9e" />

- 요청을 주고 받을 때만 연결을 유지하고 응답을 주고나면 TCP/IP 연결을 끊는다.
- 최소한의 자원으로 서버 자원을 사용한다.
- HTTP/1.0 버전에서 사용

**Connectionless의 한계**
> TCP/IP 연결을 새로 해야하므로 3-way handshake를 다시 해야하기 때문에 속도가 느려진다.

**Coneection Oriented**

<img width="529" alt="image" src="https://github.com/user-attachments/assets/9a72ec21-36ff-4ae7-a451-d0a0ee1b8e2f" />

- TCP/IP의 경우 기본적으로 연결을 유지한다.
- 클라이언트가 요청을 보내지 않아도(일정 시간, Timeout) 계속 연결을 유지한다.
- HTTP/1.1 이후로 `Persistent Connection` 기능이 추가되었다.

## HTTP/1.1
- HTTP 헤더 + body로 구성되어 있다.
  - 헤더에는 URI, Request Method, 여러 헤더 정보가 포함되어있다.
- 사람이 읽을 수 있는 문자열이 그대로 전송된다.
- TCP Connection을 이용한다.
- Connection을 재사용한다.
- 파이프라이닝 추가
  - 요청에 대한 응답이 끝나기 전에 다음 데이터를 미리 요청한다.

### HTTP/1.1에 문제들
- 바이너리가 아닌 텍스트로 데이터를 보내는 것
- TCP 를 쓰는 것
- 여전히 요청 데이터가 동기적으로 진행되는 것(1개 요청하고 대기, 도착하면 그 다음 데이터 요청)
- 헤더에 중복된 데이터가 있는 것

## HTTP/2.0
HTTP/1.1에 문제점을 개선하기 위해 구글에서 HTTP/2.0을 개발했다.

HTTP/2 에 대해 설명하기 전에 알아야 할 단어들이 몇 가지가 있다.
1. 스트림
   - 구성된 연결 내에서 전달되는 바이트의 양뱡향 흐름, 하나 이상의 메시지가 전달 가능하다.
2. 메시지
   - 요청/응답에 해당하는 프레임의 전체 시퀀스
3. 프레임
   - HTTP/2.0 통신에서의 최소 단위이며, 각 프레임에는 하나의 프레임 헤더가 포함된다.

<img width="664" alt="스크린샷 2025-02-09 오후 9 07 34" src="https://github.com/user-attachments/assets/8f4d693b-d707-49ef-84c0-36baaa7ae23e" />

## HTTP/2.0 특징

### HTTP Body가 이진 데이터이다.
HTTP Request Method, 헤더 등은 여전히 문자열로 전송되지만 body 부분이 이진 데이터로 변경 됐다. HTTP/2.0부터는 `binary framing layer`라는 공간에 이진 데이터로 전송된다. 

### 멀티플렉싱
1.0 에서 1.1 으로 넘어오며 `pipelining` 기술을 도입하였지만, 여전히 `HOL(Head-of-line) Blocking` 문제가 있다. `HOL Blocking` 은 패킷이 순서대로 도착해야 하므로, 패킷이 도착할 때까지, 그 이후의 패킷은 전송되지 못하는 것을 의미한다. 위의 스트림의 개념을 사용하자면, 1.1 버전에서는 1개의 TCP 연결 당 1개의 스트림만 이용 가능하다. 따라서 하나의 메시지가 응답될 때까지 다른 메시지를 요청하지 못한다. 그래서 여러 TCP 연결을 수립하여 여러 요청을 수행한다. 하지만 2.0 을 이용하면 스트림을 이용하여 다음과 같은 장점이 있다.

<img width="789" alt="스크린샷 2025-02-09 오후 9 13 23" src="https://github.com/user-attachments/assets/3e2c659f-75e8-4890-aa98-6705312d2692" />

**하나의 커넥션으로 여러개의 메시지 스트림을 응답 순서에 상관없이 주고 받는 것을 멀티플렉싱이라 한다.**

### 스트림 우선순위 지정
1개의 TCP 에 여러 개의 스트림을 사용할 수 있게 되었으므로 각각의 스트림에 우선순위를 둘 수 있게 되었다. 이것을 이용하여 전송 순서를 임의로 고정시킬 수는 없으나, 중요한 데이터를 먼저 보낼 수 있도록 설정하는 것이 가능하다.

### 헤더 압축
1.0 에서는 헤더가 문자열로 전송된다. 적게는 500~800바이트 크게는 수 KB를 소모한다. 이러한 성능 이슈를 해결하기 위하여 2.0 에서는 **HPACK** 압축을 이용하여 헤더를 압축하여 보낸다.

### Server push
클라이언트에게 필요한 데이터가 있을 때, 직접 요청하기 전에 서버가 미리 데이터를 전송하여 받아볼 수 있게 한다.

이를 통해 TCP 연결을 1개로 합치고 HOL 문제도 해결했지만, 여전히 TCP 프로토콜 자체의 한계 때문에 문제점이 있다.
1. Handshake의 RTT로 인한 지연시간
2. TCP 자체의 HOL Blocking 문제
   - 기본적으로 TCP는 패킷이 유실되거나 오류가 있을 때 재전송하는데, 이 재전송 과정에서 패킷의 지연이 발생하면 결국 HOL Blokcing 문제가 발생한다.

<img width="796" alt="스크린샷 2025-02-09 오후 9 27 01" src="https://github.com/user-attachments/assets/2de15e67-3a49-4b21-ad85-09357a54a687" />

이를 개선하기 위해 UDP 기반 QUIC 프로토콜을 개발하고 이 QUIC 프로토콜이 TCP/IP 계층에도 동작시키기 위해 설계된 것이 HTTP/3.0이다.
## HTTP/3.0
TCP가 아닌 UDP를 사용하는 HTTP 프로토콜이다.
정확히는 QUIC(Quick UDP Internet Protocol) 프로토콜 위에서 돌아가는 HTTP 이다.

<img width="809" alt="스크린샷 2025-02-09 오후 9 30 37" src="https://github.com/user-attachments/assets/f284874d-95d1-4f28-ad75-b9f22c82e197" />

### QUIC 장점
**빠른 연결 설정**

**0-RTT 연결 : 이전에 연결한 적이 있는 서버의 경우, 추가적인 왕복 없이 즉시 데이터를 전송할 수 있다.**

<img width="782" alt="스크린샷 2025-02-09 오후 9 36 17" src="https://github.com/user-attachments/assets/c45dfbec-539b-4223-b011-43c9a5907bab" />

**1-RTT 연결: 새로운 서버와 연결할 때도 단 한 번의 왕복으로 암호화된 데이터 전송을 시작할 수 있습니다. TCP+TLS의 경우 최소 2-3번의 왕복이 필요한 것에 비해 큰 개선입니다.**

<img width="776" alt="스크린샷 2025-02-09 오후 9 36 27" src="https://github.com/user-attachments/assets/473c7e00-7d59-460b-86f9-bd01f6da4371" />

### 향상된 멀티플렉싱
HTTP/2.0 처럼 멀티플렉싱을 제공하지만 더욱 향상된 멀티플렉싱을 제공한다.
QUIC은 하나의 연결 내에서 여러 개의 독립적인 데이터 스트림을 처리할 수 있다.

<img width="784" alt="스크린샷 2025-02-09 오후 9 39 08" src="https://github.com/user-attachments/assets/e2ff701b-dd37-4ab5-8965-db3f6afc10a5" />

### 향상된 보안
* **전체 암호화**: 헤더를 포함한 거의 모든 정보를 암호화합니다. 이는 편지의 내용뿐만 아니라 봉투의 정보까지 암호화하는 것과 같습니다.
* **최신 암호화 프로토콜**: TLS 1.3을 사용하여 최신의 보안 기능을 제공합니다.
UDP 자체는 TCP에 비해 신뢰성이 없지만 QUIC 프로토콜을 통해 신뢰성을 보장할 수 있다.

**참고 래퍼런스**

> - 컴퓨터 네트워크
> - https://jinn-blog.tistory.com/204
