커널 부팅 프로세스. 파트 1
================================================================================

부트로더에서 커널까지
--------------------------------------------------------------------------------
[[English]](linux-bootstrap-1.md#kernel-booting-process-part-1)

저의 이전 [블로그 포스트](https://0xax.github.io/categories/assembler/)를 혹시 보셨다면 아시겠지만, 저는 얼마동안 저(low)레벨 프로그래밍에 몰두했었습니다. 저는 당시 `x86_64` 리눅스를 위한 어셈블리 프로그래밍에 관한 포스트를 썼었고, 그와 함께 리눅스 커널의 소스코드를 살펴 보기 시작했습니다.

저는 프로그램이 어떻게 컴퓨터에서 움직이는지, 메모리는 어떻게 할당되는지, 어떻게 커널이 프로세스와 메모리를 관리하는지, 네트워크 스택은 어떻게 움직이는지와 같은 저(low)레벨의 것들을 알아가는데 매우 큰 흥미를 가지고 있습니다. 그래서 저는 **x86_64** 아키텍쳐의 리눅스 커널을 위한 또 하나의 포스트 시리즈를 작성하기로 마음먹었습니다.

하지만 주의 하실점은 저는 프로페셔널 커널 해커가 아니라는 점과 업무상 커널을 위한 코드를 작성하고 있지 않다는 점입니다. 이것은 그냥 취미 수준의 포스트입니다. 저는 개인적으로 저(low)레벨에 관심이 있고, 어떻게 움직이는지를 살펴보는것이 재미있습니다. 그래서 혹시 명확하지 않은 부분이나 질문/견해가 있으시면 저의 트위터로 [0xAX](https://twitter.com/0xAX) 연락 주시거나, [이메일](anotherworldofworld@gmail.com)로 연락을 하시거나 혹은 [이슈](https://github.com/0xAX/linux-insides/issues/new)를 작성해 주시면 감사하겠습니다.

모든 포스트들은 [깃헙 리포](https://github.com/0xAX/linux-insides)에서 확인하실수 있으며, 영어 문법등 어떠한 잘못된 부분을 발견하신다면 풀리퀘스트를 보내주시기를 바라겠습니다.

*주의 - 이것은 공식 문서가 아니고 배움과 지식을 공유하기 위한 포스트입니다.*

**전제지식**

* C코드의 이해
* 어셈블리 코드의 이해 (AT&T 문법)

자 그러면 지금부터 몇개의 포스트를 통해서 설명을 하도록 하겠습니다. 이것으로 간단한 소개는 마치기로하고 이제 본격적으로 리눅스 커널과 저(low)레벨을 살펴 보도록하죠.

저는 이 포스트를 `3.18` 리눅스 커널을 기준으로 작성을 시작하였기 때문에 일부 변경이 있을 수도 있습니다. 만약 어떠한 변경점이 있다면 그에 따라서 포스트도 업데이트 하도록 하겠습니다.

마법과 같은 전원 버튼, 그 다음에는 무슨일이 벌어질까?
--------------------------------------------------------------------------------

비록 이것이 리눅스 커널에 관한 포스트 시리즈 이지만, 우리는 커널 코드 부터 살펴보지는 않을 것입니다 - 적어도 지금 이 단락에서는요. 당신의 랩탑 혹은 데스크탑 컴퓨터의 전원을 누르면, 컴퓨터는 움직이기 시작합니다. 마더보드는 [전원공급장치](https://en.wikipedia.org/wiki/Power_supply)에 신호를 보내고, 신호를 받은 전원공급장치는 적절한 양의 전원을 컴퓨터에 공급하기 시작합니다. 일단 마더 보드가 [power good signal](https://en.wikipedia.org/wiki/Power_good_signal)을 받으면 그 다음으로 CPU를 가동합니다. CPU는 레지스터에 남아있는 모든 데이터를 리셋하고 미리 정해진 값으로 세팅합니다.

[80386](https://en.wikipedia.org/wiki/Intel_80386) CPU와 그 후의 CPU들은 다음과 같은 값으로 CPU 레지스터를 세팅합니다.

```
IP          0xfff0
CS selector 0xf000
CS base     0xffff0000
```
프로세서는 먼저 [real mode](https://en.wikipedia.org/wiki/Real_mode)로 가동을 시작합니다. 먼저 리얼 모드의 [메모리 세그멘테이션](https://en.wikipedia.org/wiki/Memory_segmentation)을 이해하도록 해 보죠. 리얼 모드는 [8086](https://en.wikipedia.org/wiki/Intel_8086)부터 현대의 인텔 64비트 CPU까지, 모든 x86 호환 프로세서들이 지원합니다. `8086` 프로세서는 20 비트의 어드레스 버스를 가지고 있는데, 이것은 `0-0xFFFFF` 혹은 `1 megabyte`의 주소 공간을 가지고 있음을 뜻합니다. 하지만 최대값 `2^16 - 1` 혹은 `0xffff` (64 kilobytes)의 `16-bit`의 레지스터들만 가지고 있습니다.

[Memory segmentation](https://en.wikipedia.org/wiki/Memory_segmentation)은 모든 주소 공간을 사용하기 위한 방법입니다. 모든 메모리는 미리 정해진 작은 사이즈인 `65536` bytes (64 KB)로 나뉘어지는데 16-bit 레지스터로는 `64 KB` 이상의 메모리를 지정 할수 없기 때문에, 대안이 필요했습니다.

주소는 베이스 주소를 가지는 세그먼트 셀렉터와 베이스 주소로 부터의 오프셋, 두 부분으로 구성됩니다. 리얼모드에서는 세그먼트 셀렉터의 베이스 주소는 `Segment Selector * 16` 이며, 따라서 메모리의 물리 주소를 얻기 위해서는 세그멘트 셀렉터에 `16`을 곱한후 오프셋을 더하면 됩니다.

```
PhysicalAddress = Segment Selector * 16 + Offset
```

예를 들어, 만약 `CS:IP`가 `0x2000:0x0010`인 경우, 해당하는 물리주소는 다음과 같습니다:

```python
>>> hex((0x2000 << 4) + 0x0010)
'0x20010'
```

하지만 가장 큰 값의 세그먼트 셀럭터와 오프셋인 `0xffff:0xffff`의 경우, 물리주소는 다음과 같습니다

```python
>>> hex((0xffff << 4) + 0xffff)
'0x10ffef'
```

1 MB 를 `65520` bytes 만큼 초과하는 값인데, 리얼 모드에서는 1 MB 까지 밖에 엑세스 할 수 없기 때문에 `0x10ffef` 는 `0x00ffef` 이 되고, [A20 line](https://en.wikipedia.org/wiki/A20_line)가 비활성화 됩니다.

이제 우리는 리얼 모드와 리얼 모드에서의 메모리 주소 지정에 대해서 알게 되었습니다. 그럼 다시 리셋 후의 레지스터 값에 대하여 알아보도록 합시다.

`CS` 레지스터는 보이는 세그멘트 셀럭터와 숨겨진 베이스 주소, 두개의 파트로 구성되어 있습니다. 일반적으로 베이스 주소가 세그멘트 셀렉터 값에 16을 곱하여 정해지는 반면에, CS 레지스터는 하드웨어가 리셋되는 동안에 세그먼트 셀렉터 `0xf000` 와 베이스 주소 `0xffff0000` 로 정해지고, 프로세서는 이 특별한 주소를 `CS`가 변경되기 전까지 사용합니다.

EIP 레지스터의 값을 베이스 주소에 더하는 것으로 시작 주소가 결정됩니다:

```python
>>> 0xffff0000 + 0xfff0
'0xfffffff0'
```

여기서 4GB 보다 16 bytes 적은 `0xfffffff0` 를 얻었고, 이 포인트는 [reset vector](https://en.wikipedia.org/wiki/Reset_vector) 라고 불립니다. 이곳은 리셋 후 CPU가 첫번때 명령을 찾는 메모리 공간이며, 보통 바이오스의 엔트리 포인트로 [jump](https://en.wikipedia.org/wiki/JMP_%28x86_instruction%29) (`jmp`) 하는 명령이 있습니다. 예를들어, [coreboot](https://www.coreboot.org/) 의 소스 코드 (`src/cpu/x86/16bit/reset16.inc`)를 보면:

```assembly
    .section ".reset", "ax", %progbits
    .code16
.globl	_start
_start:
    .byte  0xe9
    .int   _start16bit - ( . + 2 )
    ...
```

여기서 우리는 `0xe9` 즉, `jmp` 명령 [opcode](http://ref.x86asm.net/coder32.html#xE9) 와 목적지 주소 `_start16bit - ( . + 2)` 를 확인할 수 있습니다. 

또한 `reset` 섹션은 `16` 바이트 이고, 주소 `0xfffffff0` (`src/cpu/x86/16bit/reset16.ld`) 로 부터 컴파일 되는 것을 알 수 있습니다:

```
SECTIONS {
    /* Trigger an error if I have an unuseable start address */
    _bogus = ASSERT(_start16bit >= 0xffff0000, "_start16bit too low. Please report.");
    _ROMTOP = 0xfffffff0;
    . = _ROMTOP;
    .reset . : {
        *(.reset);
        . = 15;
        BYTE(0x00);
    }
}
```

이제 바이오스가 시작됩니다. 초기화와 하드웨어를 체크한 후, 바이오스는 부팅가능한 장치를 찾습니다. 바이오스가 부트를 시도하는 디바이스의 순서는 바이오스 설정에 저장 되어있습니다. 하드 디스크에서 부팅을 시도하는 경우, 바이오스는 부트 섹터를 찾습니다. [MBR partition layout](https://en.wikipedia.org/wiki/Master_boot_record)로 파티션된 하드 디스크의 경우, 각 섹터의 사이즈는 `512` bytes 인데, 부트 섹터는 첫번째 섹터의 처음 `446` bytes 에 저장 되어있습니다. 첫 번째 섹터의 마지막 두 바이트 `0x55` 와 `0xaa` 로 바이오스는 해당 장치가 부팅 가능한 장치임을 확인합니다.

예를 들어,

```assembly
;
; Note: this example is written in Intel Assembly syntax
;
[BITS 16]

boot:
    mov al, '!'
    mov ah, 0x0e
    mov bh, 0x00
    mov bl, 0x07

    int 0x10
    jmp $

times 510-($-$$) db 0

db 0x55
db 0xaa
```

빌드 후 다음과 같이 시작하면:

```
nasm -f bin boot.nasm && qemu-system-x86_64 boot
```
이것은 [QEMU](http://qemu.org) 를 통해 방금 만든 디스크 이미지를 `boot` 바이너리로 사용하게 합니다. 해당 바이너리는 부트 섹터로서의 조건을 만족하는 (오리진은 `0x7c00`로 설정 되었으며 적절한 시퀀스로 종료됨) 어셈블리 코드로 생성 되었기 때문에, QEMU는 이것을 디스크 이미지의 마스터 부터 레코드 (MBR) 로 간주 할 것입니다.

다음을 확인 할 수 있습니다:

![Simple bootloader which prints only `!`](http://oi60.tinypic.com/2qbwup0.jpg)

이 예제에서 우리는 이 코드가 `0x7c00`에서 시작하는 `16-bit` 리얼 모드로 실행되는 것을 알 수 있습니다. 이것이 실행되면 `!` 심볼을 출력하는 [0x10](http://www.ctyme.com/intr/rb-0106.htm) interrupt 를 호출 합니다. 그리고 나머지 `510` 를 제로로 채우며, 앞에서 말한 두개의 바이트, `0xaa` 와 `0x55` 로 끝납니다. 

`objdump` 유틸리티를 통하여 이것의 바이너리 덤프를 확인 할 수 있습니다:

```
nasm -f bin boot.nasm
objdump -D -b binary -mi386 -Maddr16,data16,intel boot
```

물론 진짜 부트 섹터는 0과 느낌표가 아니라, 부트 프로세스를 계속 하기 위한 코드와 파티션 테이블을 가지고 있습니다. 이 지점에서 바이오스는 부트로더에게 컨트롤을 넘깁니다.

**NOTE**: 앞에서 설명한 바와 같이, CPU는 리얼모드로 움직이며, 리얼 모드에서는 메모리의 물리주소를 다음과 같은 방법으로 계산합니다:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

또한 앞서 설명한 바와 같이, 우리는 오직 16 비트의 범용 레지스터를 가지고 있기 때문에, 16 비트 레지스터의 최대값은 `0xffff` 이고, 이를 통해 나타낼수 있는 최대의 값은 다음과 같습니다:

```python
>>> hex((0xffff * 16) + 0xffff)
'0x10ffef'
```

`0x10ffef` 는 `1MB + 64KB - 16b` 와 같습니다. [8086](https://en.wikipedia.org/wiki/Intel_8086) 프로세서 (리얼 모드를 지원하는 첫번째 프로세서)는 반면에 20 비트 어드레스 라인을 가지고 있기 때문에 실제로 사용 가능한 메모리는, `2^20 = 1048576`, 1MB 입니다.

일반적으로 리얼모드에서의 메모리 맵은 다음과 같습니다:

```
0x00000000 - 0x000003FF - Real Mode Interrupt Vector Table
0x00000400 - 0x000004FF - BIOS Data Area
0x00000500 - 0x00007BFF - Unused
0x00007C00 - 0x00007DFF - Our Bootloader
0x00007E00 - 0x0009FFFF - Unused
0x000A0000 - 0x000BFFFF - Video RAM (VRAM) Memory
0x000B0000 - 0x000B7777 - Monochrome Video Memory
0x000B8000 - 0x000BFFFF - Color Video Memory
0x000C0000 - 0x000C7FFF - Video ROM BIOS
0x000C8000 - 0x000EFFFF - BIOS Shadow Area
0x000F0000 - 0x000FFFFF - System BIOS
```

이 포스트의 첫부분에서 CPU에 의해서 실행되는 첫번째 명령은 `0xFFFFF` (1MB) 보다 훨씬 더 큰 `0xFFFFFFF0`에 위치한다고 하였는데, 그렇다면 CPU는 어떻게 리얼모드에서 이 위치를 나타낼까요? 이것의 답은 [coreboot](https://www.coreboot.org/Developer_Manual/Memory_map) 문서에 있습니다:

```
0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapped into address space
```

실행의 처음 부분인 바이오스는 RAM이 아니라 ROM에 있습니다.

부트로더
--------------------------------------------------------------------------------

리눅스를 부팅 할 수 있는 부트로더는 [GRUB 2](https://www.gnu.org/software/grub/) 와 [syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project) 등 몇개가 존재합니다. 리눅스 커널은 리눅스 서포트를 위한 조건을 나타내는 [Boot protocol](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt) 를 가지고 있는데, 여기에서는 GRUB 2 를 기준으로 설명합니다.

계속해서, 이제 `BIOS` 는 부트 디바이스를 선택하였고, [boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD) 에서 실행을 시작하는 부트 섹터 코드로 컨트롤을 넘깁니다. 이 코드는 사용 가능한 공간의 제약으로 매우 간단하며, GRUB 2의 코어 이미지로 점프하기 위한 포인터를 가지고 있습니다. 코어 이미지는 보통 첫번째 파티션 앞, 미사용 공간의 첫번째 섹터 바로 다음에 저장되어 있는 [diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD) 에서 시작됩니다. 위의 코드는 파일 시스템을 처리 하기 위한 GRUB 2의 커널과 드라이버인 나머지 코어 이미지를 메모리로 로드합니다. 로딩이 끝난후 [grub_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c) 함수를 실행합니다. 

`grub_main` 함수는 콘솔을 초기화하고, 모듈을 위한 베이스 주소를 얻으며, 루트 디바이스를 설정하고, grub 설정 파일을 로드/파스 하며, 모듈 로드 등을 합니다. `grub_main` 함수의 마지막에서는 grub의 노멀모드로 이동합니다. `grub_normal_execute` 함수 (`grub-core/normal/main.c` 소스코드 파일) 는 마지막 준비를 완료하며, 오퍼레이팅 시스템을 선택 할 수 있는 메뉴를 표시합니다. 메뉴에서 하나를 선택하면 `grub_menu_execute_entry` 함수가 실행되어, grub `boot` 커맨드를 실행하고 선택된 오퍼레이팅 시스템을 부팅합니다.

커널 셋업 코드에서 `0x01f1` 오프셋을 가지고 있는 커널 부트 프로토콜에서 확인 할 수 있는 것과 같이, 부트로더는 반드시 커널 셋업 헤더의 몇개의 필드를 읽거나 채워야 합니다. [linker script](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) 에서 이 오프셋의 값들을 확인 할 수 있습니다. 커널 해더 [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S) 는 다음과 같이 시작합니다:

```assembly
    .globl hdr
hdr:
    setup_sects: .byte 0
    root_flags:  .word ROOT_RDONLY
    syssize:     .long 0
    ram_size:    .word 0
    vid_mode:    .word SVGA_MODE
    root_dev:    .word 0
    boot_flag:   .word 0xAA55
```
부트로더는 반드시 리눅스 부트 프로토콜에서 `write` 타입으로 마크된 이것과 나머지 헤더를 채워야 하며, 예를 들어 [this example](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L354) 은 커맨드라인에서 받은 값과 부트중에 계산된 값이 있습니다. (우리는 여기에서 모든 필드에 대해서 확인 하지는 않겠지만, 커널이 어떻게 이것을 사용 하는지를 확인 할 때 살펴보겠습니다. 모든 필드에 대한 설명은 [boot protocol](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L156)에서 확인 할 수 있습니다.)

커널 부트 프로토콜에서 확인 할 수 있는 것과 같이 커널이 로딩된 후 메모리는 다음과 같이 맵 될것입니다:

```shell
         | Protected-mode kernel  |
100000   +------------------------+
         | I/O memory hole        |
0A0000   +------------------------+
         | Reserved for BIOS      | Leave as much as possible unused
         ~                        ~
         | Command line           | (Can also be below the X+10000 mark)
X+10000  +------------------------+
         | Stack/heap             | For use by the kernel real-mode code.
X+08000  +------------------------+
         | Kernel setup           | The kernel real-mode code.
         | Kernel boot sector     | The kernel legacy boot sector.
       X +------------------------+
         | Boot loader            | <- Boot sector entry point 0x7C00
001000   +------------------------+
         | Reserved for MBR/BIOS  |
000800   +------------------------+
         | Typically used by MBR  |
000600   +------------------------+
         | BIOS use only          |
000000   +------------------------+

```

즉, 부트로더가 컨트롤을 커널로 넘겼을때 시작되는 부분은:

```
X + sizeof(KernelBootSector) + 1
```

`X` 는 커널 부트 섹터가 로드 되는 부분입니다. 덤프를 살펴보니 저의 경우는, `X` 는 `0x10000` 입니다:

![kernel first address](http://oi57.tinypic.com/16bkco2.jpg)

부트로더가 리눅스 커널을 메모리에 로드하고 헤더 필드를 채우고 해당하는 메모리 주소로 이동을 하였습니다. 이제부터는 직적접으로 커널 셋업 코드를 살펴보겠습니다.

커널 셋업 스테이지의 시작
--------------------------------------------------------------------------------

드디어 커널을 살펴볼 차례인데, 좀 더 정확히 말하자면 아직은 커널이 실행되지 않았습니다. 먼저 커널 셋업 부분에서 반드시 디컴프레셔나 메모리 관리에 관한 부분 같은 설정이 이루어져야 합니다. 이 부분이 완료가 되면 실제 커널로 점프하게 됩니다. 셋업 부분의 실행은 [_start](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L292) 심볼의 [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S) 에서 시작됩니다.

이미 몇개의 인스트럭션이 있었기 때문에 처음에는 이상하게 보이지만, 예전에는 리눅스는 리눅스의 부트로더를 가지고 있었습니다. 하지만 지금은, 예를 들어 다음을 실행해보면,

```
qemu-system-x86_64 vmlinuz-3.18-generic
```

다음과 같은 결과를 확인 할 수 있습니다:


![Try vmlinuz in qemu](http://oi60.tinypic.com/r02xkz.jpg)

`header.S` 파일은 마법의 숫자 [MZ](https://en.wikipedia.org/wiki/DOS_MZ_executable) (위의 이미지대로) 로 시작하고, 에러 메시지가 표시된 후 [PE](https://en.wikipedia.org/wiki/Portable_Executable) 헤더:

```assembly
#ifdef CONFIG_EFI_STUB
# "MZ", MS-DOS header
.byte 0x4d
.byte 0x5a
#endif
...
...
...
pe_header:
    .ascii "PE"
    .word 0
```

이것은 [UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface) 서포트와 함께 오퍼레이팅 시스템을 로드 하기 위해서 필요합니다. 여기에 대해서 자세한 부분은 다음 챕터에서 확인 하겠습니다. 

커널 셋업의 엔트리 포인트는:

```assembly
// header.S line 292
.globl _start
_start:
```

부트로더 (grub2 같은) 는 이 포인트 (`MZ`에서 오프셋 `0x200`) 를 알고 있기 때문에 에러 메시지를 출력하는, `.bstext` 섹션에서 시작하는 `header.S`가 아니라 해당 포인트로 바로 점프를 합니다:

```
//
// arch/x86/boot/setup.ld
//
. = 0;                    // current position
.bstext : { *(.bstext) }  // put .bstext section to position 0
.bsdata : { *(.bsdata) }
```

커널의 셋업 엔트리 포인트는:

```assembly
    .globl _start
_start:
    .byte  0xeb
    .byte  start_of_setup-1f
1:
    //
    // rest of the header
    //
```

여기서 우리는 `start_of_setup-1f` 로 점프하는 `jmp` 인스트럭션의 명령코드 (`0xeb`) 를 확인 할 수 있습니다. 예를 들어 `Nf` 노테이션 `2f`는 로컬 레이블 `2:` 을 참조 하고, 이 경우는 점프 후에는 나머지 셋업 코드 [header](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L156) 가 있는 레이블 `1` 이 됩니다. 셋업 헤더 바로 다음에는 우리는 `start_of_setup` 레이블에서 시작하는 `.entrytext` 섹션을 확인 할 수 있습니다.

앞선 점프 인스트럭션을 제외 하면 이부분이 실제로 실행되는 첫번째 코드라고 할 수 있습니다. 첫번째 512 bytes 부분, 커널 리얼 모드 시작에서 `0x200` 오프셋만큼 떨어진 첫번째 `jmp` 인스트럭션으로 부트로더로부터 커널 셋업 파트가 컨트롤을 받은 후는 리눅스 커널 부트 프로토콜이나 grub2 소스 양쪽에서 확인 할 수 있습니다:

```C
segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;
```

저의 경우에는 커널이 피지컬 어드레스 `0x10000` 에 로드 되었습니다. 이것은 세그멘트 레지스터들이 커널 셋업 시작 후에 다음과 같은 값을 가지는 것을 뜻합니다:

```
gs = fs = es = ds = ss = 0x1000
cs = 0x1020
```

`start_of_setup`로 점프 한 후, 커널은 다음을 필요로 합니다:

* 모든 세그멘트 레지스터의 값이 같을 것
* 필요하면 올바르게 스택을 셋업
* [bss](https://en.wikipedia.org/wiki/.bss) 의 셋업
* C 코드 [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) 로 점프

임플리멘테이션을 살펴 보도록 하겠습니다.

세그멘트 레지스터 정렬
--------------------------------------------------------------------------------

먼저 커널은 `ds` 와 `es` 세그먼트가 같은 주소를 가르키게 합니다. 그 다음 `cld` 명령으로 디렉션 플래그를 클리어합니다:

```assembly
    movw    %ds, %ax
    movw    %ax, %es
    cld
```
앞서 제가 알려드린것과 같이 `grub2`는 기본적으로 커널 셋업 코드를 어드레스 `0x10000` 에, `cs` 를 `0x1020` 에 로드하는데 이는 실행이 파일 처음에서 시작 되는게 아니라 점프에서 시작하기 때문입니다:

```assembly
_start:
    .byte 0xeb
    .byte start_of_setup-1f
```

[4d 5a](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L46) 에서 `512` byte 오프셋입니다. 또한 `cs` 를 다른 세그멘트 레지스터와 같이 `0x1020` 에서 `0x1000` 로 정렬합니다. 그 다음 스택을 셋업합니다:

```assembly
    pushw   %ds
    pushw   $6f
    lretw
```
[6](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L602) 라벨과 `lretw` 인스트럭션 실행 후 `ds` 값을 스택에 푸쉬합니다. `lretw` 익스트럭션이 실행 될 때, `6` 레이블의 주소를 [instruction pointer](https://en.wikipedia.org/wiki/Program_counter) 레지스터에 로드하고 `cs` 를 `ds` 의 값으로 로드 합니다. 그 이후 `ds` 와 `cs` 는 같은 값을 가지게 됩니다.

스택 셋업
--------------------------------------------------------------------------------

거의 대부분의 셋업 코드는 리얼모드에서 C 언어 환경을 준비 하기 위함입니다. 다음 [step](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L575) 은 `ss` 레지스터의 값을 체크하고 만약 `ss` 의 값이 올바르지 않다면 올바른 스택을 셋업합니다:

```assembly
    movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f
```

여기에는 3가지의 시나리오가 있습니다:

* `ss` 가 올바른 값 `0x1000` 을 가지고 있다 (`cs` 를 포함하여 다른 모든 레지스터가 그런것과 같이)
* `ss` 의 값이 올바르지 않고 `CAN_USE_HEAP` 플레그가 설정되었다 (아래 참조)
* `ss` 의 값이 올바르지 않고 `CAN_USE_HEAP` 플레그가 설정되지 않았다 (아래 참조)

모든 3가지의 시나리오를 차례로 살펴 보겠습니다:

* `ss` 가 올바른 메모리 (`0x1000`) 를 가지고 있다. 이 경우 레이블 [2](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L589) 로 갑니다:

```assembly
2:  andw    $~3, %dx
    jnz     3f
    movw    $0xfffc, %dx
3:  movw    %ax, %ss
    movzwl  %dx, %esp
    sti
```

여기서 부트로더로 부터 받은 `sp` 의 값을 가지고 있는 `dx` 의 정렬을 `4` bytes 로 설정하고 제로인지 체크를 합니다. 만약 제로라면, `dx` 를 `0xfffc` (64KB 세그먼트에서 뒷부분 4-byte 정렬된 주소) 로 설정하고, 만약 제로가 아니면 부트로더로 부터 받은 `sp` 값 (이 경우 0xf7f4) 을 계속 사용합니다. 그 다음, `ax` 의 값을 `ss` 에 설정하는데, 이 것은 `ss` 의 값이 `0x1000` 임을 뜻합니다. 이제 올바른 스택을 가지게 되었습니다:

![stack](http://oi58.tinypic.com/16iwcis.jpg)

* 두번째 시나리오 (`ss` != `ds`) 에서는, 먼저 [_end](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) (셋업 코드의 마지막 주소) 의 값을 `dx` 에 넣고, `testb` 인스트럭션으로 힙을 사용할 수 있는지 `loadflags` 헤더 필드를 확인합니다. [loadflags](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L320) 는 비트마스크 헤더로 다음과 같이 정의 됩니다:

```C
#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)
```

그리고 부트 프로토콜을 보면:

```
Field name: loadflags

  This field is a bitmask.

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.
```

만약 `CAN_USE_HEAP` 비트가 설정된 상태면, `heap_end_ptr` 를 `dx` (`_end` 를 가르키는) 에 넣고 `STACK_SIZE` (최대 스택 사이즈 `1024` bytes) 를 더합니다. 이후 만약 `dx` 에 자리 올림이 없으면 (올림은 발생하지 않을겁니다, `dx = _end + 1024`), 레이블 `2` (앞의 상황과 같이) 로 점프하고 올바른 스택을 만듭니다.

![stack](http://oi62.tinypic.com/dr7b5w.jpg)

* `CAN_USE_HEAP` 가 설정되지 않은 상태의 경우, `_end` 부터 `_end + STACK_SIZE` 까지를 최소한의 스택으로 사용합니다:

![minimal stack](http://oi60.tinypic.com/28w051y.jpg)

BSS 셋업
--------------------------------------------------------------------------------

메인 C 코드로 점프 하기 전, 마지막 2개의 스텝은 [BSS](https://en.wikipedia.org/wiki/.bss) 영역의 셋업과 "매직" 시그니쳐의 체크 입니다. 먼저 시그니쳐를 체크합니다:

```assembly
    cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad
```
여기서는 단순히 [setup_sig](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) 와 매직넘버`0x5a5aaa55` 를 비교합니다. 만약 둘이 같지 않으면, fatal error 가 리포트 됩니다.

만약 매직 넘버와 매치하면, 세그멘트 레지스터와 스택의 셋업이 잘 끝났다는 것이고, 남은 스탭은 C 코드로 점프 하기 전에 BSS 섹션을 셋업하는 것입니다.

BSS 섹션은 정적으로 할당되는 비초기화 데이터를 저장하는데 사용됩니다. 리눅스는 다음의 코드로 조심스럽게 이 영역의 메모리가 전부 제로로 시작하는지를 확인합니다:

```assembly
    movw    $__bss_start, %di
    movw    $_end+3, %cx
    xorl    %eax, %eax
    subw    %di, %cx
    shrw    $2, %cx
    rep; stosl
```

먼저 [__bss_start](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) 주소가 `di` 로 이동됩니다. 다음으로 `_end + 3` 주소 (+3 - 4 bytes 로 정렬) 는 `cx` 로 이동됩니다. `eax` 레지스터는 클리어 되고 (`xor` 인스트럭션을 이용해서), 그리고 bss 섹션 사이즈 (`cx`-`di`) 를 계산하여 `cx` 에 넣습니다. 그런 다음, `cx` 는 4로 나뉘어지고(워드 사이즈), `stosl` 인스트럭션이 반복적으로 사용되어, 4씩 증가하는 `di` 에 의해 `cx` 가 제로에 도달 할 때 까지 `eax` 의 값 (제로) 을 해당 주소에 저장합니다. 이 코드로 `__bss_start` 에서 `_end` 까지 제로로 채워집니다:

![bss](http://oi59.tinypic.com/29m2eyr.jpg)

메인으로 점프
--------------------------------------------------------------------------------

여기까지 스택과 BSS가 마무리되고 `main()` C 함수로 점프할 수 있습니다:

```assembly
    calll main
```

`main()` 함수는 여기 [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) 에서 확인 할 수 있습니다. 그리고 다음 파트에서는 이 코드에 대해서 살펴보겠습니다.

마무리
--------------------------------------------------------------------------------

여기까지가 리눅스 인사이드의 첫번째 파트입니다. 질문이나 제안이 있으신분은 제 트위터 [0xAX](https://twitter.com/0xAX)로 연락을 주시거나, [email](anotherworldofworld@gmail.com), 혹은 [issue](https://github.com/0xAX/linux-internals/issues/new)를 등록해 주세요. 다음파트에서는 첫번째 커널 셋업에서 실행되는 C 코드와 `memset`, `memcpy`, `earlyprintk` 와 같은 메모리 루틴, 초기의 콘솔 임플리멘테이션과 초기화 등을 살펴 보겠습니다. 

**Please note that English is not my first language and I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

  * [Intel 80386 programmer's reference manual 1986](http://css.csail.mit.edu/6.858/2014/readings/i386.pdf)
  * [Minimal Boot Loader for Intel® Architecture](https://www.cs.cmu.edu/~410/doc/minimal_boot.pdf)
  * [8086](https://en.wikipedia.org/wiki/Intel_8086)
  * [80386](https://en.wikipedia.org/wiki/Intel_80386)
  * [Reset vector](https://en.wikipedia.org/wiki/Reset_vector)
  * [Real mode](https://en.wikipedia.org/wiki/Real_mode)
  * [Linux kernel boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)
  * [coreboot developer manual](https://www.coreboot.org/Developer_Manual)
  * [Ralf Brown's Interrupt List](http://www.ctyme.com/intr/int.htm)
  * [Power supply](https://en.wikipedia.org/wiki/Power_supply)
  * [Power good signal](https://en.wikipedia.org/wiki/Power_good_signal)


--------------------------------------------------------------------------------
[[한국어]](linux-bootstrap-1.md#커널-부팅-프로세스-파트-1)

Kernel booting process. Part 1.
================================================================================

From the bootloader to the kernel
--------------------------------------------------------------------------------

If you've read my previous [blog posts](https://0xax.github.io/categories/assembler/), you might have noticed that I  have been involved with low-level programming for some time. I have written some posts about assembly programming for `x86_64` Linux and, at the same time, I have also started to dive into the Linux kernel source code.

I have a great interest in understanding how low-level things work, how programs run on my computer, how they are located in memory, how the kernel manages processes and memory, how the network stack works at a low level, and many many other things. So, I have decided to write yet another series of posts about the Linux kernel for the **x86_64** architecture.

Note that I'm not a professional kernel hacker and I don't write code for the kernel at work. It's just a hobby. I just like low-level stuff, and it is interesting for me to see how these things work. So if you notice anything confusing, or if you have any questions/remarks, ping me on Twitter [0xAX](https://twitter.com/0xAX), drop me an [email](anotherworldofworld@gmail.com) or just create an [issue](https://github.com/0xAX/linux-insides/issues/new). I appreciate it.

All posts will also be accessible at [github repo](https://github.com/0xAX/linux-insides) and, if you find something wrong with my English or the post content, feel free to send a pull request.

*Note that this isn't official documentation, just learning and sharing knowledge.*

**Required knowledge**

* Understanding C code
* Understanding assembly code (AT&T syntax)

Anyway, if you are just starting to learn such tools, I will try to explain some parts during this and the following posts. Alright, this is the end of the simple introduction, and now we can start to dive into the Linux kernel and low-level stuff.

I've started writing this book at the time of the `3.18` Linux kernel, and many things might have changed since that time. If there are changes, I will update the posts accordingly.

The Magical Power Button, What happens next?
--------------------------------------------------------------------------------

Although this is a series of posts about the Linux kernel, we will not be starting directly from the kernel code - at least not, in this paragraph. As soon as you press the magical power button on your laptop or desktop computer, it starts working. The motherboard sends a signal to the [power supply](https://en.wikipedia.org/wiki/Power_supply) device. After receiving the signal, the power supply provides the proper amount of electricity to the computer. Once the motherboard receives the [power good signal](https://en.wikipedia.org/wiki/Power_good_signal), it tries to start the CPU. The CPU resets all leftover data in its registers and sets up predefined values for each of them.

The [80386](https://en.wikipedia.org/wiki/Intel_80386) CPU and later CPUs define the following predefined data in CPU registers after the computer resets:

```
IP          0xfff0
CS selector 0xf000
CS base     0xffff0000
```

The processor starts working in [real mode](https://en.wikipedia.org/wiki/Real_mode). Let's back up a little and try to understand [memory segmentation](https://en.wikipedia.org/wiki/Memory_segmentation) in this mode. Real mode is supported on all x86-compatible processors, from the [8086](https://en.wikipedia.org/wiki/Intel_8086) CPU all the way to the modern Intel 64-bit CPUs. The `8086` processor has a 20-bit address bus, which means that it could work with a `0-0xFFFFF` or `1 megabyte` address space. But it only has `16-bit` registers, which have a maximum address of `2^16 - 1` or `0xffff` (64 kilobytes).

[Memory segmentation](https://en.wikipedia.org/wiki/Memory_segmentation) is used to make use of all the address space available. All memory is divided into small, fixed-size segments of `65536` bytes (64 KB). Since we cannot address memory above `64 KB` with 16-bit registers, an alternate method was devised.

An address consists of two parts: a segment selector, which has a base address, and an offset from this base address. In real mode, the associated base address of a segment selector is `Segment Selector * 16`. Thus, to get a physical address in memory, we need to multiply the segment selector part by `16` and add the offset to it:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

For example, if `CS:IP` is `0x2000:0x0010`, then the corresponding physical address will be:

```python
>>> hex((0x2000 << 4) + 0x0010)
'0x20010'
```

But, if we take the largest segment selector and offset, `0xffff:0xffff`, then the resulting address will be:

```python
>>> hex((0xffff << 4) + 0xffff)
'0x10ffef'
```

which is `65520` bytes past the first megabyte. Since only one megabyte is accessible in real mode, `0x10ffef` becomes `0x00ffef` with the [A20 line](https://en.wikipedia.org/wiki/A20_line) disabled.

Ok, now we know a little bit about real mode and memory addressing in this mode. Let's get back to discussing register values after reset.

The `CS` register consists of two parts: the visible segment selector, and the hidden base address. While the base address is normally formed by multiplying the segment selector value by 16, during a hardware reset the segment selector in the CS register is loaded with `0xf000` and the base address is loaded with `0xffff0000`; the processor uses this special base address until `CS` is changed.

The starting address is formed by adding the base address to the value in the EIP register:

```python
>>> 0xffff0000 + 0xfff0
'0xfffffff0'
```

We get `0xfffffff0`, which is 16 bytes below 4GB. This point is called the [reset vector](https://en.wikipedia.org/wiki/Reset_vector). This is the memory location at which the CPU expects to find the first instruction to execute after reset. It contains a [jump](https://en.wikipedia.org/wiki/JMP_%28x86_instruction%29) (`jmp`) instruction that usually points to the BIOS entry point. For example, if we look in the [coreboot](https://www.coreboot.org/) source code (`src/cpu/x86/16bit/reset16.inc`), we will see:

```assembly
    .section ".reset", "ax", %progbits
    .code16
.globl	_start
_start:
    .byte  0xe9
    .int   _start16bit - ( . + 2 )
    ...
```

Here we can see the `jmp` instruction [opcode](http://ref.x86asm.net/coder32.html#xE9), which is `0xe9`, and its destination address at `_start16bit - ( . + 2)`.

We can also see that the `reset` section is `16` bytes and is compiled to start from the address `0xfffffff0` (`src/cpu/x86/16bit/reset16.ld`):

```
SECTIONS {
    /* Trigger an error if I have an unuseable start address */
    _bogus = ASSERT(_start16bit >= 0xffff0000, "_start16bit too low. Please report.");
    _ROMTOP = 0xfffffff0;
    . = _ROMTOP;
    .reset . : {
        *(.reset);
        . = 15;
        BYTE(0x00);
    }
}
```

Now the BIOS starts; after initializing and checking the hardware, the BIOS needs to find a bootable device. A boot order is stored in the BIOS configuration, controlling which devices the BIOS attempts to boot from. When attempting to boot from a hard drive, the BIOS tries to find a boot sector. On hard drives partitioned with an [MBR partition layout](https://en.wikipedia.org/wiki/Master_boot_record), the boot sector is stored in the first `446` bytes of the first sector, where each sector is `512` bytes. The final two bytes of the first sector are `0x55` and `0xaa`, which designates to the BIOS that this device is bootable.

For example:

```assembly
;
; Note: this example is written in Intel Assembly syntax
;
[BITS 16]

boot:
    mov al, '!'
    mov ah, 0x0e
    mov bh, 0x00
    mov bl, 0x07

    int 0x10
    jmp $

times 510-($-$$) db 0

db 0x55
db 0xaa
```

Build and run this with:

```
nasm -f bin boot.nasm && qemu-system-x86_64 boot
```

This will instruct [QEMU](http://qemu.org) to use the `boot` binary that we just built as a disk image. Since the binary generated by the assembly code above fulfills the requirements of the boot sector (the origin is set to `0x7c00` and we end it with the magic sequence), QEMU will treat the binary as the master boot record (MBR) of a disk image.

You will see:

![Simple bootloader which prints only `!`](http://oi60.tinypic.com/2qbwup0.jpg)

In this example, we can see that the code will be executed in `16-bit` real mode and will start at `0x7c00` in memory. After starting, it calls the [0x10](http://www.ctyme.com/intr/rb-0106.htm) interrupt, which just prints the `!` symbol; it fills the remaining `510` bytes with zeros and finishes with the two magic bytes `0xaa` and `0x55`.

You can see a binary dump of this using the `objdump` utility:

```
nasm -f bin boot.nasm
objdump -D -b binary -mi386 -Maddr16,data16,intel boot
```

A real-world boot sector has code for continuing the boot process and a partition table instead of a bunch of 0's and an exclamation mark :) From this point onwards, the BIOS hands over control to the bootloader.

**NOTE**: As explained above, the CPU is in real mode; in real mode, calculating the physical address in memory is done as follows:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

just as explained above. We have only 16-bit general purpose registers; the maximum value of a 16-bit register is `0xffff`, so if we take the largest values, the result will be:

```python
>>> hex((0xffff * 16) + 0xffff)
'0x10ffef'
```

where `0x10ffef` is equal to `1MB + 64KB - 16b`. An [8086](https://en.wikipedia.org/wiki/Intel_8086) processor (which was the first processor with real mode), in contrast, has a 20-bit address line. Since `2^20 = 1048576` is 1MB, this means that the actual available memory is 1MB.

In general, real mode's memory map is as follows:

```
0x00000000 - 0x000003FF - Real Mode Interrupt Vector Table
0x00000400 - 0x000004FF - BIOS Data Area
0x00000500 - 0x00007BFF - Unused
0x00007C00 - 0x00007DFF - Our Bootloader
0x00007E00 - 0x0009FFFF - Unused
0x000A0000 - 0x000BFFFF - Video RAM (VRAM) Memory
0x000B0000 - 0x000B7777 - Monochrome Video Memory
0x000B8000 - 0x000BFFFF - Color Video Memory
0x000C0000 - 0x000C7FFF - Video ROM BIOS
0x000C8000 - 0x000EFFFF - BIOS Shadow Area
0x000F0000 - 0x000FFFFF - System BIOS
```

In the beginning of this post, I wrote that the first instruction executed by the CPU is located at address `0xFFFFFFF0`, which is much larger than `0xFFFFF` (1MB). How can the CPU access this address in real mode? The answer is in the [coreboot](https://www.coreboot.org/Developer_Manual/Memory_map) documentation:

```
0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapped into address space
```

At the start of execution, the BIOS is not in RAM, but in ROM.

Bootloader
--------------------------------------------------------------------------------

There are a number of bootloaders that can boot Linux, such as [GRUB 2](https://www.gnu.org/software/grub/) and [syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project). The Linux kernel has a [Boot protocol](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt) which specifies the requirements for a bootloader to implement Linux support. This example will describe GRUB 2.

Continuing from before, now that the `BIOS` has chosen a boot device and transferred control to the boot sector code, execution starts from [boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD). This code is very simple, due to the limited amount of space available, and contains a pointer which is used to jump to the location of GRUB 2's core image. The core image begins with [diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD), which is usually stored immediately after the first sector in the unused space before the first partition. The above code loads the rest of the core image, which contains GRUB 2's kernel and drivers for handling filesystems, into memory. After loading the rest of the core image, it executes the [grub_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c) function.

The `grub_main` function initializes the console, gets the base address for modules, sets the root device, loads/parses the grub configuration file, loads modules, etc. At the end of execution, the `grub_main` function moves grub to normal mode. The `grub_normal_execute` function (from the `grub-core/normal/main.c` source code file) completes the final preparations and shows a menu to select an operating system. When we select one of the grub menu entries, the `grub_menu_execute_entry` function runs, executing the grub `boot` command and booting the selected operating system.

As we can read in the kernel boot protocol, the bootloader must read and fill some fields of the kernel setup header, which starts at the `0x01f1` offset from the kernel setup code. You may look at the boot [linker script](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) to confirm the value of this offset. The kernel header [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S) starts from:

```assembly
    .globl hdr
hdr:
    setup_sects: .byte 0
    root_flags:  .word ROOT_RDONLY
    syssize:     .long 0
    ram_size:    .word 0
    vid_mode:    .word SVGA_MODE
    root_dev:    .word 0
    boot_flag:   .word 0xAA55
```

The bootloader must fill this and the rest of the headers (which are only marked as being type `write` in the Linux boot protocol, such as in [this example](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L354)) with values which it has either received from the command line or calculated during boot. (We will not go over full descriptions and explanations for all fields of the kernel setup header now, but we shall do so when we discuss how the kernel uses them; you can find a description of all fields in the [boot protocol](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L156).)

As we can see in the kernel boot protocol, the memory will be mapped as follows after loading the kernel:

```shell
         | Protected-mode kernel  |
100000   +------------------------+
         | I/O memory hole        |
0A0000   +------------------------+
         | Reserved for BIOS      | Leave as much as possible unused
         ~                        ~
         | Command line           | (Can also be below the X+10000 mark)
X+10000  +------------------------+
         | Stack/heap             | For use by the kernel real-mode code.
X+08000  +------------------------+
         | Kernel setup           | The kernel real-mode code.
         | Kernel boot sector     | The kernel legacy boot sector.
       X +------------------------+
         | Boot loader            | <- Boot sector entry point 0x7C00
001000   +------------------------+
         | Reserved for MBR/BIOS  |
000800   +------------------------+
         | Typically used by MBR  |
000600   +------------------------+
         | BIOS use only          |
000000   +------------------------+

```

So, when the bootloader transfers control to the kernel, it starts at:

```
X + sizeof(KernelBootSector) + 1
```

where `X` is the address of the kernel boot sector being loaded. In my case, `X` is `0x10000`, as we can see in a memory dump:

![kernel first address](http://oi57.tinypic.com/16bkco2.jpg)

The bootloader has now loaded the Linux kernel into memory, filled the header fields, and then jumped to the corresponding memory address. We can now move directly to the kernel setup code.

The Beginning of the Kernel Setup Stage
--------------------------------------------------------------------------------

Finally, we are in the kernel! Technically, the kernel hasn't run yet; first, the kernel setup part must configure stuff such as the decompressor and some memory management related things, to name a few. After all these things are done, the kernel setup part will decompress the actual kernel and jump to it. Execution of the setup part starts from [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S) at the [_start](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L292) symbol.

It may looks a little bit strange at first sight, as there are several instructions before it. A long time ago, the Linux kernel used to have its own bootloader. Now, however, if you run, for example,

```
qemu-system-x86_64 vmlinuz-3.18-generic
```

then you will see:

![Try vmlinuz in qemu](http://oi60.tinypic.com/r02xkz.jpg)

Actually, the file `header.S` starts with the magic number [MZ](https://en.wikipedia.org/wiki/DOS_MZ_executable) (see image above), the error message that displays and, following that, the [PE](https://en.wikipedia.org/wiki/Portable_Executable) header:

```assembly
#ifdef CONFIG_EFI_STUB
# "MZ", MS-DOS header
.byte 0x4d
.byte 0x5a
#endif
...
...
...
pe_header:
    .ascii "PE"
    .word 0
```

It needs this to load an operating system with [UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface) support. We won't be looking into its inner workings right now and will cover it in upcoming chapters.

The actual kernel setup entry point is:

```assembly
// header.S line 292
.globl _start
_start:
```

The bootloader (grub2 and others) knows about this point (at an offset of `0x200` from `MZ`) and makes a jump directly to it, despite the fact that `header.S` starts from the `.bstext` section, which prints an error message:

```
//
// arch/x86/boot/setup.ld
//
. = 0;                    // current position
.bstext : { *(.bstext) }  // put .bstext section to position 0
.bsdata : { *(.bsdata) }
```

The kernel setup entry point is:

```assembly
    .globl _start
_start:
    .byte  0xeb
    .byte  start_of_setup-1f
1:
    //
    // rest of the header
    //
```

Here we can see a `jmp` instruction opcode (`0xeb`) that jumps to the `start_of_setup-1f` point. In `Nf` notation, `2f`, for example, refers to the local label `2:`; in our case, it is the label `1` that is present right after the jump, and it contains the rest of the setup [header](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L156). Right after the setup header, we see the `.entrytext` section, which starts at the `start_of_setup` label.

This is the first code that actually runs (aside from the previous jump instructions, of course). After the kernel setup part receives control from the bootloader, the first `jmp` instruction is located at the `0x200` offset from the start of the kernel real mode, i.e., after the first 512 bytes. This can be seen in both the Linux kernel boot protocol and the grub2 source code:

```C
segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;
```

In my case, the kernel is loaded at `0x10000` physical address. This means that segment registers will have the following values after kernel setup starts:

```
gs = fs = es = ds = ss = 0x1000
cs = 0x1020
```

After the jump to `start_of_setup`, the kernel needs to do the following:

* Make sure that all segment register values are equal
* Set up a correct stack, if needed
* Set up [bss](https://en.wikipedia.org/wiki/.bss)
* Jump to the C code in [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)

Let's look at the implementation.

Aligning the Segment Registers 
--------------------------------------------------------------------------------

First of all, the kernel ensures that the `ds` and `es` segment registers point to the same address. Next, it clears the direction flag using the `cld` instruction:

```assembly
    movw    %ds, %ax
    movw    %ax, %es
    cld
```

As I wrote earlier, `grub2` loads kernel setup code at address `0x10000` by default and `cs` at `0x1020` because execution doesn't start from the start of file, but from the jump here:

```assembly
_start:
    .byte 0xeb
    .byte start_of_setup-1f
```

which is at a `512` byte offset from [4d 5a](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L46). We also need to align `cs` from `0x1020` to `0x1000`, as well as all other segment registers. After that, we set up the stack:

```assembly
    pushw   %ds
    pushw   $6f
    lretw
```

which pushes the value of `ds` to the stack, followed by the address of the [6](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L602) label and executes the `lretw` instruction. When the `lretw` instruction is called, it loads the address of label `6` into the [instruction pointer](https://en.wikipedia.org/wiki/Program_counter) register and loads `cs` with the value of `ds`. Afterward, `ds` and `cs` will have the same values.

Stack Setup
--------------------------------------------------------------------------------

Almost all of the setup code is for preparing the C language environment in real mode. The next [step](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L575) is checking the `ss` register's value and setting up a correct stack if `ss` is wrong:

```assembly
    movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f
```

This can lead to 3 different scenarios:

* `ss` has a valid value `0x1000` (as do all the other segment registers beside `cs`)
* `ss` is invalid and the `CAN_USE_HEAP` flag is set     (see below)
* `ss` is invalid and the `CAN_USE_HEAP` flag is not set (see below)

Let's look at all three of these scenarios in turn:

* `ss` has a correct address (`0x1000`). In this case, we go to label [2](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L589):

```assembly
2:  andw    $~3, %dx
    jnz     3f
    movw    $0xfffc, %dx
3:  movw    %ax, %ss
    movzwl  %dx, %esp
    sti
```

Here we set the alignment of `dx` (which contains the value of `sp` as given by the bootloader) to `4` bytes and check if it is zero. If it is, we set `dx` to `0xfffc` (The last 4-byte aligned address in a 64KB segment). If it is not zero, we continue to use the value of `sp` given by the bootloader (0xf7f4 in my case). After this, we put the value of `ax` into `ss`, which means `ss` contains the value `0x1000`. We now have a correct stack:

![stack](http://oi58.tinypic.com/16iwcis.jpg)

* In the second scenario, (`ss` != `ds`). First, we put the value of [_end](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) (the address of the end of the setup code) into `dx` and check the `loadflags` header field using the `testb` instruction to see whether we can use the heap. [loadflags](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L320) is a bitmask header which is defined as:

```C
#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)
```

and, as we can read in the boot protocol:

```
Field name: loadflags

  This field is a bitmask.

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.
```

If the `CAN_USE_HEAP` bit is set, we put `heap_end_ptr` into `dx` (which points to `_end`) and add `STACK_SIZE` (the minimum stack size, `1024` bytes) to it. After this, if `dx` is not carried (it will not be carried, `dx = _end + 1024`), jump to label `2` (as in the previous case) and make a correct stack.

![stack](http://oi62.tinypic.com/dr7b5w.jpg)

* When `CAN_USE_HEAP` is not set, we just use a minimal stack from `_end` to `_end + STACK_SIZE`:

![minimal stack](http://oi60.tinypic.com/28w051y.jpg)

BSS Setup
--------------------------------------------------------------------------------

The last two steps that need to happen before we can jump to the main C code are setting up the [BSS](https://en.wikipedia.org/wiki/.bss) area and checking the "magic" signature. First, signature checking:

```assembly
    cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad
```

This simply compares the [setup_sig](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) with the magic number `0x5a5aaa55`. If they are not equal, a fatal error is reported.

If the magic number matches, knowing we have a set of correct segment registers and a stack, we only need to set up the BSS section before jumping into the C code.

The BSS section is used to store statically allocated, uninitialized data. Linux carefully ensures this area of memory is first zeroed using the following code:

```assembly
    movw    $__bss_start, %di
    movw    $_end+3, %cx
    xorl    %eax, %eax
    subw    %di, %cx
    shrw    $2, %cx
    rep; stosl
```

First, the [__bss_start](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) address is moved into `di`. Next, the `_end + 3` address (+3 - aligns to 4 bytes) is moved into `cx`. The `eax` register is cleared (using a `xor` instruction), and the bss section size (`cx`-`di`) is calculated and put into `cx`. Then, `cx` is divided by four (the size of a 'word'), and the `stosl` instruction is used repeatedly, storing the value of `eax` (zero) into the address pointed to by `di`, automatically increasing `di` by four, repeating until `cx` reaches zero). The net effect of this code is that zeros are written through all words in memory from `__bss_start` to `_end`:

![bss](http://oi59.tinypic.com/29m2eyr.jpg)

Jump to main
--------------------------------------------------------------------------------

That's all - we have the stack and BSS, so we can jump to the `main()` C function:

```assembly
    calll main
```

The `main()` function is located in [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c). You can read about what this does in the next part.

Conclusion
--------------------------------------------------------------------------------

This is the end of the first part about Linux kernel insides. If you have questions or suggestions, ping me on Twitter [0xAX](https://twitter.com/0xAX), drop me an [email](anotherworldofworld@gmail.com), or just create an [issue](https://github.com/0xAX/linux-internals/issues/new). In the next part, we will see the first C code that executes in the Linux kernel setup, the implementation of memory routines such as `memset`, `memcpy`, `earlyprintk`, early console implementation and initialization, and much more.

**Please note that English is not my first language and I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

  * [Intel 80386 programmer's reference manual 1986](http://css.csail.mit.edu/6.858/2014/readings/i386.pdf)
  * [Minimal Boot Loader for Intel® Architecture](https://www.cs.cmu.edu/~410/doc/minimal_boot.pdf)
  * [8086](https://en.wikipedia.org/wiki/Intel_8086)
  * [80386](https://en.wikipedia.org/wiki/Intel_80386)
  * [Reset vector](https://en.wikipedia.org/wiki/Reset_vector)
  * [Real mode](https://en.wikipedia.org/wiki/Real_mode)
  * [Linux kernel boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)
  * [coreboot developer manual](https://www.coreboot.org/Developer_Manual)
  * [Ralf Brown's Interrupt List](http://www.ctyme.com/intr/int.htm)
  * [Power supply](https://en.wikipedia.org/wiki/Power_supply)
  * [Power good signal](https://en.wikipedia.org/wiki/Power_good_signal)
