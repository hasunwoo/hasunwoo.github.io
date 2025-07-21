# Task를 동시에 실행하려면 꼭 spawn이 필요한가?

2025.07.21

---

## 문제 상황

Embedded 개발 도중 embassy 프레임워크에서 Tcp 소켓을 동시에 송수신해야 했었는데, embassy 런타임에서는 spawn을 사용할 때 task에 `'static` lifetime만 넘겨줄 수 있다.
embassy-net의 `TcpSocket`을 `split()` 함수의 정의는 다음과 같다.

```rust
pub fn split(&mut self) -> (TcpReader<'_>, TcpWriter<'_>)
```

이 함수의 정의를 보면 알 수 있듯이 `split()`을 호출하면 반환되는 Read half와 Write Half의 lifetime이 `'static`이 아닐 수 있다.
tokio의 Tcp socket은 `split()` 을 하면 소유권이 있는 `OwnedReadHalf`와 `OwnedWriteHalf`로 깔끔히 분리되어 별도의 task로 분리하여 처리하기 쉽다.
그러나, embassy는 embedded 환경에서 돌아가는 최소한의 런타임이라 tokio처럼 편리하게 쓰는건 어렵다고 생각했다.

### 1-1. 초기 해결책

처음으로 생각했던 방법은 `select`를 써서 Future를 수동으로 제어하는 것이였다. 해당 코드는 아래와 같다.

```rust
let (mut reader, mut writer) = socket.split();

// data receiving future
let mut recv_fut = None;

'data: loop {
    // reset recv_fut if none
    if recv_fut.is_none() {
        drop(recv_fut);
        recv_fut = Some(recv_bincode_data::<Packet>(&mut reader));
    }

    // pin recv_fut for polling
    let mut recv_fut_pin = unsafe { Pin::new_unchecked(recv_fut.as_mut().unwrap()) };

    match select(&mut recv_fut_pin, packet_receiver.receive()).await {
        select::Either::First(recv_packet) => {
            // reset recv_fut
            recv_fut = None;

            let packet = match recv_packet {
                Some(data) => data,
                None => {
                    // recv failed. reset connection
                    break 'data;
                }
            };

            defmt::info!("got packet: {}", packet);

            if let Packet::RelayControl(level) = packet {
                relay_pin.set_level(level.into());
            }
        }
        select::Either::Second(send_packet) => {
            // send packet
            if !send_bincode_data(&mut writer, &send_packet).await {
                // send failed. reset connection
                break 'data;
            }
        }
    }
}
```

이 코드가 잘 동작했으나 '더 간단한 방법은 없을까?', '이게 최선인가?' 라는 생각이 계속 맴돌았다.


### 1-2. LLM에게 질문

결국 ChatGPT에게 상황을 설명하였고, 아래와 같은 해결책을 내게 알려줬다.

```rust
let (mut reader, mut writer) = socket.split();

let send_task = async {
    loop {
        let packet = packet_receiver.receive().await;
        if !send_bincode_data(&mut writer, &packet).await {
            // send failed. reset connection
            break;
        }
    }
};

let recv_task = async {
    loop {
        let packet = match recv_bincode_data::<Packet>(&mut reader).await {
            Some(data) => data,
            None => {
                // recv failed. reset connection
                break;
            }
        };

        defmt::info!("got packet: {}", packet);

        if let Packet::RelayControl(level) = packet {
            relay_pin.set_level(level.into());
        }
    }
};

select(send_task, recv_task).await;
```

이 코드를 보고 많은걸 느꼇다. 내가 몇일동안 고민해서 복잡한 방법으로 해결했던 문제를 너무 간단하게 해결해줬다.
저 코드가 개념적으로 어려운건 절대 아니다. 내가 다 알고 있는 개념인데 저렇게 조합하는 방법을 생각해지 못했었다.
내가 왜 저 방법을 생각하지 못했을까? 를 고민하며 나를 돌아봤다.


## 왜 쉬운 방법을 생각 해내지 못했을까?

인터넷에 돌아다니는 대부분의 공식 예제에서는 `spawn` + `select!` 조합을 사용한다. 내가 처음 비동기를 공부했을 때 접했던 정보는 대부분 아래와 같았다.

```rust
let recv_task = tokio::spawn(async move {
    while let Some(msg) = ws_receiver.next().await {
        ...
    }
});

let send_task = tokio::spawn(async move {
    while let Some(msg) = send_channel.recv().await {
        ws_sender.send(msg).await.unwrap();
    }
});

tokio::select! {
    _ = recv_task => send_task.abort(),
    _ = send_task => recv_task.abort(),
}
```

대부분의 예제가 위와 같은 구조를 가진다. 아니, 대부분이 아니라 내가 봤던 예제는 다 저런식으로 `spawn`을 사용했다.
`spawn`이 꼭 필요한가? 라는 의심을 한 적은 없었다. 아무 비판 없이 공식 예제가 정답인 줄 마냥 이 패턴을 받아들였고, 동시에 여러 Task를 처리하려면 무조건 저 조합을 써야한다는 생각이
뇌에 깊이 박혔다.

위 예제에서는 `spawn`이 과할 수 있다. `send_task`와 `recv_task`는 소켓 하나당 각각 생성된다. task의 개수가 무한히 늘어나지 않기 때문에 굳이 `spawn`을 쓸 필요가 없다.
`spawn`을 사용하게 되면 tokio 런타임이 task를 여러 OS thread에 분배할 수 있다. 만약 task의 수가 동적으로 계속 늘어나는 경우라면 당연히 `spawn`을 써야 한다.
그래야 하드웨어 스레드의 이점을 제대로 활용할 수 있다.

위의 경우에서는 Tcp 연결이 들어올 때는 task를 생성해서 처리하고, 각 연결 안에서 send와 recv를 분리할 때는 `spawn`이 필요하지 않다.
위 코드에서 불필요한 `spawn`을 제거하면 아래와 같다.

```rust
let recv_loop = async {
    while let Some(msg) = ws_receiver.next().await {
        ...
    }
};

let send_loop = async {
    while let Some(msg) = rx.recv().await {
        ws_sender.send(msg).await.unwrap();
    }
};

tokio::select! {
    _ = recv_loop => {},
    _ = send_loop => {},
}
```

내가 이 개념을 몰라서 못 떠올린건 아니였다. `async` 블럭이 `Future`이고, `Future`는 `select`가 가능하다는 사실을 알고 있었다.
`JoinHandle`만 `Future`이 아니다. `async` 블럭도 `Future` 이라서 `select`로 둘 중 하나가 완료될 때 까지 polling이 가능하다.
그러나 이런 방식의 코드를 한 번도 본 적이 없었고 공식 예제에 너무 사로잡혀 있어서 떠올리지 못했던거 같다.

## `spawn`을 쓰면 좋은 경우와 쓰지 않아도 되는 경우

| 상황 | spawn 사용 여부 | 설명 |
|------|------------------|------|
| 백그라운드 작업 | ✅ 필요 | 독립적으로 실행되어야 하기 때문에 |
| 소켓 수신 루프 (accept loop) | ✅ 필요 | task 수가 동적으로 늘어나기 때문에 |
| 하나의 연결 내 송/수신 | ❌ 불필요 | select!로 처리 가능 |
| 고정된 수의 sub-task | ❌ 불필요 | spawn 없이도 async block으로 충분 |


## 느낀점

1. 너무 복잡한거 같거나 이상한 느낌을 무시하지 말고 더 나은 방법을 고민하자.
2. 확증 편향을 경계하자. 항상 내 방식을 의심하고 동료 개발자나 LLM의 조언을 무시하지 말자.
3. 공식 예제라도 무조건적으로 받아들이지 말자. 공식 예제의 코드만 정답이 아니다.
4. 기존의 패턴에 갖히지 말고 다양한 방식을 생각해보자.(물론, 인간 인지력의 한계 때문에 어렵긴 하다. 동료 개발자나 LLM의 조언이 그래서 중요하다.)

개발자에게 가장 무서운 사고방식은 내 방식이 정답일거라고 단정하는 것인거 같다. 항상 겸손하게 더 나은 방법을 탐색하고
다른 사람이나 LLM의 생각도 참고하고 내 자신을 가장 경계해야 할것 같다.

