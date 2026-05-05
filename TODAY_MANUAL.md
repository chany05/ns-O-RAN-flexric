# 오늘 만든 xApp 및 시나리오 설명 매뉴얼

## 1. 오늘 작업의 핵심

오늘 만든 것은 GUI 자체가 아니라, ns-3 O-RAN 시나리오에서 KPM을 받아 RF/RLC 제어를 수행하는 xApp과 그 xApp을 테스트할 수 있는 RF channel reconfiguration 시나리오다.

전체 구조는 다음과 같다.

```text
ns-3 orange-rf-channel-reconfiguration scenario
  -> E2 Agent
  -> nearRT-RIC
  -> RF Control xApp
  -> RC Control message
  -> ns-3 gNB 설정 변경
```

즉, 시연할 때의 주제는 다음 세 가지다.

```text
1. ns-3가 KPM을 만들어 RIC로 보낸다.
2. xApp이 KPM을 보고 정책을 판단한다.
3. xApp이 RC Control을 보내고 ns-3가 ACK 및 설정 반영을 수행한다.
```

## 2. RF Control xApp 설명

소스 위치:

```text
~/flexric/examples/xApp/c/ctrl/xapp_rc_rf_ctrl.c
```

실행 바이너리:

```text
~/flexric/build/examples/xApp/c/ctrl/xapp_rc_rf_ctrl
```

이 xApp은 FlexRIC 기반 C xApp이며, KPM service model과 RC service model을 같이 사용한다.

사용하는 E2 function ID:

```text
KPM: 2
RC:  3
```

구독하는 주요 KPM:

```text
DRB.UEThpDl.UEID
RRU.PrbUsedDl
Energy.TotalConsumption
RSRP.UEID
```

제어하는 RAN parameter:

```text
CellId:        1000
TxPower:       1001
RetTilt:       1002
RlcBufferSize: 1003
```

지원하는 제어 모드:

```text
txp          Tx power만 제어
ret          antenna RET tilt만 제어
rlc-buffer   RLC buffer size만 제어
txp-ret      Tx power + RET 제어
all          Tx power + RET + RLC buffer 정책 제어
```

대표 실행 예:

```bash
cd ~/flexric/build/examples/xApp/c/ctrl
./xapp_rc_rf_ctrl --mode all --cell 1 --txp-init 30 --tilt-init -6 --rlc-init 1048576 --iterations 0
```

## 3. xApp 정책 로직

xApp은 KPM indication을 받을 때마다 평균 throughput, PRB 사용률, energy, RSRP, edge UE 비율을 계산한다.

정책 판단 기준은 다음과 같다.

| 상태 | 조건 | 제어 |
| --- | --- | --- |
| `COVERAGE_LIMITED` | edge UE 비율이 높고 edge UE throughput이 낮음 | RET tilt 조정 |
| `CAPACITY_LIMITED` | 평균 throughput이 낮고 PRB 사용률이 높음 | Tx power 증가 |
| `BUFFER_LIMITED` | 평균 throughput이 낮고 PRB 사용률이 낮음 | RLC buffer 증가 |
| `ENERGY_WASTE` | throughput은 충분한데 energy가 높음 | Tx power 감소 |
| `STABLE` | 위 조건에 해당하지 않음 | 제어 없음 |

현재 기본 실험에서는 아래처럼 판단된다.

```text
avg_thp = 약 27473 kbps
prb     = 약 0.55
energy  = 약 3.2
UEs     = 5
state   = BUFFER_LIMITED
```

그래서 `--mode all` 실행 시 주로 RLC buffer control이 반복된다.

정상 동작 로그 예:

```text
[xApp RF] === Iteration #N ===
KPM: avg_thp=27473 kbps, prb=0.55, energy=3.2, UEs=5
State: BUFFER_LIMITED
[Policy RLC] buffer_pressure ... => RLC x2 = 67108864
[xApp]: CONTROL-REQUEST tx
[xApp]: CONTROL ACK rx
[xApp]: Successfully received CONTROL-ACK
```

이 로그가 있으면 `KPM 수신 -> 정책 판단 -> RC Control 전송 -> ACK 수신`까지 된 것이다.

## 4. orange RF 시나리오 설명

소스 위치:

```text
~/ns-O-RAN-flexric/mmwave-LENA-oran/scratch/orange-rf-channel-reconfiguration.cc
```

이 시나리오는 mmWave/NR gNB와 UE를 만들고, E2 Agent를 통해 nearRT-RIC와 연결되도록 구성한 테스트 시나리오다.

주요 command-line parameter:

```text
N_MmWaveEnbNodes       gNB 개수
N_Ues                  UE 개수
CenterFrequency        중심 주파수
Bandwidth              대역폭
IntersideDistanceUEs   UE 배치 반경
IntersideDistanceCells gNB 간 거리
simTime                시뮬레이션 시간
KPM_E2functionID       KPM function ID
RC_E2functionID        RC function ID
```

대표 실행 예:

```bash
cd ~/ns-O-RAN-flexric/mmwave-LENA-oran
./ns3 run "orange-rf-channel-reconfiguration --N_MmWaveEnbNodes=1 --N_Ues=5 --CenterFrequency=20e9 --Bandwidth=50e6 --IntersideDistanceUEs=500 --IntersideDistanceCells=1000 --simTime=100"
```

시나리오에서 하는 일:

```text
1. gNB를 중앙에 배치한다.
2. UE를 gNB 주변 원형 영역에 배치한다.
3. NR/mmWave channel과 mobility를 설정한다.
4. KPM E2 function ID 2, RC E2 function ID 3을 활성화한다.
5. gNB가 KPM indication을 만들고 RIC로 보낸다.
6. xApp에서 온 RC Control을 받아 TxPower, RET, RLC buffer를 적용한다.
```

## 5. ns-3 쪽 RC Control 반영

RC Control을 실제로 처리하는 위치:

```text
~/ns-O-RAN-flexric/mmwave-LENA-oran/src/nr/model/nr-gnb-net-device.cc
```

구현된 제어 함수:

```text
ApplyRanTxPowerControl
ApplyRanRetControl
ApplyRanRlcBufferSizeControl
```

각 함수의 역할:

```text
TxPower control:
  gNB PHY의 TxPower 값을 변경한다.

RET control:
  antenna array의 beta angle을 변경한다.

RLC buffer control:
  UE별 DRB의 RLC MaxTxBufferSize 값을 변경한다.
```

따라서 xApp의 RC Control은 단순 로그가 아니라 ns-3 gNB 객체의 설정 변경 함수까지 연결되어 있다.

## 6. KPM 값 점검 결과

현재 GUI와 InfluxDB에서 확인한 주요 KPM 상태는 다음과 같다.

| KPM | GUI 표시 | 상태 | 비고 |
| --- | --- | --- | --- |
| `DRB.UEThpDl.UEID` | `TP_Combined_PDCP_ENDC_kbps`, `TP_Combined_RLC_ENDC_kbps` | 수신됨 | 현재 ns-3 생성식 때문에 고정값 |
| `RRU.PrbUsedDl` | `RRU_PrbUsedDl`, `dlPrbUsage_percentage` | 수신됨 | UE 수 기반 생성식 때문에 고정값 |
| `Energy.TotalConsumption` | `Energy_TotalConsumption` | 수신됨 | TxPower 기반 계산값 |
| `RSRP.UEID` | `RSRP` | 수신됨 | IMSI 기반 placeholder 값 |
| `MeanActiveUEsDownlink` | `MeanActiveUEsDownlink` | 수신됨 | 현재 활성 UE 수 |

수정 후 확인된 예:

```text
UE 1 RSRP: -81
UE 2 RSRP: -82
UE 3 RSRP: -83
UE 4 RSRP: -84
UE 5 RSRP: -85

Cell 1 Energy_TotalConsumption: 3.162278
```

InfluxDB measurement도 생성된다.

```text
ue_1_rsrp.ueid
du-cell-1_energy.totalconsumption
```

## 7. THP가 고정으로 보이는 이유

throughput이 계속 `27472` 또는 `27473 kbps` 근처로 고정되는 것은 GUI 파서 문제가 아니다.

현재 ns-3 KPM 생성 코드가 실제 PDCP/RLC counter를 읽는 방식이 아니라, UE 수와 PRB 추정값으로 throughput을 만들어낸다.

현재 로직의 핵심:

```text
prbUsedRatio = min(1.0, totalUes * 30.0 / 273.0)
cellCapKbps  = prbUsedRatio * 100000.0 * 2.5
dlThpKbps    = cellCapKbps / totalUes
```

UE가 5개이면:

```text
prbUsedRatio = 5 * 30 / 273 = 약 0.55
dlThpKbps    = 약 27472.5 kbps
```

그래서 UE 수가 같으면 throughput과 PRB가 계속 같은 값으로 나온다.

현재 KPM의 성격은 다음처럼 설명하면 된다.

```text
KPM 전달 경로와 xApp 제어 루프 검증용 값은 정상적으로 흐른다.
다만 throughput, PRB, RSRP 일부는 실제 무선/트래픽 counter 기반 실측값이 아니라 ns-3 쪽에서 만든 placeholder 성격의 값이다.
```

## 8. Energy 값이 안 변하는 이유

Energy는 현재 TxPower 기반으로 계산된다.

```text
energy = 10^(txPowerDbm / 10) / 1000
```

기본 TxPower가 30 dBm이면:

```text
10^(30 / 10) / 1000 = 1 W
```

BWP 합산 구조에서는 현재 약 `3.162278`로 보인다.

그런데 현재 `--mode all` 정책에서는 상태가 `BUFFER_LIMITED`로 판단되어 RLC buffer control만 반복된다. TxPower control이 선택되지 않으면 TxPower가 바뀌지 않으므로 Energy도 고정으로 보인다.

Energy 변화를 확인하려면 `txp` 모드로 실행하거나 정책 threshold를 바꿔 TxPower 제어가 발생하도록 만들어야 한다.

예:

```bash
cd ~/flexric/build/examples/xApp/c/ctrl
./xapp_rc_rf_ctrl --mode txp --cell 1 --txp-init 30 --iterations 0
```

## 9. 아직 N/A로 남을 수 있는 값

아래 값들은 현재 시나리오에서 해당 원천 파일 또는 counter가 충분히 채워지지 않으면 `N/A`로 보일 수 있다.

```text
L3servingSINR_dB
L3neighSINR_dB
PDCP_PDU_Volume
PDCP_Throughput_kbps
Tx_Bytes
Cell_Average_Latency_ms_x_0_1
LTE_Cell_PDCP_Volume
```

이 값들이 `N/A`인 것은 KPM 전체 실패가 아니다. 현재 시나리오와 pusher가 주로 `du-cell-*.txt` 기반 KPM을 GUI에 올리고 있기 때문이다.

## 10. 시연할 때 설명 순서

1. `orange-rf-channel-reconfiguration.cc`가 테스트 시나리오라고 설명한다.
2. gNB/UE 수, center frequency, bandwidth, UE 배치 반경을 GUI에서 설정한다고 설명한다.
3. nearRT-RIC를 먼저 실행하고, GUI에서 ns-3 시나리오를 시작한다.
4. RF Control xApp을 켜면 KPM subscription이 시작된다고 설명한다.
5. xApp 로그에서 KPM 값과 `State: BUFFER_LIMITED`를 보여준다.
6. 이어서 `CONTROL-REQUEST tx`와 `CONTROL ACK rx`를 보여준다.
7. GUI에서 UE throughput, PRB, RSRP, Energy가 표시되는 것을 보여준다.
8. THP가 고정인 이유는 현재 KPM generator가 실측 counter가 아니라 UE 수 기반 계산식을 쓰기 때문이라고 설명한다.

## 11. 질문 받을 때 답변

Q. RIC control이 실제로 이뤄지나?

```text
그렇다. xApp 로그에서 CONTROL-REQUEST tx와 CONTROL ACK rx가 반복되고, ns-3 쪽에는 TxPower, RET, RLC buffer control을 적용하는 함수가 연결되어 있다.
```

Q. throughput이 왜 계속 같은가?

```text
현재 ns-3 KPM 생성부가 실제 트래픽 counter가 아니라 UE 수 기반 공식으로 throughput을 만들기 때문이다. UE 수가 5로 고정이면 약 27472 kbps가 반복된다.
```

Q. 다른 KPM은 잘 나오나?

```text
DRB throughput, PRB, Energy, RSRP, active UE 수는 GUI/InfluxDB까지 올라온다. 다만 throughput, PRB, RSRP는 현재 placeholder 성격이고, Energy는 TxPower가 바뀔 때 의미 있게 변한다.
```

Q. 그러면 현재 결과의 의미는 무엇인가?

```text
오늘 결과는 실제 최적화 성능 평가라기보다, ns-3 E2 Agent, nearRT-RIC, xApp, GUI가 연결되고 RC Control loop가 동작한다는 검증이다.
```

## 12. 다음 개선 방향

성능 평가용으로 만들려면 KPM 생성부를 실제 counter 기반으로 바꿔야 한다.

우선순위는 다음과 같다.

```text
1. DRB.UEThpDl.UEID를 실제 PDCP/RLC byte counter 기반으로 계산
2. RRU.PrbUsedDl을 실제 scheduler PRB 사용량 기반으로 계산
3. RSRP.UEID를 실제 channel/PHY 측정값 기반으로 연결
4. TxPower/RET 변경 후 throughput, RSRP, PRB가 동적으로 변하는지 검증
```

현재 버전은 제어 루프 시연용으로는 동작하지만, 논문/성능평가용 수치로 쓰기에는 KPM 생성부 보강이 필요하다.
