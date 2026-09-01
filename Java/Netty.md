#CS #Java #면접

관련: [[네트워크 프로그래밍]] · [[Redis 클라이언트]] · [[가상 스레드]]

## 목차
- [[#Netty란? — NIO를 직접 안 다루게 해주는 프레임워크]]
- [[#EventLoop — Selector를 감싼 일꾼]]
- [[#Channel과 ChannelPipeline — 이벤트가 흐르는 파이프라인]]
- [[#코드로 보는 Netty 서버]]
- [[#Netty가 주는 실질적인 이점]]
- [[#실제로 어디에 쓰일까?]]
- [[#한 장 요약]]
- [[#Q&A로 복습하기]]

---

## Netty란? — NIO를 직접 안 다루게 해주는 프레임워크

[[네트워크 프로그래밍]]에서 다룬 순수 NIO(`Selector`, `Channel`)는 강력하지만, **직접 다루려면 코드가 상당히 저수준이고 손이 많이 가요** — 이벤트 종류별 분기 처리, 버퍼 관리, 스레드 안전성까지 개발자가 다 신경 써야 해요.

**Netty**는 이 NIO를 **훨씬 사용하기 편한 형태로 감싼 비동기 네트워크 프레임워크**예요. "Selector를 직접 돌리며 이벤트를 분기 처리하는" 대신, **"이런 이벤트가 오면 이렇게 처리해줘"라고 선언적으로 등록**하면, 나머지 저수준 작업은 Netty가 알아서 해줘요.

```mermaid
graph TD
    subgraph "순수 NIO — 직접 관리"
        Dev1["개발자가 직접"] --> Sel["Selector 순회"]
        Sel --> Branch["이벤트 종류별 분기"]
        Branch --> Buf["버퍼 직접 관리"]
    end
    subgraph "Netty — 프레임워크가 대신"
        Dev2["개발자는 핸들러만 등록"] --> Netty["Netty가 이벤트 루프,<br/>버퍼, 스레드 관리를 대신"]
    end
```

> **비유**: 순수 NIO가 "부품을 사다가 직접 자동차를 조립하는 것"이라면, Netty는 "엔진, 변속기가 이미 조립된 섀시 위에 원하는 옵션만 골라 끼우는 것"에 가까워요.

---

## EventLoop — Selector를 감싼 일꾼

Netty의 핵심에는 **EventLoop**가 있어요. [[네트워크 프로그래밍]]에서 다룬 `Selector`를 **하나의 스레드가 계속 돌리며, 등록된 이벤트가 생기면 알맞은 핸들러를 실행해주는 일꾼**이라고 보면 돼요.

```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);   // 연결 수락 담당 (적은 스레드)
EventLoopGroup workerGroup = new NioEventLoopGroup();   // 실제 데이터 처리 담당 (여러 스레드)
```

- **`EventLoopGroup`**: EventLoop들을 여러 개 묶어둔 그룹. Netty 서버는 보통 **연결을 받는 그룹(boss)** 과 **실제 데이터를 처리하는 그룹(worker)** 을 분리해서 써요.
- 각 `EventLoop`는 **자기가 담당하는 Channel들을 전담**해요. 하나의 Channel은 항상 같은 EventLoop(같은 스레드)에서 처리되니까, **그 Channel에 관해서는 별도 동기화(`synchronized` 같은) 없이도 스레드 안전**해요.

---

## Channel과 ChannelPipeline — 이벤트가 흐르는 파이프라인

Netty에서 데이터가 들어오면, 그 데이터는 **파이프라인**을 통과하며 여러 **핸들러(Handler)** 를 순서대로 거쳐요.

```mermaid
graph LR
    In["데이터 수신"] --> H1["Handler 1<br/>(디코딩)"]
    H1 --> H2["Handler 2<br/>(로직 처리)"]
    H2 --> H3["Handler 3<br/>(인코딩)"]
    H3 --> Out["데이터 송신"]
```

각 핸들러는 **자기가 맡은 역할 하나만** 담당해요 — "바이트를 객체로 변환", "실제 비즈니스 로직 실행", "객체를 다시 바이트로 변환"처럼요. [[../Java/디자인 패턴|디자인 패턴]]에서 다룬 **Decorator 패턴**과 비슷하게, 필요한 기능을 파이프라인에 겹겹이 끼워 넣는 구조예요.

---

## 코드로 보는 Netty 서버

간단한 에코 서버를 Netty로 만들어볼게요.

```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();

ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(bossGroup, workerGroup)
    .channel(NioServerSocketChannel.class)
    .childHandler(new ChannelInitializer<SocketChannel>() {
        @Override
        protected void initChannel(SocketChannel ch) {
            ch.pipeline().addLast(new EchoHandler()); // 파이프라인에 핸들러 등록
        }
    });

ChannelFuture future = bootstrap.bind(8080).sync(); // 8080 포트에서 시작
future.channel().closeFuture().sync();
```

```java
class EchoHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ctx.writeAndFlush(msg); // 받은 데이터를 그대로 돌려줌
    }
}
```

[[네트워크 프로그래밍]]에서 본 순수 NIO의 `Selector` 루프, 이벤트 종류별 분기(`isAcceptable()`, `isReadable()`) 같은 코드가 **여기선 전혀 안 보여요** — `channelRead()`라는 콜백 하나에 "데이터가 오면 뭘 할지"만 적으면 끝이에요.

---

## Netty가 주는 실질적인 이점

- **이벤트 기반 파이프라인**: 프로토콜 파싱, 로깅, 비즈니스 로직을 핸들러 단위로 깔끔하게 분리할 수 있어요.
- **메모리 효율(`ByteBuf`)**: Java 표준 NIO의 `ByteBuffer`보다 개선된 자체 버퍼 구현으로, **불필요한 데이터 복사를 줄이는 제로카피(Zero-copy)** 최적화를 지원해요.
- **검증된 안정성**: 수많은 대규모 서비스에서 오랫동안 검증된 라이브러리라, 저수준 NIO를 직접 짤 때 흔히 발생하는 버그(메모리 누수, 스레드 경쟁 등)를 피할 수 있어요.
- **높은 처리량**: 소수의 스레드(EventLoop)로 수만 개의 동시 연결을 효율적으로 처리할 수 있어요.

---

## 실제로 어디에 쓰일까?

이 vault에서 이미 다룬 여러 기술들이 실제로 Netty 위에 만들어져 있어요.

| 기술 | Netty와의 관계 |
|---|---|
| [[Redis 클라이언트]]의 **Lettuce** | Netty 기반으로 만들어져, 커넥션 하나로 여러 스레드가 비동기 요청 처리 |
| **Kafka 클라이언트** | 내부 네트워크 통신에 NIO 기반 구조 사용 (Netty 계열 원리와 동일한 이벤트 기반 설계) |
| **Spring WebFlux** | 기본 서버로 **Reactor Netty**를 사용 — 리액티브 논블로킹 웹 서버 |
| **gRPC (Java)** | 기본 전송 계층으로 Netty를 사용 |
| **Elasticsearch Java 클라이언트** | 내부 통신에 Netty 사용 |

**정리하면**: 대부분의 개발자는 Netty를 직접 만지기보다, **Netty 위에 만들어진 더 상위 레벨의 라이브러리(Lettuce, WebFlux 등)를 통해 간접적으로** 그 성능 이점을 누려요.

---

## 한 장 요약

| 질문 | 답 |
|---|---|
| Netty란? | 순수 NIO를 사용하기 쉽게 감싼 비동기 네트워크 프레임워크 |
| EventLoop란? | Selector를 감싸 이벤트를 처리하는 일꾼, 같은 Channel은 항상 같은 EventLoop가 전담 |
| ChannelPipeline이란? | 데이터가 여러 핸들러를 순서대로 거치며 처리되는 구조 |
| Netty의 이점은? | 이벤트 기반 파이프라인, 제로카피 버퍼(ByteBuf), 검증된 안정성, 높은 처리량 |
| 실제 어디에 쓰이나? | Lettuce, Spring WebFlux(Reactor Netty), gRPC 등 여러 라이브러리의 기반 |

---

## Q&A로 복습하기

### Q. Netty가 순수 NIO보다 사용하기 쉬운 이유는?
A. Selector를 직접 순회하며 이벤트 종류별로 분기 처리하는 저수준 코드를 개발자가 짤 필요 없이, "이런 이벤트가 오면 이렇게 처리해줘"라는 핸들러만 등록하면 나머지는 프레임워크가 대신 처리해주기 때문이다.

### Q. 하나의 Channel이 항상 같은 EventLoop에서 처리되는 것이 왜 유리한가?
A. 그 Channel에 대한 처리가 항상 같은 스레드에서 일어나기 때문에, 별도의 동기화 없이도 스레드 안전성을 자연스럽게 보장할 수 있다.

### Q. ChannelPipeline의 핸들러 구조는 어떤 디자인 패턴과 유사한가?
A. Decorator 패턴과 유사하다. 데이터를 디코딩, 로직 처리, 인코딩 등 각자 역할을 맡은 핸들러들이 파이프라인에 순서대로 끼워져, 필요한 처리를 겹겹이 조합하는 방식이기 때문이다.

### Q. Lettuce가 Netty 기반이라는 사실이 시사하는 바는?
A. Lettuce가 커넥션 하나로 여러 스레드의 비동기 요청을 효율적으로 처리할 수 있는 이유가, Netty의 이벤트 기반 비동기 처리 구조를 그대로 활용하고 있기 때문임을 보여준다.
