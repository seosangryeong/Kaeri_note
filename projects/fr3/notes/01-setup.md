---
title: 환경 구축과 첫 실물 이동
nav: 환경 구축과 첫 실물 이동
date: 기록 1
---

!!! summary "한 줄 요약"
    로봇과 PC를 연결하고 ROS 2 제어 스택을 빌드해서, RViz의 Plan → Execute로 로봇을 처음 움직였다. 정해진 HOME 자세로 보내는 `move_home`도 만들었다.

## 한 일

- Ubuntu 22.04 + ROS 2 Humble에 franka_ros2 v2.0.2, libfranka 0.15.0 빌드
- 로봇 쪽 FCI(외부 제어 인터페이스) 켜기, 실시간 스케줄링 권한 설정
- 실행 순서 정리: bringup → MoveIt → 팔 컨트롤러 로드(평소엔 꺼 둠) → 움직일 때만 켜고 끝나면 끄기
- `move_home`: 공식 SRDF의 "ready" 자세로 보내는 첫 스킬
- 새 노트북(NUC)으로 옮겨 같은 과정을 처음부터 다시 재현

## 트러블슈팅

!!! issue "빌드가 실패함"
    **문제** eigenpy, pinocchio 같은 패키지가 빌드되지 않았다.

    **원인** conda가 자동으로 켜져서 시스템 python 대신 numpy 없는 conda python3가 잡혔다. 예전 빌드 캐시에도 그 경로가 남아 있었다.

    **해결** build·install·log를 지우고 `-DPYTHON_EXECUTABLE=/usr/bin/python3`를 명시해 다시 빌드했다.

!!! issue "실시간 우선순위가 안 걸림"
    **문제** limits.d에 rtprio 99를 넣었는데도 실제로는 0이었다.

    **원인** GNOME 세션이 systemd 사용자 서비스 밑에서 돌고, 그 서비스 자체가 `LimitRTPRIO=0`으로 고정돼 있었다.

    **해결** systemd 드롭인 설정으로 서비스의 제한을 풀었다.

!!! issue "Plan은 되는데 로봇이 안 움직임"
    **문제** RViz에서 Plan → Execute를 눌러도 실물이 그대로였다.

    **원인** MoveIt 실행 설정이 꺼져 있었고, 팔 컨트롤러가 활성화되지 않은 상태였다.

    **해결** MoveIt을 `execute:=true`로 띄우고, 움직이기 직전에만 컨트롤러를 켜도록 순서를 바꿨다.

## 결과

실물 Plan/Execute 성공, `move_home` 실물 검증. 이후 모든 이동은 "계획을 먼저 보여 주고, `--execute`를 붙여야만 움직인다"는 원칙으로 만들었다.

## 원리

??? ROS 2에서 로봇팔이 움직이기까지
    명령 한 번에 여러 프로그램이 순서대로 관여한다.

    | 단계 | 하는 일 |
    |---|---|
    | bringup (ros2_control) | 로봇과 1 kHz로 통신하며 관절 상태를 읽고 명령을 보낸다 |
    | 팔 컨트롤러 | 받은 궤적(시간별 관절 목표)을 따라가도록 매 순간 토크를 계산한다 |
    | MoveIt | 목표까지 충돌 없는 궤적을 계획하고, 컨트롤러에 넘긴다 |
    | 내 스크립트 | MoveIt에 목표를 주고, 컨트롤러를 켜고 끈다 |

    컨트롤러를 평소에 꺼 두는 이유는 안전 때문이다. 꺼져 있으면 실수로 명령이 가도 팔이 따라가지 않는다.
