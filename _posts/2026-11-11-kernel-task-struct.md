---
title: "Task Struct"
description: "리눅스 task_struct의 식별자, 상태, 프로세스 관계, 메모리 및 스케줄링 정보를 정리한다."
date: 2026-11-11 21:55:58 +0900
categories: [Operating System, Kernel]
tags: [linux, kernel]
permalink: /operating-system/kernel/2026/11/11/task-struct/
---

Linux kernel 내에서 프로세스는 task_struct라는 구조체로 표현된다. 해당 구조체에는 프로세스를 표현하는 데 필요한 모든 데이터와 다른 프로세스와의 관계를 유지하고, 계정을 관리하는 데 필요한 여러 가지 데이터가 포함되어 있다. 내부적으로 프로세스의 attribute들을 기록하고, struct에 기록된 내용을 통해 프로세스들을 관리한다. task_struct는 include/linux/sched.h에 위치한다. task_struct의 모든 멤버를 설명하기에는 내용이 너무 방대하기에 다 설명할 수는 없고, 역할 별로 일부분만 설명을 하겠다. linux 5.0.9 버전을 기준으로 작성되었다.

- **Task identification**

**pid_t pid:** Task의 고유한 process ID이며, 프로세스마다 부여하는 정수형 값이다. 계속해서 증가하는 값이며, 0에서 다시 시작하는 경우는 없다. PID는 프로세스의 우선순위를 조정하거나, 종료하는 등의 다양한 함수 호출에 사용된다. Task에는 process와 thread가 포함된다. POSIX에서의 pid와는 다르다.

pid_t tgid: 스레드 그룹 아이디를 표현하는 값이다. 해당 프로세스가 스레드 리더인 경우, tgid와 pid는 같고, child thread인 경우 tgid와 pid는 다르다.

char comm[TASK_COMM_LEN]: 프로세스의 이름을 저장한다. get_task_comm/set_task_comm으로 얻을 수 있다.

- **Task** **state**

volatile long state: Task의 상태를 나타내는 bit set이다. TASK_RUNNING(실행 중이거나 곧 실행되기 위해 실행 큐에 있는 프로세스), TASK_INTERRUPTIBLE(대기 중인 프로세스), TASK_UNINTERRUPTIBLE(특정 조건에 깨어나는 프로세스), TASK_STOPPED(중지된 프로세스) 등등의 상태가 존재한다.

unsigned int flags: 프로세스 종류와 프로세스 실행 상태를 저장하는 멤버이다. 이 멤버에 저장된 값으로 프로세스의 세부 실행 상태를 알 수 있다.

int exit_state: 프로세스의 종료 상태가 저장되는 멤버이다. EXIT_DEAD, EXIT_ZOMBIE, EXIT_TRACE중 하나 값을 저장한다.

int exit_code: 프로세스의 종료 코드를 저장하는 멤버이다.

- **Task relationship**

struct task_struct *real_parent: 프로세스를 생성한 부모 프로세서의 task descriptor 주소를 저장하는 멤버이다.

struct task_struct *parent: 부모 프로세스를 의미한다. real_ parent 는 해당 프로세스를 생성해준 프로세스를 의미한다. real_parent가 소멸된 경우 부모 프로세스를 다른 프로세스로 지정해야 하는데, 계층 구조에서 부모 프로세스가 없는 경우 init 프로세스가 부모 프로세스로 변경된다.

struct list_head children: 자식 프로세스를 생성하는 경우 children이란 연결 리스트에 자식 프로세스를 등록한다.

struct list_head sibling: 같은 부모 프로세스에서 생성된 형제 관계 프로세스들의 연결 리스트이다.

struct list_head tasks: 커널에서 구동 중인 모든 프로세스는 tasks 연결 리스트에 등록되어 있다. 프로세스는 처음 생성될 때 init_task 전역 변수 필드인 tasks 연결 리스트에 등록된다.

- **Task time information**

u64 utime: 유저 모드에서 프로세스를 실행한 시각을 담는 멤버이다.

u64 stime: 시스템 모드(커널 모드)에서 프로세스를 실행한 시각을 담는 멤버이다.

u64 acct_timexpd: last update로부터 stime과 utime 값을 합한 값을 갖는 멤버이다.

u64 start_time, u64 real_start_time: 두개 다 start time을 갖는 멤버이지만, time base가 다르다. start_time은 monotonic, real_start_time은 boot based이다.

- **File information**

struct fs_struct *fs: filesystem 정보를 갖는 멤버이다.

struct files_struct *files: Open된 file의 정보를 갖는 멤버이다.

void *journal_info: Journaling filesystem의 정보를 갖는 멤버이다.

- **Signal information**

int exit_signal: 프로세스가 종료하는 시점에 SIGCHLD 시그널 값을 갖는 멤버이다. 해당 값은 종료하는 프로세스의 부모 프로세스에 전달된다.

int pdeath_signal: Parent process가 죽을 때 오는 시그널이 담기는 멤버이다.

struct signal_struct *signal, struct sighand_struct *sighand, sigset_t blocked, sigset_t real_blocked: signal handler들에 관한 정보가 담긴 멤버들이다. blocked에는 프로세스가 받지 않도록 설정한 시그널이 들어가있다.

sigset_t saved_sigmask: 만약 set_restore_sigmask()가 사용된다면 해당 멤버를 사용하여 복구한다.

struct sigpending pending: 프로세스가 시그널을 받았지만, 아직 이를 처리하지 못했음을 나타낸다.

- **Memory information**

struct mm_struct *mm, *active_mm: 프로세스의 메모리 관리 정보를 갖는 멤버이다. 프로세스의 메모리 상에서 위치를 나타내는 메모리 맵 정보를 가지고 있다.

u64 acct_rss_mem1: 누적된 rss 사용량을 갖는 멤버이다.

u64 acct_vm_mem1: 누적된 virtual memory 사용량을 갖는 멤버이다.

nodemask_t mems_allowed: 메모리를 얻을 수 있는 메모리 노드를 나타내는 멤버이다.

unsigned long min_flt, maj_flt: MM fault와 swap에 관한 정보를 갖는 멤버이다.

- **Scheduling information**

struct sched_info sched_info: 프로세스의 스케줄링 정보를 저장하는 멤버이다.

const struct sched_class *sched_class: 스케줄링과 관련된 함수들의 정보를 갖는 멤버이다.

struct sched_entity se: 스케줄링 entity 값들을 갖는 멤버이다.

struct sched_rt_entity rt: 실시간 task 스케줄링 entity 값들을 갖는 멤버이다.

unsigned int policy: 스케줄링 정책에 대한 설정 값을 갖는 멤버이다. SCHED_OTHER, SCHER_FIFO, SCHER_RR과 같은 값을 가질 수 있다.

int prio, static_prio, normal_prio: 프로세스에 할당된 정적 우선순위, 동적 우선순위 값을 갖는 멤버들이다.

unsigned int rt_priority: 실시간 task 스케줄링을 위한 우선순위 설정 값을 갖는 멤버이다.

struct ftrace_ret_stack *ret_stack: 마지막 스케줄로부터의 time stamp를 갖는 멤버이다.

- **Various field**

unsigned int cpu: 현재 구동되고 있는 CPU를 나타내는 멤버이다.

void *stack: 프로세스의 커널 stack을 가리키는 포인터이다.

unsigned long nvcsw, nivcsw: Context switch의 count값을 갖는 멤버이다.

const struct cred __rcu *cred: 현재 task의 credential 정보를 가리킨다.

struct thread_info thread_info: 프로세스 실행 흐름을 관리하는 중요한 정보를 저장하는 멤버이다. 컨텍스트 정보는 thread_info 구조체의 preempt_count 멤버에서 확인 가능하다.
