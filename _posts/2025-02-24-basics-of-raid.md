---
title: RAID의 기본 이해하기
date: 2025-02-24 15:18:52 +/-TTTT
categories: [Linux]
tags: [CS, Raid]
---


### 들어가기 앞서

온프레미스 서버의 시스템 로그 파일(/var/log/messages)에서 아래와 같은 오류를 발견했습니다.

```bash
smartd[1902]: Device: /dev/bus/0 [megaraid_disk_01], SMART Failure: DATA CHANNEL IMPENDING FAILURE GENERAL HARD DRIVE FAILURE
```

`df -h` 명령어로 시스템에 마운트된 디스크들을 확인해보았으나, 오류가 발생한 디스크가 목록에 없었습니다.
RAID 컨트롤러를 확인해본 결과 로그에 출력된 것처럼, RAID 구성의 1번 디스크에서 발생한 문제로 확인되었습니다.

오류를 접하기 전까지는 RAID가 무엇인지 정확하게 몰랐기에, 해당 포스트에서 학습한 내용을 정리합니다.


## RAID 란

`Redundant Array of Independent Disks`로,  여러 개의 하드디스크를 하나의 디스크처럼 사용하는 기술입니다.

RAID의 주요 목적은 아래와 같습니다.

- 데이터 안정성 향상 - 하나의 디스크가 고장나도 데이터를 보호 (RAID 1, RAID 5, RAID 10 등)
- 성능 개선 - 여러 디스크에 동시에 읽고 쓸 수 있어 속도가 향상 (RAID 0, RAID 5, RAID 6, RAID 10 등)

흔히 사용되는 RAID LEVEL은 RAID 0 / RAID 1 / RAID 5 / RAID 6 / RAID 10이 있습니다.

## Parity 란

데이터 오류를 검출하고 복구하는데 사용되는 추가적인 정보입니다.

RAID에서는 이 패리티 정보를 사용해 디스크가 고장났을 때 데이터를 복구할 수 있습니다.

## RAID의 종류

### RAID 0 (Striping)

![raid0.jpg](/assets/img/basics-of-raid/raid0.jpg)

- 데이터를 여러 디스크에 분산 저장하여 읽기/쓰기 속도가 향상됩니다.
- 최소 2개의 디스크가 필요합니다.
- 속도는 빠르지만 하나의 디스크 고장 시 모든 데이터가 손실될 수 있어 안정성이 낮습니다.
- 사용 가능한 총 용량은 모든 디스크의 합과 같습니다.

### RAID 1(Mirroring)

![raid1.jpg](/assets/img/basics-of-raid/raid1.jpg)

- 데이터를 항상 쌍으로 된 디스크에 동일하게 저장합니다.
- 최소 2개의 디스크가 필요합니다.
- 하나의 디스크가 고장 나도 데이터 손실이 없어 안정성이 높습니다.
- 디스크를 2개씩 쌍으로 묶어서 미러링하므로, 실제 사용 가능 용량은 전체 디스크 용량의 50%입니다.

### RAID 5

![raid5.jpg](/assets/img/basics-of-raid/raid5.jpg)

- 제일 사용 빈도가 높은 RAID Level입니다.
- Block 단위로 Striping을 수행합니다.
- 패리티 정보는 모든 디스크에 분산되어 저장됩니다.
- 최소 3개의 디스크로 구성이 가능합니다.
- 하나의 디스크 고장 시 데이터 복구가 가능합니다.
- 사용 가능한 총 용량은 (전체 디스크 수 - 1) × 디스크 용량입니다.
- 쓰기 성능은 패리티 계산으로 인해 다소 저하될 수 있습니다.
- 2개 이상의 디스크 고장 시 데이터가 손실됩니다.

### RAID 6

![raid6.jpg](/assets/img/basics-of-raid/raid6.jpg)

- RAID 5를 강화한 버전입니다.
- Block 단위로 striping을 수행합니다.
- 패리티를 2개의 디스크에 분산 저장합니다.
- 최소 4개의 디스크로 구성이 가능합니다.
- 사용 가능한 총 용량은 (전체 디스크 수 - 2) × 디스크 용량입니다.
- 2개의 디스크 에러 시에도 복구가 가능합니다.

### RAID 1 + 0(RAID 10)

![raid10.jpg](/assets/img/basics-of-raid/raid10.jpg)

- RAID 1과 RAID 0을 혼용한 방식입니다.
- 최소 4개의 디스크가 필요합니다.
- 먼저 Mirroring한 후 Striping을 수행합니다.
- 미러링으로 묶인 하드를 통해 손실된 데이터만 복원이 가능합니다.
- 높은 안정성과 성능을 동시에 제공합니다.
- 사용 가능한 총 용량은 전체 디스크 용량의 50%입니다.

<br>

---
참고 자료
- [RAID 정리 1. RAID 기본 설명 및 RAID Level (레이드 레벨)](https://devocean.sk.com/blog/techBoardDetail.do?ID=163608)
