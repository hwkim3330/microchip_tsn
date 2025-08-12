## **실험 환경 설명**

### 1. 목적

* **NXP GoldBox**를 송신기(sender)로, **Intel i210 NIC**를 수신기(receiver)로 사용하여 `iperf3` UDP 트래픽 전송 실험을 수행.
* 중간에 **Microchip EVB-LAN9662** 스위치를 넣어, 스위치에서 CBS(Credit-Based Shaper) 등 TSN 기능 적용 시 성능 변화를 측정.

---

### 2. 네트워크 구성

```
[NXP GoldBox end0]  →  [Microchip EVB-LAN9662]  →  [Intel i210 enp2s0]
```

| 장치                        | 역할                      | 인터페이스          | OS/환경               | IP             |
| ------------------------- | ----------------------- | -------------- | ------------------- | -------------- |
| **NXP GoldBox**           | 송신(Sender)              | `end0`         | Linux               | 169.254.59.200 |
| **Microchip EVB-LAN9662** | 중간 스위치(포트 포워딩 + TSN 기능) | Port 1, Port 2 | VelocityDRIVE-SP FW | -              |
| **Intel i210 NIC PC**     | 수신(Receiver)            | `enp2s0`       | Linux               | 169.254.59.170 |

---

### 3. iperf3 명령어

#### 송신측(NXP GoldBox)

```bash
iperf3 -c 169.254.59.170 -u -p 5202 -b 10M
```

* `-u` : UDP 모드
* `-p 5202` : 포트 지정
* `-b 10M` : 전송 대역폭 10Mbps

#### 수신측(Intel i210)

```bash
iperf3 -s
```

* 수신 서버 모드

---

### 4. 데이터 흐름

1. NXP GoldBox의 `end0` 인터페이스에서 UDP 트래픽 발생
2. Microchip EVB-LAN9662 스위치 Port 1으로 유입
3. Port 2로 전달되어 Intel i210 NIC의 `enp2s0` 인터페이스로 전송
4. i210 PC에서 `iperf3 -s`로 트래픽 수신 및 성능 측정

---

### 5. 활용 포인트

* **CBS 실험 시나리오**
  EVB-LAN9662에서 PCP=3 큐에 CBS 적용 → Idle Slope 값 변경 → i210 측에서 레이턴시/지터/손실율 비교 가능.
* **다중 스트림 실험**
  NXP에서 AVB(PCP=3)와 BE(PCP=0) 트래픽 동시 송출 → 스위치의 대역폭 스케줄링 효과 검증.

네, 이해했습니다.
그럼 앞에서 작성했던 **NXP GoldBox → Microchip EVB-LAN9662 → Intel i210** 실험 환경 설명에 이어서,
이번에는 **CBS를 5 Mb/s로 설정하고 iperf 전송 속도를 1 Mb/s부터 10 Mb/s까지 순차적으로 변경**한 실험 환경을 이어서 정리해 드리겠습니다.

---

## **실험 환경 설명 (CBS 제한 + 송신 속도 변화)**

### 1. 목적

* **Microchip EVB-LAN9662** 스위치에서 **출력 포트에 CBS(Credit-Based Shaper)** 를 적용하여 대역폭 제한 효과를 검증.
* **Idle Slope = 5000 kbps (5 Mb/s)** 로 설정.
* 송신측(NXP GoldBox)에서 `iperf3` 전송 속도를 **1 Mb/s → 10 Mb/s**까지 단계적으로 증가시키며, 수신측(Intel i210)에서 실제 수신 속도를 관찰.

---

### 2. 네트워크 구성

```
[NXP GoldBox end0]  →  [Microchip EVB-LAN9662]  →  [Intel i210 enp2s0]
```

| 장치                        | 역할           | 인터페이스          | OS/환경            | IP             |
| ------------------------- | ------------ | -------------- | ---------------- | -------------- |
| **NXP GoldBox**           | 송신(Sender)   | `end0`         | Linux            | 169.254.59.200 |
| **Microchip EVB-LAN9662** | 스위치(CBS 적용)  | Port 1, Port 2 | VelocityDRIVE-SP | -              |
| **Intel i210 NIC PC**     | 수신(Receiver) | `enp2s0`       | Linux            | 169.254.59.170 |

---

### 3. CBS 설정 (Microchip EVB-LAN9662)

* 포트 2의 **Traffic Class 0**에 CBS 적용.
* Idle Slope를 \*\*5000 kbps (5 Mb/s)\*\*로 설정.
* 설정 예:

```yaml
- ? "/ietf-interfaces:interfaces/interface[name='2']/mchp-velocitysp-port:eth-qos/config/traffic-class-shapers"
  : traffic-class: 0
    credit-based:
      idle-slope: 5000   # 단위 kbps → 5 Mb/s
```

* 적용:

```bash
sudo dr mup1cc -d /dev/ttyACM0 -m ipatch -i ipatch-p2-cbs-tc0-5m.yaml
```

---

### 4. iperf3 송수신 설정

#### 수신측 (Intel i210)

```bash
iperf3 -s -p 5202
```

#### 송신측 (NXP GoldBox)

```bash
# 예시: 전송속도 1 Mb/s
iperf3 -c 169.254.59.170 -u -p 5202 -b 1M
# 2M ~ 10M까지 순차 실행
```

---

### 5. 실험 절차

1. CBS Idle Slope = **5 Mb/s** 로 고정.
2. NXP GoldBox에서 `iperf3` UDP 전송 속도를
   **1M, 2M, 3M, …, 10M** 순서로 변경.
3. 각 속도에서 **Intel i210 수신 측 평균 비트레이트**를 기록.
4. Wireshark I/O Graph에서 **출력 속도 변화** 및 CBS 동작 형태(버스트+쉬는 구간)를 확인.

---

### 6. 예상 결과

* 송신 속도 ≤ 5 Mb/s → 수신 속도 ≈ 송신 속도.
* 송신 속도 > 5 Mb/s → 수신 속도 ≈ 5 Mb/s로 제한 (CBS shaping 동작).
* Wireshark 그래프에서 **톱니 모양**의 전송 패턴 관찰됨
  (hiCredit까지 전송 후 idle 상태로 대기하는 CBS 특성).

---
 
