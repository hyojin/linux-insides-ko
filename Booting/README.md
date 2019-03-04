# 커널 부트 프로세스
[[English]](README.md#kernel-boot-process)

이 챕터에서는 리눅스 커널 부트 프로세스에 대해서 설명합니다. 커널 로딩 프로세스의 모든
과정에 관한 설명을 아래쪽 포스트에서 확인할 수 있습니다.

* [부트로더에서 커널까지](linux-bootstrap-1.md) - 컴퓨터의 전원을 켜고 난 다음부터 커널의 첫 번째 명령 실행까지의 설명입니다.
* [커널 셋업 코드의 첫 스탭](linux-bootstrap-2.md) - 커널 셋업 코드의 첫 번째 스텝에 관한 설명입니다. 힙의 초기화와 EDD, IST 등 여러 파라미터들의 쿼리에 대해 확인할 수 있습니다.
* [비디오 모드 초기화와 보호 모드 전환](linux-bootstrap-3.md) - 커널 셋업 코드의 비디오 모드 초기화와 보호 모드로의 전환에 대한 설명입니다.
* [64비트 모드 전환](linux-bootstrap-4.md) - 64비트 모드로의 전환과 준비과정에 대한 설명입니다.
* [커널 디컴프레션](linux-bootstrap-5.md) - 커널 압축해제와 준비과정에 대한 설명입니다.

--------------------------------------------------------------------------------

[[한국어]](README.md#커널-부트-프로세스)

# Kernel Boot Process

This chapter describes the linux kernel boot process. Here you will see a series of posts which describes the full cycle of the kernel loading process:

* [From the bootloader to kernel](linux-bootstrap-1.md) - describes all stages from turning on the computer to running the first instruction of the kernel.
* [First steps in the kernel setup code](linux-bootstrap-2.md) - describes first steps in the kernel setup code. You will see heap initialization, query of different parameters like EDD, IST and etc...
* [Video mode initialization and transition to protected mode](linux-bootstrap-3.md) - describes video mode initialization in the kernel setup code and transition to protected mode.
* [Transition to 64-bit mode](linux-bootstrap-4.md) - describes preparation for transition into 64-bit mode and details of transition.
* [Kernel Decompression](linux-bootstrap-5.md) - describes preparation before kernel decompression and details of direct decompression.
* [Kernel random address randomization](linux-bootstrap-6.md) - describes randomization of the Linux kernel load address.

This chapter coincides with `Linux kernel v4.17`.
