---
title: 드론 방사선 측정·위치 전송
nav: 드론 방사선 측정·위치 전송
description: 드론에 단 GM 계수기의 CPM과 GPS 위경도를 LTE로 서버에 보내 OpenStreetMap 지도와 Isaac Sim에 색으로 표시한 기록.
---

!!! summary "한 줄 요약"
    드론에 **GM 계수기**(방사선 계수)와 **SIM7000E**(LTE + GPS) 모듈을 달아, 분당 계수(CPM)와 위경도를 LTE로 서버에 보냈다. 서버 쪽 PyQt5 프로그램이 받은 값을 **OpenStreetMap 지도**에 색(CPM 낮음 파랑 → 높음 빨강)으로 찍는다. 같은 데이터를 **Isaac Sim**에서 연구실 건물 모델 위에 3D 구로 띄우는 시각화도 만들었다.

## 측정 장치

| 구성 | 내용 |
|---|---|
| 센서 | GM 계수기: 방사선이 들어올 때마다 펄스 하나 |
| 컨트롤러 | Arduino Uno: 펄스를 외부 인터럽트(2번 핀, 하강 에지)로 셈 |
| 통신·위치 | DFRobot SIM7000E: LTE Cat-M1 데이터 통신 + GNSS 위치, SoftwareSerial 19200 bps |
| 케이스 | 3D 프린팅 2층 구조: 1층 GM 계수기, 2층 LTE 모듈과 배터리, 위에 커버 |

<div class="gallery">
<figure><img src="assets/drone-case-1f.png" alt="드론 케이스 1층 모델"><figcaption>1층: GM 계수기</figcaption></figure>
<figure><img src="assets/drone-case-2f.png" alt="드론 케이스 2층 모델"><figcaption>2층: LTE 모듈 + 배터리</figcaption></figure>
<figure><img src="assets/drone-case-cover.png" alt="드론 케이스 커버 모델"><figcaption>커버</figcaption></figure>
<figure><img src="assets/drone-case.png" alt="드론 케이스 조립 모델"><figcaption>조립</figcaption></figure>
</div>

## 데이터 흐름

| 단계 | 하는 일 |
|---|---|
| 1. 계수 | 인터럽트로 펄스를 세고, 1초마다 60칸짜리 이동 창에 넣어 최근 60초 합계를 CPM으로 씀 |
| 2. 위치 | SIM7000E에 `AT+CGNSINF`로 GPS 위경도를 물어봄 |
| 3. 전송 | 새 펄스가 있으면 `cpm52.18lat36.425464lon127.367106` 같은 한 줄 문자열을 LTE TCP로 보냄 |
| 4. 수신 | 고정 IP 장비에 포트포워딩을 걸고, 그 IP·포트로 PC의 TCP 서버에 연결 |
| 5. 표시 | PyQt5 프로그램이 `cpm`·`lat`·`lon` 글자 위치로 값을 잘라 내고, folium으로 지도에 원을 찍어 HTML로 저장 |

![Radiation Map 프로그램](assets/radiation-gui.png)
*수신 프로그램. IP·포트를 넣고 Connect를 누르면 받은 CPM·위도·경도가 표시되고, Show Map으로 지도를 연다.*

![지도에 찍은 측정 위치](assets/radiation-map.png){ width="620" }
*LTE로 받은 위치를 OpenStreetMap에 찍은 결과. 점 색이 CPM을 나타낸다.*

장비 없이 시험할 때는 CPM(40–80)과 연구실 주변 위경도를 무작위로 만들어 1초마다 보내는 송신 스크립트를 썼다.

!!! issue "GPS 위경도의 소수점 자리가 잘림"
    **문제** 받은 위경도의 자릿수가 모자라, 드론이 수십 m 움직여도 지도에서는 같은 점으로 찍혔다.

    **원인** DFRobot SIM7000 라이브러리의 `getPosition()`이 응답에서 위도·경도를 **7글자**씩만 복사했다. 위도는 `36.4254`(소수점 넷째 자리, 약 10 m), 경도는 `127.370`(셋째 자리, 약 90 m)까지만 남았다.

    **해결** 복사 길이를 위도 9글자, 경도 10글자로 늘렸다(소수점 여섯째 자리, 약 0.1 m). 헤더의 `_latitude[10]`, `_longitude[10]` 버퍼도 끝의 널 문자까지 들어가도록 키웠다.

!!! issue "위치를 못 잡을 때 (0, 0)이 찍힘"
    **문제** 켜자마자나 위성이 안 잡힐 때 위도 0, 경도 0인 데이터가 들어왔다. 그대로 찍으면 지도 범위가 아프리카 앞바다까지 넓어진다.

    **원인** GPS가 위치를 잡기 전에는 라이브러리가 위경도 대신 `"0"`을 돌려준다.

    **해결** 수신 프로그램에서 위도와 경도가 모두 0인 데이터는 지도에 찍지 않고 버렸다.

## Isaac Sim 시각화

측정점을 실제 공간에 겹쳐 보려고, 연구실 건물을 모델링해 Isaac Sim 4.5.0(Ubuntu 22.04)에 띄우고 받은 데이터를 3D 구로 표시했다.

1. **건물 모델:** SolidWorks에서 건물을 모델링하고 [sw_urdf_exporter](https://wiki.ros.org/sw_urdf_exporter)로 URDF로 내보낸 뒤 USD로 불러왔다. URDF에 색까지 넣으려면 면이 아니라 형상 전체에 색을 지정해야 했다.
2. **데이터 수신:** 시뮬레이터 안에서 TCP 서버(포트 5050)를 열고 같은 형식(`cpm…lat…lon…`)의 문자열을 받는다.
3. **좌표 변환:** 연구실 위치를 원점으로 두고 위경도를 동·북 방향 미터 거리로 바꾼다. 고도는 측정하지 않아 10–30 m 사이 무작위 값을 썼다.
4. **표시:** 그 위치에 반투명 구를 만들고, CPM 40이면 파랑, 80이면 빨강, 그 사이는 섞은 색으로 칠한다.

![연구실 건물 모델](assets/lab-building.png){ width="520" }
*SolidWorks로 만든 연구실 건물 모델.*

<div class="gallery">
<figure><img src="assets/isaac-cpm.png" alt="Isaac Sim에서 건물 위에 떠 있는 CPM 구들"><figcaption>건물 위에 띄운 측정점 (시험용 무작위 데이터)</figcaption></figure>
<figure><img src="assets/isaac-cpm-top.png" alt="위에서 본 CPM 구 분포"><figcaption>위에서 본 모습</figcaption></figure>
</div>

!!! issue "데이터를 받는 동안 시뮬레이션 화면이 멈춤"
    **문제** 첫 버전은 송신 쪽이 연결해 있는 동안 시뮬레이션 화면이 갱신되지 않았다.

    **원인** 소켓 서버가 메인 흐름에서 돌아서, 연결이 끊길 때까지 `app.update()`를 부르는 시뮬레이션 루프로 넘어가지 못했다.

    **해결** 소켓 수신을 별도 스레드로 옮기고, 받은 값은 큐에 넣기만 했다. 구 생성은 Isaac Sim API를 메인 스레드에서 불러야 하므로 시뮬레이션 루프가 매 프레임 큐를 비우며 처리한다(cpmv2).

## 조이스틱 조종과 Genesis

- **조이스틱:** ROS 2 Humble의 `joy` 노드로 Xbox 360 패드를 읽고, 직접 만든 `joystick_control` 노드가 축 값을 `/cmd_vel`(전후·좌우·상하·yaw·roll·pitch)로, 버튼 하나를 `/reset`으로 바꿔 시뮬레이터의 드론 컨트롤러에 보낸다. 버튼 번호는 직접 눌러 보며 맞췄다.
- **Genesis:** Genesis 시뮬레이터의 드론 호버링 예제로 강화학습(PPO, 환경 8192개, 300회 반복)을 돌리고, 학습된 정책 파일(.pt)로 드론을 움직여 봤다.

![조이스틱 ROS 2 노드 그래프](assets/joystick-graph.png)
*조이스틱 입력이 드론 컨트롤러까지 가는 ROS 2 노드 그래프 (rqt_graph).*

??? 원리: CPM과 이동 창
    GM 계수기는 방사선이 관을 지날 때마다 펄스를 하나 낸다. 1분 동안 센 펄스 수가 CPM(counts per minute)이다.

    1분을 다 기다렸다 한 번 내보내면 값이 1분에 한 번만 바뀐다. 그래서 1초마다 그 1초 동안의 펄스 수를 60칸짜리 원형 배열에 넣고, 가장 오래된 칸을 빼고 새 칸을 더해 **최근 60초 합계**를 1초마다 갱신했다. 드론이 움직이는 동안에도 값이 매초 바뀐다.

??? 원리: 위경도를 미터 좌표로 바꾸기
    위경도는 각도라서 시뮬레이터에 바로 쓸 수 없다. 기준점(원점)에서 동쪽·북쪽·위쪽으로 몇 m인지(ENU 좌표)로 바꿔야 한다.

    | 좌표계 | 설명 | 단위 |
    |---|---|---|
    | Geodetic | 위도, 경도, 고도 | 도, m |
    | ECEF | 지구 중심을 원점으로 한 3D 직교 좌표 | m |
    | ENU | 기준점에서 동(E)·북(N)·상(U) 방향 로컬 좌표 | m |

    **첫 버전:** 위경도·고도 → ECEF → ENU. WGS84 타원체에서 위도 방향 곡률 반경 N = a / √(1 − e² sin²φ)를 구해 ECEF 좌표를 만들고, 기준점과의 차이 벡터를 기준점 위경도로 만든 회전 행렬로 동·북·상 방향에 맞춰 돌린다.

    **cpmv2:** 측정 범위가 수백 m 안이라 지구 곡률을 거의 무시할 수 있다. 하버사인 공식으로 원점에서 같은 위도로 동쪽까지, 같은 경도로 북쪽까지의 거리를 따로 구해 x, y로 썼다. 계산이 단순하고 결과는 거의 같다.
