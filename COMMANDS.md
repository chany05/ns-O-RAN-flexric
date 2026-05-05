# ns-O-RAN-flexric 실행 매뉴얼

## 사전 조건

- ns-3 NR 빌드 완료 (`./ns3 build`)
- FlexRIC 빌드 완료 (`cd flexric/build && make`)
- Docker 실행 중 (GUI용)

---

## 1. GUI (Docker) 시작/종료

```bash
# 시작
cd ~/ns-O-RAN-flexric/mmwave-LENA-oran/GUI
docker compose up -d

# 종료
docker compose down

# 상태 확인
docker compose ps
```

- Grafana: http://localhost:3000 (admin/admin)
- GUI Dashboard: http://localhost:8000
- InfluxDB: http://localhost:8086

---

## 2. 전체 제어 루프 실행 (순서 중요!)

### Step 1: nearRT-RIC 시작

```bash
cd ~/flexric/build
./examples/ric/nearRT-RIC
```

### Step 2: ns-3 시뮬레이션 시작 (RIC가 준비된 후 3초 대기)

```bash
cd ~/ns-O-RAN-flexric/mmwave-LENA-oran
./ns3 run orange-rf-channel-reconfiguration
```

### Step 3: xApp 시작 (ns-3가 E2 연결된 후 ~10초 대기)

```bash
cd ~/flexric/build/examples/xApp/c/ctrl
./xapp_rc_rf_ctrl --mode all
```

---

## 3. xApp 모드 옵션

| 모드 | 설명 | 제어 파라미터 |
|------|------|--------------|
| `txp` | TX Power만 제어 | TxPower (20-46 dBm, step 2) |
| `ret` | RET Tilt만 제어 | RetTilt (-15~0 deg, step 1) |
| `rlc-buffer` | RLC Buffer만 제어 | RlcBufferSize (1MB-64MB, x2) |
| `txp-ret` | TXP + RET 동시 | TxPower + RetTilt |
| `all` | 전부 제어 | TxPower + RetTilt + RlcBufferSize |

### xApp 실행 예시

```bash
# 기본 (all 모드, 기본 파라미터)
./xapp_rc_rf_ctrl --mode all

# TXP만, 초기값 지정
./xapp_rc_rf_ctrl --mode txp --txp-init 35

# 커스텀 threshold
./xapp_rc_rf_ctrl --mode all --thp-target 80000 --prb-high 0.7 --energy-high 50
```

### xApp 전체 옵션

```
--mode <txp|ret|rlc-buffer|txp-ret|all>  제어 모드
--cell <id>              대상 셀 ID (기본: 1)
--txp-init <dBm>         초기 TXP (기본: 30)
--ret-init <deg>         초기 RET tilt (기본: -6)
--rlc-init <bytes>       초기 RLC buffer (기본: 10485760)
--thp-target <kbps>      throughput 목표 (기본: 50000)
--prb-high <ratio>       PRB 사용률 high threshold (기본: 0.80)
--energy-high <W>        energy high threshold (기본: 100)
--rsrp-edge <dBm>        edge UE 판정 RSRP (기본: -95)
--period <sec>           제어 주기 (기본: 5)
--max-iter <n>           최대 반복 (0=무한, 기본: 0)
```

---

## 4. 백그라운드 실행 (로그 포함)

```bash
# RIC
cd ~/flexric/build
./examples/ric/nearRT-RIC > /tmp/ric.log 2>&1 &

# ns-3
cd ~/ns-O-RAN-flexric/mmwave-LENA-oran
./ns3 run orange-rf-channel-reconfiguration > /tmp/ns3.log 2>&1 &

# xApp (line-buffered 출력)
cd ~/flexric/build/examples/xApp/c/ctrl
stdbuf -oL -eL ./xapp_rc_rf_ctrl --mode all > /tmp/xapp.log 2>&1 &
```

### 로그 모니터링

```bash
tail -f /tmp/ric.log     # RIC 로그
tail -f /tmp/ns3.log     # ns-3 로그
tail -f /tmp/xapp.log    # xApp 로그
```

### 주요 로그 메시지

```bash
# E2 연결 확인
grep "E2 SETUP-REQUEST" /tmp/ric.log

# KPM indication 전송 확인
grep "NR KPM Indication sent" /tmp/ns3.log

# RC Control 적용 확인
grep "RAN Control" /tmp/ns3.log

# xApp 정책 결정 확인
grep "State:\|Policy" /tmp/xapp.log
```

---

## 5. 전체 프로세스 종료

```bash
pkill -f nearRT-RIC
pkill -f "orange-rf-channel"
pkill -f xapp_rc_rf_ctrl
```

---

## 6. 빌드 명령어

```bash
# ns-3 전체 빌드
cd ~/ns-O-RAN-flexric/mmwave-LENA-oran
./ns3 build

# xApp만 빌드
cd ~/flexric/build
make xapp_rc_rf_ctrl

# RIC 빌드
cd ~/flexric/build
make nearRT-RIC
```

---

## 7. GUI xApp Trigger (REST API)

GUI에서 xApp을 제어할 때 사용하는 HTTP API:

```bash
# RF Control xApp 시작
curl -X POST http://localhost:8000/trigger-xapp \
  -H "Content-Type: application/json" \
  -d '{"type":"rf_ctrl","mode":"all","cell":1,"txp_init":30,"ret_init":-6,"rlc_init":10485760}'

# xApp 중지
curl -X POST http://localhost:8000/stop-xapp \
  -H "Content-Type: application/json" \
  -d '{"type":"rf_ctrl"}'
```

---

## 8. 정책 동작 요약

| 상태 | 조건 | 동작 |
|------|------|------|
| COVERAGE_LIMITED | edge_ratio > 0.5, edge_thp < target | RET++ (틸트 올림) |
| CAPACITY_LIMITED | prb > 0.8, thp < target | TXP++ (파워 올림) |
| BUFFER_LIMITED | thp < target, prb < 0.8 | RLC x2 (버퍼 증가) |
| ENERGY_WASTE | thp >= target, energy > high | TXP-- (파워 내림) |
| STABLE | 위 조건 해당 없음 | 유지 |

---

## 9. 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| xApp "Resending Setup Request" | RIC가 안 떠있음 | RIC 먼저 시작 |
| RIC "No global node connected" | ns-3 전에 xApp 연결 | ns-3 먼저 시작 후 xApp |
| ns-3 E2 연결 안됨 | enableE2FileLogging=true | 기본값 false 확인 |
| KPM 값 0 | KpmSubscriptionCallback 미호출 | xApp subscribe 확인 |
| SCTP_SHUTDOWN | E2Termination::Start 중복 호출 | DoInitialize에서 제거 |

---

## 10. GUI 데이터 파이프라인 (UE KPI 표시)

GUI에서 UE KPI를 보려면 host 측 trigger 서버가 필요:

```bash
# host에서 실행 (ns-3 디렉토리에서)
cd ~/ns-O-RAN-flexric/mmwave-LENA-oran

# gui_trigger.py: sim_data_pusher.py + stop_ns3.py 자동 시작
python3 gui_trigger.py &

# xApp trigger 서버 (RF Control/Handover xApp 시작/중지)
cd "GUI/FlexRIC xApp GUI trigger"
python3 xApp_trigger.py &
python3 stop_xApp.py &
```

데이터 흐름:
```
ns-3 (du-cell-*.txt) → sim_data_pusher.py → InfluxDB → GUI Dashboard
```

---

## 11. 실행 순서 한눈에 보기

```
1. docker compose up -d          (GUI - port 8000)
2. python3 gui_trigger.py &      (host trigger 서버 - port 38866)
3. cd "GUI/FlexRIC xApp GUI trigger" && python3 xApp_trigger.py & && python3 stop_xApp.py &
4. ./nearRT-RIC                  (RIC - port 36421)
5. [3초 대기]
6. ./ns3 run orange-rf-...       (ns-3 E2 node)
7. [10초 대기 - E2 연결]
8. ./xapp_rc_rf_ctrl --mode all  (xApp - KPM sub + RC ctrl)
   또는 GUI에서 "Start RF Ctrl xApp" 버튼 클릭
```
