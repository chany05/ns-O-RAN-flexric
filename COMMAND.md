# 시뮬레이션 명령어 정리

## 시뮬레이션 시작

```bash
# 1. GUI / Grafana / InfluxDB
cd ~/ns-O-RAN-flexric/mmwave-LENA-oran/GUI
docker compose up -d --build

# 2. ns-3 GUI trigger 서버
cd ~/ns-O-RAN-flexric/mmwave-LENA-oran
python3 gui_trigger.py

# 3. xApp GUI trigger 서버
cd ~/ns-O-RAN-flexric/mmwave-LENA-oran/GUI/FlexRIC\ xApp\ GUI\ trigger
python3 xApp_trigger.py

# 4. RIC
cd ~/flexric/build
./examples/ric/nearRT-RIC
```

그 다음 GUI:

```text
http://localhost:8000
```

순서:

```text
1. GUI에서 ns-3 Start
2. 10초 정도 대기
3. GUI에서 RF Ctrl xApp On
```

## 종료

```bash
# xApp 종료
pkill -f xapp_rc_rf_ctrl

# ns-3 종료
pkill -f "orange-rf-channel"
pkill -f "ns3.42-orange-rf-channel-reconfiguration"

# RIC 종료
pkill -f nearRT-RIC

# host trigger 서버 종료
pkill -f gui_trigger.py
pkill -f sim_data_pusher.py
pkill -f stop_ns3.py
pkill -f xApp_trigger.py
pkill -f stop_xApp.py

# GUI / Grafana / InfluxDB 종료
cd ~/ns-O-RAN-flexric/mmwave-LENA-oran/GUI
docker compose down
```

## 로그 분석

```bash
# 실행 중 프로세스 확인
pgrep -af 'nearRT-RIC|ns3.42|orange-rf-channel|xapp_rc_rf_ctrl|gui_trigger.py|xApp_trigger.py|sim_data_pusher.py|stop_ns3.py|stop_xApp.py'

# 포트 확인
ss -ltnp | grep -E '8000|3000|8086|38866|38867|38868|38869'

# Docker 상태
cd ~/ns-O-RAN-flexric/mmwave-LENA-oran/GUI
docker compose ps

# ns-3 로그
cd ~/ns-O-RAN-flexric/mmwave-LENA-oran
tail -f ns3_run.log

# 데이터 pusher 로그
tail -f sim_data_pusher.log

# xApp 로그
tail -f ~/ns-O-RAN-flexric/mmwave-LENA-oran/GUI/FlexRIC\ xApp\ GUI\ trigger/xapp_rc_rf_ctrl.log

# xApp trigger 로그
tail -f ~/ns-O-RAN-flexric/mmwave-LENA-oran/GUI/FlexRIC\ xApp\ GUI\ trigger/xApp_trigger.log

# 주요 이벤트 검색
grep "NR KPM Indication sent" ~/ns-O-RAN-flexric/mmwave-LENA-oran/ns3_run.log
grep "CONTROL-REQUEST tx" ~/ns-O-RAN-flexric/mmwave-LENA-oran/GUI/FlexRIC\ xApp\ GUI\ trigger/xapp_rc_rf_ctrl.log
grep "CONTROL ACK rx" ~/ns-O-RAN-flexric/mmwave-LENA-oran/GUI/FlexRIC\ xApp\ GUI\ trigger/xapp_rc_rf_ctrl.log
grep "State:" ~/ns-O-RAN-flexric/mmwave-LENA-oran/GUI/FlexRIC\ xApp\ GUI\ trigger/xapp_rc_rf_ctrl.log
grep "Policy" ~/ns-O-RAN-flexric/mmwave-LENA-oran/GUI/FlexRIC\ xApp\ GUI\ trigger/xapp_rc_rf_ctrl.log
```
