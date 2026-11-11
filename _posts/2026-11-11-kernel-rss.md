---
title: "RSS"
description: "RSS의 의미와 공유 라이브러리에 따른 프로세스 메모리 집계 특성을 정리한다."
date: 2026-11-11 21:55:58 +0900
categories: [Operating System, Kernel]
tags: [linux, kernel]
permalink: /operating-system/kernel/2026/11/11/rss/
---

RSS(Resident Set Size)란 실제로 물리 메모리에 할당된 프로세스의 page의 총량을 나타낸다. Swapped out된 메모리는 포함하지 않지만, shared library의 메모리는 포함한다. 프로세스에 의해 요구되는 공간이 RSS보다 크다면, 남은 부분은 swap영역에 저장된다.

해당 값이 프로세스에 의해 소비되는 실제 메모리 양을 정확히 나타낸다고 생각할 수 있지만, shared library와 dynamic linking library때문에 그렇지 않다. Shared library는 다수개의 프로세스에 의해 공유되지만, 그들은 물리 메모리에 한 번만 load되고 각각의 프로세스에 map된다. 그러나 프로세스의 RSS를 계산할 때는 이러한 사실을 고려하지 않고 shared library 메모리를 포함하여 계산한다. 

이는 프로세스가 실제로 그 부분을 사용하기에 틀린 것이 아니다. 따라서 프로세스들이 shared library를 사용한다면, 프로세스들의 RSS값의 합은, shared library 메모리 값이 중복되어 더해지기에 프로세스들이 실제 할당 받은 물리 메모리 값 보다 커지게 된다. 

예를 들어, 같은 shared library를 사용하는 3개의 프로세스가 있다면, 각각의 프로세스의 RSS값은 모두 shared library의 메모리를 포함하게 되고, 3개 프로세스의 RSS값 합은 3개의 프로세스가 실제 할당 받은 물리 메모리 값보다 커지게 될 것이다. 리눅스에서는 현재 프로세스의 RSS값을 get_mm_rss라는 함수를 통해서 얻을 수 있다.
