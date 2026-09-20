# 실험 전 레포트: LAB2-02 클록 분주

작성자: 상혁 (2025440084) / 작성일: 2026-09-20 / 소스 커밋: `<git rev-parse --short HEAD 결과 기입>` / workspace: `LAB1.code-workspace` (템플릿 v2.0.1) / OS: `<기입>` / Python: `<python --version 결과 기입>` / 시뮬레이터: Icarus Verilog `<iverilog -V 첫 줄 기입>`

> 이 레포트는 VS Code(Icarus) 시뮬레이션까지의 사전 검증이다. Vivado GUI와 실물 보드 결과는 실험 후 레포트([post](../post/post_report.md))에서 다룬다. 시각은 clk 상승 에지(5, 15, 25, … ns)의 1 ns 뒤, 즉 TB가 비교하는 시각으로 적었다.

## 목적과 예상 동작

입력 클록을 DIVISOR 배로 나눈 관찰용 출력 `divided`와, 다른 회로의 clock-enable로 쓰는 `tick`을 만든다. 분주비, 듀티, 위상, tick 위치를 계산하고 파형과 비교한다.

### 포트 (`clock_divider.v`의 `clock_divider`)

| 포트 | 방향 | 비트 폭 | 설명 |
|---|---|---|---|
| clk | in | 1 | 상승 에지 기준 클록 |
| rst | in | 1 | 동기 active-high 리셋 |
| divided | out (reg) | 1 | 관찰용 분주 출력(듀티 50 %) |
| tick | out (wire) | 1 | count==DIVISOR−1인 클록 동안 1. 리셋 중에는 0 |
| DIVISOR | parameter | – | 분주비. 2 이상의 짝수 |

최상위(`lab2_clock_divider.v`, `lab2_clock_divider`): `clk`(B6), `rst`(K4), `button`, `sw[7:0]`, `led[7:0]`. DIVISOR=2, 10, 50, 1000의 분주기 4개를 두고 `led = {3'b000, tick, divided, div50, div10, div2}`로 내보낸다. 버튼과 스위치는 분주비를 바꾸지 않는다.

### 동작 규칙과 경계 입력

- 규칙: count는 0부터 DIVISOR−1까지 세고 0으로 돌아온다. count==DIVISOR/2−1인 에지에서 divided←1, count==DIVISOR−1인 에지에서 divided←0. tick = !rst && (count==DIVISOR−1).
- 정상·경계: DIVISOR=10에서 리셋 해제 후 5번째 에지에 divided=1, 10번째 에지에 0(주기 10클록, 듀티 50 %). tick은 9번째와 10번째 에지 사이에 1이 되어 10번째 에지에서 소비된다. 30클록에 tick 3번. 최소 분주비 2는 매 클록 토글.
- 시간 기준: TB 클록은 10 ns 주기(상승 에지 5, 15, 25, … ns)이고 입력은 에지 1 ns 뒤에 바꾼다. 레지스터는 다음 상승 에지에서 갱신된다. 실제 보드 클록(1 kHz)은 시뮬레이션의 10 ns와 별개다.

## 소스와 테스트벤치

- 설계 top: `lab2_clock_divider` / 시뮬레이션 top: `tb_clock_divider`
- 소스: [`src/clock_divider.v`](../../src/clock_divider.v), [`src/input_frontend.v`](../../src/input_frontend.v), [`src/lab2_clock_divider.v`](../../src/lab2_clock_divider.v)
- 테스트벤치: [`sim/tb_clock_divider.sv`](../../sim/tb_clock_divider.sv)
- 제약: [`constraints/lab2_clock_divider.xdc`](../../constraints/lab2_clock_divider.xdc)
- 설정: [`simulation.json`](../../simulation.json) (sources 3개, testbench `sim/tb_clock_divider.sv`, simulation_top `tb_clock_divider`)

| 파일 | 역할 |
|---|---|
| `src/clock_divider.v` | 핵심 동작을 담은 코어 `clock_divider`. TB가 이 모듈을 직접 검사한다. |
| `src/input_frontend.v` | 버튼·스위치를 클록에 맞추는 입력 회로. 리셋 2단 해제 동기화, 버튼·스위치 2단 동기화 플립플롭, STABLE_CYCLES=20(1 kHz에서 20 ms) 안정 확인 뒤 한 클록짜리 `press` 펄스를 만든다. |
| `src/lab2_clock_divider.v` | 보드 top. 프런트엔드와 코어를 연결하고 LED로 출력한다. |
| `sim/tb_clock_divider.sv` | 입력 자극, 기대값 계산, 자동 비교(`check`), PASS/FAIL 출력, VCD 생성, watchdog. |
| `constraints/lab2_clock_divider.xdc` | 핀 번호·전압과 1 kHz 클록 정의. Icarus는 XDC를 읽지 않으므로 이 사전 시뮬레이션에는 사용되지 않는다. |
| `simulation.json` | VS Code 시뮬레이션 작업이 읽는 소스 목록·테스트벤치·시뮬레이션 top. |

### 테스트벤치 동작

- 자극 순서: DIVISOR=10(dut)과 DIVISOR=2(minimum)를 같은 클록으로 동시에 검사한다. 리셋 해제 후 30클록 동안 매 에지마다 divided·tick·div2·tick2를 비교하고, tick 펄스 수 3을 확인한다. 이어 7클록 뒤 rst=1(tick 마스킹, 주기 중간 리셋), 해제 후 4클록 low 유지와 5번째 에지의 첫 전이를 확인한다.
- 검사 횟수: 1(reset)+30×4(divided, tick, div2, tick2)+1(pulses)+1(reset masks tick)+1(reset mid period)+4(restart low half)+1(first transition) = 129
- 종료·watchdog: 마지막 검사 뒤 `finish` task가 `LAB2_PASS clock_divider checks=N`을 출력하고 `$finish`한다. 별도로 100000 ns(100 µs) 뒤에 `watchdog timeout`으로 `$fatal` 처리한다. 예상 종료 시각은 436 ns이다.
- 이 TB는 코어 `clock_divider`만 시험한다. 입력 동기화·디바운스와 실제 핀·타이밍이 통과했다는 뜻은 아니다.

### XDC 설명

`lab2_clock_divider.xdc`은 포트 이름을 `lab2_clock_divider.v`과 맞춰 핀을 지정한다. 모든 I/O는 `LVCMOS33`이고 `create_clock -name trainer_1khz -period 1000000.000 [get_ports clk]`로 주 클록을 1 kHz(주기 1,000,000 ns)로 정의하며 `set_false_path -from [get_ports {rst button sw[*]}]`로 비동기 입력을 타이밍 경로에서 제외한다.

| 포트 | 핀 | 보드 대응 |
|---|---|---|
| clk | B6 | 1 kHz 주 클록 |
| rst | K4 | 리셋 (active-high) |
| button | N8 | 스텝 버튼 |
| sw[7:0] | U4(sw[0]), V4, W1, W4, T1, U2, W3, Y1(sw[7]) | DIPSW8..DIPSW1 (DIPSW1..8 = sw[7]..sw[0]) |
| led[7:0] | N5(led[0]), M1, M3, M7, N7, M2, M4, L4(led[7]) | LED0..LED7 |

## VS Code 실행 과정

1. File → New Window → File → Open Workspace from File...로 `LAB1.code-workspace`를 연다. 확장(slang, VaporView, vscode-pdf)을 설치한다.
2. RTL·TB·XDC·`simulation.json`을 직접 입력하고 File → Save All.
3. Terminal → Run Task... → `01 Check tools`로 Git·Python·iverilog·vvp 버전을 확인한다.
4. `02 Simulate`를 실행해 `LAB2_PASS`와 종료 시각을 확인한다.
5. `03 Open waveform`으로 `build/sim/wave.vcd`를 VaporView로 연다. 신호: clk, rst, divided, tick, div2, tick2 (선택: dut.count).

정상 실행 로그(본인 로그로 교체하고 `evidence/pre/`에 복사):

```text
LAB2_PASS clock_divider checks=129
sim/tb_clock_divider.sv:19: $finish called at 436000 (1ps)
```

- 본인 실행 로그: [`../../evidence/pre/lab2_02_normal.log`](../../evidence/pre/lab2_02_normal.log)
- VCD: [`../../evidence/pre/lab2_02_wave_normal.vcd`](../../evidence/pre/lab2_02_wave_normal.vcd)
- 파형 캡처(VaporView): `evidence/pre/lab2_02_wave_full.png`(전체 Zoom Fit), `evidence/pre/lab2_02_wave_zoom.png`(상승 에지 확대)
- 오류: 첫 실행에서 발생한 오류가 있으면 첫 오류 → 수정 → 재실행 로그 순서로 기록한다. (없으면 "없음", Python 실행 경로를 고쳤다면 그 내용 기입)

### 사전 파형 해석

| 시간 구간 | 입력 | 예상 | 실제 파형 | 해석 |
|---|---|---|---|---|
| 6 ns | rst=1 | divided=0, tick=0, div2=0, tick2=0 | divided=0, tick=0, div2=0, tick2=0 | 동기 리셋 이후 모든 출력이 0. |
| 16 ns | i=1: 리셋 해제 후 첫 에지(15 ns) | divided=0, tick=0, div2=1, tick2=1, count=1 | divided=0, tick=0, div2=1, tick2=1, count=1 | DIVISOR=2는 첫 에지에서 divided가 1, tick2도 1(다음 에지에서 소비). |
| 26 ns | i=2 | divided=0, tick=0, div2=0, tick2=0, count=2 | divided=0, tick=0, div2=0, tick2=0, count=2 | div2는 매 클록 토글(주기 2클록). |
| 56 ns | i=5: 5번째 에지(55 ns) | divided=1, tick=0, div2=1, tick2=1, count=5 | divided=1, tick=0, div2=1, tick2=1, count=5 | DIVISOR/2−1=4에서 divided←1. 첫 전이는 55 ns. |
| 96 ns | i=9: 9번째 에지(95 ns) | divided=1, tick=1, div2=1, tick2=1, count=9 | divided=1, tick=1, div2=1, tick2=1, count=9 | count=9인 동안 tick=1(10번째 에지에서 소비될 enable). |
| 106 ns | i=10: 10번째 에지(105 ns) | divided=0, tick=0, div2=0, tick2=0, count=0 | divided=0, tick=0, div2=0, tick2=0, count=0 | 경계: count가 0으로 돌아오고 divided←0. 주기는 10클록(100 ns). |
| 306 ns | i=30 | divided=0, tick=0, div2=0, tick2=0, count=0 | divided=0, tick=0, div2=0, tick2=0, count=0 | 30클록 동안 tick이 i=9, 19, 29에서 총 3번(pulses=3). |
| 377 ns | 7클록 뒤 rst=1, 에지 전(377 ns) | divided=1, tick=0, div2=1, tick2=0 | divided=1, tick=0, div2=1, tick2=0 | 리셋이 걸리는 순간 tick2는 !rst 항 때문에 즉시 0(아직 에지 전이라 divided는 유지). |
| 386 ns | 리셋 에지 385 ns 뒤 | divided=0, tick=0, div2=0, tick2=0 | divided=0, tick=0, div2=0, tick2=0 | 주기 중간 리셋 시 divided도 0으로 복귀. |
| 436 ns | rst=0, 5번째 에지(435 ns) | divided=1, tick=0, div2=1, tick2=1 | divided=1, tick=0, div2=1, tick2=1 | 재시작 후에도 4클록 low 뒤 5번째 에지에서 첫 전이. |

표의 값은 상승 에지 직후 안정된 값이다. `LAB2_PASS`만 적지 않고 각 행에서 입력, 이전 상태, 다음 상태를 비교한다. 파형 캡처에서 위 시각을 확대해 본인 화면으로 확인한다.

## 코드 수정·실패·복구 실험

- 변경: divided가 1이 되는 조건을 `DIVISOR/2 - 1`에서 `DIVISOR/2`로 바꾼다(한 클록 늦춤).
- 변경한 파일과 위치: `src/clock_divider.v` 19행
- 테스트벤치 기대값은 바꾸지 않는다.

```diff
-    if (count == DIVISOR/2 - 1) divided <= 1'b1;
+    if (count == DIVISOR/2) divided <= 1'b1;
```

실행 전 계산: DIVISOR=2에서 원래는 count==0에서 divided←1이라 첫 에지(15 ns)에 div2=1이다. 변경 후에는 count==1에서야 1이 되므로 15 ns 에지 뒤에도 div2=0이다. TB는 16 ns에 `div2===(1%2==1)`을 검사하므로 이 시점에 실패한다. 같은 16 ns의 DIVISOR=10 쪽 divided·tick 검사는 아직 통과하고(첫 전이는 5번째 에지), 그 뒤에 나오는 minimum divisor duty 검사가 첫 실패가 된다.

| 단계 | 소스 커밋 또는 해시 | 실행 폴더·로그 링크 | 입력·기대값·실제값 | 해석 |
|---|---|---|---|---|
| 정상 코드 | `<커밋/해시 기입>` | [normal.log](../../evidence/pre/lab2_02_normal.log) | 16 ns 기대 div2=1, 실제 div2=1. `LAB2_PASS clock_divider checks=129` | 모든 검사 통과, 436 ns 종료. |
| 지정한 RTL 변경 | `<커밋/해시 기입>` | [mod.log](../../evidence/pre/lab2_02_mod.log) | 16 ns 기대 div2=1, 실제 div2=0. `LAB2_FAIL minimum divisor duty time=16000`, `FATAL: sim/tb_clock_divider.sv:12: check failed` | `minimum divisor duty` 검사가 변경을 발견했다(로그의 time은 ps 단위, 16000 ps = 16 ns). |
| 원래 코드로 복구 | `<커밋/해시 기입>` | [recover.log](../../evidence/pre/lab2_02_recover.log) | 복구 후 전체 검사 재실행. `LAB2_PASS clock_divider checks=129`, `$finish called at 436000 (1ps)` | PASS와 종료 시각이 정상 실행과 같고 새 VCD를 확인한다. |

- 첫 실패 이후에는 `$fatal`로 시뮬레이션이 끝나므로 뒤의 검사는 실행되지 않는다. 변경 전후 파형은 각각 별도 폴더에 보관한다.
- 문법 오류를 경험했다면 오류 위치로 이동한 화면, 원인, 수정 내용과 재실행 로그도 이 절에 연결한다.

## 보드 실험 계획

- 부품·프로젝트: Vivado 2026.1, RTL Project `lab2_clock_divider`, 부품 `xc7s75fgga484-1`(정확히 -1). Design Sources: `clock_divider.v`, `input_frontend.v`, `lab2_clock_divider.v`(Copy sources 해제). Simulation Sources: `tb_clock_divider.sv`(Set as Top: `tb_clock_divider`). Constraints: `lab2_clock_divider.xdc`. Project Summary의 Top module name은 `lab2_clock_divider`이다.
- 장비 클록·제약: Combo II-DLD S75 주 클록 B6을 1 kHz로 맞추고 XDC의 `trainer_1khz`(1,000,000 ns)와 일치시킨다.
- 입력: K4=리셋. N8과 DIP 스위치는 분주비를 바꾸지 않는다. 주 클록은 1 kHz.
- 출력: LED[0]=div2, LED[1]=div10, LED[2]=div50, LED[3]=divided(DIVISOR=1000), LED[4]=tick, LED[7:5]=0.

| 조작 | 예상 LED / 동작 |
|---|---|
| LED[0] | 500 Hz (파형·측정 장비로 확인) |
| LED[1] | 100 Hz (파형·측정 장비) |
| LED[2] | 20 Hz (주기 비교) |
| LED[3] | 1 Hz, 0.5초 켜짐/0.5초 꺼짐 |
| LED[4] | 1 ms 폭 tick (눈으로는 거의 안 보임) |

500 Hz, 100 Hz, 20 Hz는 눈에 깜빡임이 구분되지 않으므로 밝기 차이로 단정하지 않고 측정 장비나 파형으로 비교한다. 시뮬레이션 클록(10 ns)과 보드 클록(1 kHz)은 별개다.

촬영할 장면: 보드 전체(배선·입력·출력이 함께 보이는 사진)와 위 표의 조작별 LED 상태 사진·영상. 예상되는 차이: 버튼·스위치는 동기화·안정 확인 지연이 있고, 빠른 신호는 눈으로 구분되지 않을 수 있다.

이 단계에서는 Vivado GUI와 실물 보드의 결과를 수행한 것처럼 기록하지 않는다. 합성·구현·bit 생성과 실제 장치 기록은 실험 후 레포트에서 다룬다.
