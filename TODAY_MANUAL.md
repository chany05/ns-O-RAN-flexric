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

기본 threshold와 제어 범위는 다음과 같다.

| 항목 | 기본값 | CLI 옵션 | 의미 |
| --- | ---: | --- | --- |
| Throughput target | `50000 kbps` | `--thp-target` | 평균 DL throughput 목표 |
| PRB high threshold | `0.80` | `--prb-high` | PRB 사용률이 높은 상태를 판단하는 기준 |
| Energy high threshold | `100.0` | `--energy-high` | energy waste 판단 기준 |
| Edge RSRP threshold | `-95 dBm` | `--rsrp-edge` | 이 값보다 RSRP가 낮으면 edge UE로 분류 |
| Edge ratio high | `0.50` | 코드 기본값 | 전체 UE 중 edge UE 비율이 높은 상태 |
| Edge ratio low | `0.20` | 코드 기본값 | edge UE 비율이 낮은 상태 |
| Control period | `5 sec` | `--period` | 정책 판단 주기 |
| KPM report period | `1000 ms` | 코드 기본값 | KPM subscription 주기 |
| Cooldown | `2 cycles` | 코드 기본값 | 제어 후 2번의 control period 동안 관찰만 수행 |

제어 파라미터 범위와 step은 다음과 같다.

| 제어 대상 | 범위 | 변경량 | 초기값 |
| --- | ---: | ---: | ---: |
| TxPower | `20 ~ 46 dBm` | `2 dB` | `30 dBm` |
| RET tilt | `-15 ~ 0 deg` | `1 deg` | `-6 deg` |
| RLC buffer | `1 ~ 64 MB` | `x2` 또는 `/2` | `10 MB` |

정책 판단에 쓰는 집계값은 다음과 같다.

```text
avg_dl_thp       = 유효 UE의 DL throughput 평균
prb_used_ratio   = RRU.PrbUsedDl
energy           = Energy.TotalConsumption
edge_ue          = RSRP < -95 dBm 인 UE
edge_ratio       = edge_ue / total_ue
edge_avg_thp     = edge UE들의 DL throughput 평균
```

`--mode all`에서의 상태 분류는 아래 순서대로 평가된다. 먼저 만족한 조건 하나만 선택하고, 한 control cycle에 하나의 control만 보낸다.

| 상태 | 조건 | 제어 |
| --- | --- | --- |
| `COVERAGE_LIMITED` | `edge_ratio > 0.50` 그리고 `edge_avg_thp < 50000` | RET tilt `+1 deg` |
| `CAPACITY_LIMITED` | `avg_dl_thp < 50000` 그리고 `prb_used_ratio > 0.80` | TxPower `+2 dB` |
| `BUFFER_LIMITED` | `avg_dl_thp < 50000` 그리고 `prb_used_ratio < 0.80` | RLC buffer `x2` |
| `ENERGY_WASTE` | `avg_dl_thp >= 50000` 그리고 `energy > 100.0` | TxPower `-2 dB` |
| `STABLE` | 위 조건에 해당하지 않음 | 제어 없음 |

개별 모드의 정책은 다음과 같다.

| 모드 | 조건 | 동작 |
| --- | --- | --- |
| `txp` | `avg_dl_thp < 50000` 그리고 `prb_used_ratio > 0.80` | TxPower `+2 dB` |
| `txp` | `avg_dl_thp >= 50000` 그리고 `energy > 100.0` | TxPower `-2 dB` |
| `ret` | `edge_ratio > 0.50` 그리고 `edge_avg_thp < 50000` | RET `+1 deg` |
| `ret` | `edge_ratio < 0.20` 그리고 `avg_dl_thp < 50000` | RET `-1 deg` |
| `rlc-buffer` | `avg_dl_thp < 50000` 그리고 `prb_used_ratio < 0.80` | RLC buffer `x2` |
| `rlc-buffer` | `avg_dl_thp >= 50000` 그리고 `prb_used_ratio > 0.80` | RLC buffer `/2` |
| `txp-ret` | `edge_ratio > 0.50`이면 RET 정책 우선 | RET 조건 만족 시 RET 제어 |
| `txp-ret` | RET 조건이 아니고 `avg_dl_thp < 50000`, `prb_used_ratio > 0.80` | TxPower `+2 dB` |
| `txp-ret` | `avg_dl_thp >= 50000`, `energy > 100.0` | TxPower `-2 dB` |

기존 기본 실험 로그에서는 아래처럼 판단됐다.

```text
avg_thp = 약 27473 kbps
prb     = 약 0.55
energy  = 약 3.2
UEs     = 5
state   = BUFFER_LIMITED
```

현재 throughput/PRB 계산을 counter/scheduler 기반으로 바꿨기 때문에 실제 값은 트래픽과 scheduler allocation에 따라 달라질 수 있다. 다만 아래 예처럼 `avg_dl_thp < target`이고 `prb < 0.80`이면 `--mode all`에서는 RLC buffer control이 선택된다.

위 값으로 상태를 계산하면 다음과 같다.

```text
avg_thp < 50000       true
prb 0.55 > 0.80       false
prb 0.55 < 0.80       true
energy 3.2 > 100.0    false

결과: BUFFER_LIMITED
동작: RLC buffer x2
```

정상 동작 로그 예:

```text
[xApp RF] === Iteration #N ===
KPM: avg_thp=<counter 기반 값> kbps, prb=<scheduler 기반 값>, energy=3.2, UEs=5
State: BUFFER_LIMITED 또는 현재 KPM 조건에 맞는 상태
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
| `DRB.UEThpDl.UEID` | `TP_Combined_PDCP_ENDC_kbps`, `TP_Combined_RLC_ENDC_kbps` | 수신됨 | NR RLC byte counter 기반 계산 |
| `RRU.PrbUsedDl` | `RRU_PrbUsedDl`, `dlPrbUsage_percentage` | 수신됨 | NR MAC scheduler의 실제 DL RB-symbol allocation 기반 계산 |
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

## 7. THP/PRB 계산 방식

기존에는 throughput과 PRB가 UE 수 기반 placeholder라서 UE 수가 같으면 값이 고정됐다.

기존 로직의 핵심:

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

현재는 다음처럼 보강했다.

```text
DRB.UEThpDl.UEID = reporting period 동안 NR RLC가 전송한 byte counter / period
RRU.PrbUsedDl    = NR MAC scheduler가 실제 배정한 DL RB-symbol / 관측된 DL RB-symbol budget
RRU.PrbUsedDl.UEID = 해당 RNTI가 실제 배정받은 DL RB-symbol / cell DL RB-symbol budget
```

따라서 PRB는 더 이상 단순 UE 수 기반 값이 아니고, scheduler allocation이 바뀌면 같이 변해야 한다.

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
8. THP와 PRB는 UE 수 기반 placeholder에서 counter/scheduler 기반으로 보강했다고 설명한다.

## 11. 질문 받을 때 답변

Q. RIC control이 실제로 이뤄지나?

```text
그렇다. xApp 로그에서 CONTROL-REQUEST tx와 CONTROL ACK rx가 반복되고, ns-3 쪽에는 TxPower, RET, RLC buffer control을 적용하는 함수가 연결되어 있다.
```

Q. throughput이 왜 계속 같은가?

```text
기존에는 UE 수 기반 공식이라 고정됐다. 현재는 NR RLC byte counter 기반으로 바꿨기 때문에 트래픽이 실제로 변하면 throughput도 변해야 한다.
```

Q. 다른 KPM은 잘 나오나?

```text
DRB throughput, PRB, Energy, RSRP, active UE 수는 GUI/InfluxDB까지 올라온다. 현재 throughput은 RLC counter 기반, PRB는 MAC scheduler allocation 기반이고, RSRP는 아직 placeholder 성격이다. Energy는 TxPower가 바뀔 때 의미 있게 변한다.
```

Q. 그러면 현재 결과의 의미는 무엇인가?

```text
오늘 결과는 실제 최적화 성능 평가라기보다, ns-3 E2 Agent, nearRT-RIC, xApp, GUI가 연결되고 RC Control loop가 동작한다는 검증이다.
```

## 12. 다음 개선 방향

성능 평가용으로 만들려면 KPM 생성부를 실제 counter 기반으로 바꿔야 한다.

우선순위는 다음과 같다.

```text
1. RSRP.UEID를 실제 channel/PHY 측정값 기반으로 연결
2. TxPower/RET 변경 후 throughput, RSRP, PRB가 동적으로 변하는지 검증
3. PDCP 기반 throughput counter를 별도로 연결
4. Energy model을 TxPower 단순 환산이 아니라 실제 radio energy model과 연결
```

현재 버전은 제어 루프 시연용으로 동작하고, throughput/PRB는 이전 placeholder보다 실제 counter에 가까워졌다. 다만 RSRP와 energy는 논문/성능평가용 수치로 쓰기 전에 추가 보강이 필요하다.
