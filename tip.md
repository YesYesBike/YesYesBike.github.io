% C, Perl, Linux
% yesyesbike
%
# "돈을 벌려면 먼저 돈이 있어야 한다." -미상

***

[홈으로](index.html)

[잡다](jobda.html)

<br>

인생에 도움이 될지도 모르는 가치있는 팁 모음.
물론 그대로 이행해서 일어나는 모든 일은 본인의 책임이니 알아서 적절히 판단할 것.

<br>

### 001
ftrace와 관련한 작은 팁 하나.

`set_ftrace_filter`에 `available_filter_functions`에 있는 함수 이름을
입력해 그 함수 호출을 추적할 수 있는 건 다들 알 것이다. 하지만 문자열 처리에 큰 비용이 들 수 있다.
이를 피하기 위한 방법이 있다. 그러려면 먼저 `available_filter_functions`
에서 추적하고자 하는 함수가 몇 번째 줄에 있는지 알아야 한다.

```
$ grep -n kernel_clone /sys/kernel/debug/tracing/available_filter_functions
1860:kernel_clone
```

이렇게 줄 번호를 알아냈다면 그 다음에는 `set_ftrace_filter`에 그 번호를 입력하면 된다.

```
$ echo 1860 > /sys/kernel/debug/tracing/set_ftrace_filter
```

이 둘을 하나의 스크립트로 합칠 수 있을 것이다.

```
#!/bin/sh

[ -z "$*" ] && echo "No Arg" && exit 1

set -e

avail="/sys/kernel/debug/tracing/available_filter_functions"

for arg in $@
do
    tmp=$(grep -wn $arg $avail | cut -d: -f1)
    [ -z "$tmp" ] && echo "No result for \"$arg\"" && exit 1
    list="$list\n$tmp"
done

echo $list > /sys/kernel/debug/tracing/set_ftrace_filter
```

더 읽어보기: [커널 ftrace 문서](https://www.kernel.org/doc/html/latest/trace/ftrace.html#the-file-system)에서 `set_ftrace_filter` 항목 참조

<br>

### 002
ARM 어셈블리 코드를 읽다가 새로운 사실을 알게 되었다.
그 전까지는 이런 식으로 코드를 짰다.

```
.macro addr4 addr
.4byte (LOAD_ADDRESS+(\addr-_start))
.endm

main:
    mov r7, #SYS_WRITE
    mov r0, #STDOUT
    ldr r1, TEXT_ADDR
    mov r2, #(TEXT_END-TEXT)
    swi #0

    mov r7, #SYS_EXIT
    mov r0, #0
    swi #0

TEXT_ADDR:
    addr4 TEXT
TEXT:
    .ascii "Hello\n"
TEXT_END:
```

ARM 어셈블리에서는 32비트 즉시값을 바로 레지스터에 대입할 수 없어서
간접적으로 주소를 지정한 다음 그 주소에서 값을 불러와야 한다.
물론 `ldr r1, =0x12345678` 이런 식으로 값의 주소를 신경쓰지 않고 리터럴 값을 불러올 수 있다.
하지만 이 방법으로는 값의 주소(`TEXT_ADDR`의 위치)를 직접 지정할 수 없다.
그래서 매번 주소를 할당해(`TEXT_ADDR`) 거기에 주소(`TEXT`의 위치)를 저장했다.

그러다가 큰 깨달음을 얻게 되었다.
프로그램 카운터(pc)를 이용하면 되는 것이었다.
그러면 다음처럼 다시 작성할 수 있다.

```
main:
    mov r7, #SYS_WRITE
    mov r0, #STDOUT
    add r1, pc, #(4*4)
    mov r2, #(TEXT_END-TEXT)
    swi #0

    mov r7, #SYS_EXIT
    mov r0, #0
    swi #0

TEXT:
    .ascii "Hello\n"
TEXT_END:
```

이러면 메모리 공간도 아끼고 실행 속도도 절약할 수 있다.
하지만 이러면 명령어 주소를 일일이 계산해야 하는 문제가 있다.
이를 해결하려면 r1 레지스터를 대입하는 줄을 다음과 같이 고치면 된다.

```
add r1, pc, #(TEXT-(.+8))
```

한눈에 알아보기 힘들지만 차근차근 이해해보자.
주소는 임의로 덧붙인 것이다.

```
	50  add r1, pc, #(TEXT-(.+8))
    54  mov r2, #(TEXT_END-TEXT)
    58  swi #0
    5c  mov r7, #SYS_EXIT
    60  mov r0, #0
    64  swi #0
TEXT:
    68  .ascii "Hello\n"
```

일단 .은 현재 주소(0x50)를 나타낸다.
그리고 pc는 실행되는 명령어 주소에서 8을 더한 값을 가진다(0x58).
TEXT 라벨은 0x68에 있다. 이제 차근차근 계산해보자.

```
0x58 + (0x68 - (0x50 + 8))
0x58 + (0x68 - 0x58)
0x58 + 0x10
0x68
```

오프셋 자체는 컴파일 상수이기 때문에 두 번째 예시와 성능 차이는 없다.
다만 직관적이지 않아서 실수하기 쉽다. 매크로로 지정하면 이를 예방할 수 있다.

```
    .macro load_addr reg, addr
    add \reg, pc, #(\addr-(.+8))
    .endm
```

이를 적용한 전체 코드이다.

```
main:
    mov r7, #SYS_WRITE
    mov r0, #STDOUT
	load_addr r1, TEXT
    mov r2, #(TEXT_END-TEXT)
    swi #0

    mov r7, #SYS_EXIT
    mov r0, #0
    swi #0

TEXT:
    .ascii "jk rolling stones\n"
TEXT_END:
```

더 읽어보기: [왜 pc에 8을 더하는가?(Stackoverflow)](https://stackoverflow.com/a/24116090)
