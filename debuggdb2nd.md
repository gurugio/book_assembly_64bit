#분량조절실패로 쓰는 gdb로 어셈블리 코드 디버깅하기 2부

3가지 의문이 있었습니다.

1. 어셈블리 프로그래밍은 mov rax, 1로 구현했는데 왜 gdb에서는 mov  eax, 1이 되서 32비트 레지스터에 1을 저장하고있을까?

2. fd 값이 0이면 stdin을 의미해서 메세지가 출력이 안되야될것 같은데 왜 출력이 될까?

3. 왜 세그먼트 폴트가 발생할까?

하나씩 실험으로 밝혀보겠습니다.


## mov rax,1이 왜 mov eax,1이 될까?

mov eax, 1이 32비트 명령인지 64비트 명령인지 확인을 위해서 소스 첫 부분에 mov rax, 0x1234567812345678을 추가했습니다. 그리고 mov rax, 4를 합니다.


```
section .data

message:
db      'hello, world!', 0

section .text

global _start
_start:
    mov     rax, 0x1234567812345678
    mov     rax, 4
	mov     rdi, 1
	mov     rsi, message
	mov     rdx, 13
	syscall

	mov     rax, 1
	xor     rdi, rdi
	syscall
	ret
```

다시 빌드해서 gdb를 실행합니다.
```
$ nasm -f elf64 p.nasm
$ ld -o p p.o
$ gdb p
(gdb) disassemble _start
Dump of assembler code for function _start:
   0x00000000004000b0 <+0>:    movabs $0x1234567812345678,%rax
   0x00000000004000ba <+10>:    mov    $0x4,%eax
   0x00000000004000bf <+15>:    mov    $0x1,%edi
   0x00000000004000c4 <+20>:    movabs $0x6000e0,%rsi
   0x00000000004000ce <+30>:    mov    $0xd,%edx
   0x00000000004000d3 <+35>:    syscall 
   0x00000000004000d5 <+37>:    mov    $0x1,%eax
   0x00000000004000da <+42>:    xor    %rdi,%rdi
   0x00000000004000dd <+45>:    syscall 
   0x00000000004000df <+47>:    retq   
End of assembler dump.
```



ni명령으로 한줄씩 실행하면서 rax 레지스터의 값을 확인해봅니다.
```
(gdb) b _start
Breakpoint 1 at 0x4000b0
(gdb) r
Starting program: /home/gohkim/assembly_64bit/p 
Breakpoint 1, 0x00000000004000b0 in _start ()
(gdb) ni
0x00000000004000ba in _start ()
(gdb) info registers rax
rax            0x1234567812345678    1311768465173141112
(gdb) ni
0x00000000004000bf in _start ()
(gdb) info registers rax
rax            0x4    4
```
만약 mov    $0x4,%eax 명령이 정말로 eax 레지스터에만 값을 쓰는 명령이라면 rax의 값은 0x1234567800000004가 되야하는데 0x4가 되었습니다. 64비트 전체의 값이 바뀐 것입니다. 왜 그런지는 nasm메뉴얼 http://www.nasm.us/xdoc/2.11.08/html/nasmdo11.html#section-11.2 에 나와있습니다. 상수값이 64비트 값이거나 주소값이 아닌 경우에는 32비트 명령으로 어셈블링한다고 나와있습니다. 왜 그럴까요? 이유는 disassemble /r _start 명령을 실행해보면 알 수 있습니다.
```
(gdb) disassemble /r _start
Dump of assembler code for function _start:
   0x00000000004000b0 <+0>:    48 b8 78 56 34 12 78 56 34 12    movabs $0x1234567812345678,%rax
   0x00000000004000ba <+10>:    b8 04 00 00 00    mov    $0x4,%eax
   0x00000000004000bf <+15>:    bf 01 00 00 00    mov    $0x1,%edi
   0x00000000004000c4 <+20>:    48 be e0 00 60 00 00 00 00 00    movabs $0x6000e0,%rsi
   0x00000000004000ce <+30>:    ba 0d 00 00 00    mov    $0xd,%edx
   0x00000000004000d3 <+35>:    0f 05    syscall 
   0x00000000004000d5 <+37>:    b8 01 00 00 00    mov    $0x1,%eax
   0x00000000004000da <+42>:    48 31 ff    xor    %rdi,%rdi
   0x00000000004000dd <+45>:    0f 05    syscall 
   0x00000000004000df <+47>:    c3    retq   
End of assembler dump.
```



/r 명령은 기계어를 출력하는 명령입니다. 첫줄에 48 b8이 mov에 해당하는 기계여이고 그 다음 78 56 34 12 78 56 34 12가 메모리에 써진 상수값입니다. 리틀엔디안 기억하시나요? 그래서 거꾸로 표시된 것입니다. 64비트 레지스터에 값을 쓰는 명령은 총 길이가 10바이트입니다. mov $0x6000e0, %rsi 명령을 보시면 상수값이 64비트가 아니지만 주소값이므로 64비트 명령을 만들어서 총 길이가 10바이트인 명령어가 생성된 것을 알 수 있습니다. 반면에 32비트 명령어를 보시면 길이가 5바이트입니다. 

만약 모든 명령어를 64비트 명령어로 생성했다면 프로그램의 크기가 얼마나 커졌을까요? 프로그램이 커지면 왜 안좋은지 아시나요? 작은 프로그램이면 메모리에서 한번 읽어서 캐시 한두개에 들어갈게, 프로그램이 2배로 커지면 메모리에서 두번 읽어서 캐시 서너개에 들어가야합니다. 캐시를 많이 잡아먹을 수록 캐시 미스의 확률도 커지고 시스템 전체의 성능에 악영향을 줍니다. 지금은 작은 프로그램이니까 몇바이트 차이가 안나겠지만 시스템의 모든 프로그램이 다 2배로 커진다고 생각해보세요. 캐시도 2배로 큰 프로세서를 사야 동일한 성능이 나올 것이고, 같은 패밀리 프로세서인데 캐시가 2배인 프로세서의 값을 찾아보세요. 데스크탑을 조립해본 사람은 알겁니다.

결국 64비트 레지스터를 쓰는 명령이지만 32비트 레지스터로 표기함으로써 명령어 길이가 짧아진다는 것을 알 수 있습니다. greet 프로그램도 확인해볼 수 있습니다.

objdump -S greet 명령을 실행해보세요. 엄청 긴 디스어셈블 코드가 나타납니다. 너무 기니까 mov 만 비교해보겠습니다.
```
$ objdump -s greet | grep mov 
  4003e4:    48 8b 05 0d 0c 20 00     mov    0x200c0d(%rip),%rax        # 600ff8 <_DYNAMIC+0x1d0>
  400442:    49 89 d1                 mov    %rdx,%r9
  400446:    48 89 e2                 mov    %rsp,%rdx
  40044f:    49 c7 c0 e0 05 40 00     mov    $0x4005e0,%r8
  400456:    48 c7 c1 70 05 40 00     mov    $0x400570,%rcx
  40045d:    48 c7 c7 52 05 40 00     mov    $0x400552,%rdi
  400470:    b8 57 10 60 00           mov    $0x601057,%eax
  400480:    48 89 e5                 mov    %rsp,%rbp
  400485:    b8 00 00 00 00           mov    $0x0,%eax
  400490:    bf 50 10 60 00           mov    $0x601050,%edi
  4004b0:    be 50 10 60 00           mov    $0x601050,%esi
  4004c1:    48 89 e5                 mov    %rsp,%rbp
  4004c4:    48 89 f0                 mov    %rsi,%rax
  4004d3:    b8 00 00 00 00           mov    $0x0,%eax
  4004de:    bf 50 10 60 00           mov    $0x601050,%edi
  4004fa:    48 89 e5                 mov    %rsp,%rbp
  400503:    c6 05 46 0b 20 00 01     movb   $0x1,0x200b46(%rip)        # 601050 <__TMC_END__>
  400510:    bf 20 0e 60 00           mov    $0x600e20,%edi
  400520:    b8 00 00 00 00           mov    $0x0,%eax
  40052b:    48 89 e5                 mov    %rsp,%rbp
  400541:    48 bf 40 10 60 00 00     movabs $0x601040,%rdi
  400553:    48 89 e5                 mov    %rsp,%rbp
  40055b:    b8 00 00 00 00           mov    $0x0,%eax
  400574:    41 89 ff                 mov    %edi,%r15d
  40058b:    49 89 f6                 mov    %rsi,%r14
  40058e:    49 89 d5                 mov    %rdx,%r13
  4005b0:    4c 89 ea                 mov    %r13,%rdx
  4005b3:    4c 89 f6                 mov    %r14,%rsi
  4005b6:    44 89 ff                 mov    %r15d,%edi
```

대강 살펴보면 같은 mov 명령이라도 레지스터가 rax와 같이 64비트로 표시된게 있고 eax같이 32비트로 표시된게 있습니다. 각각 7바이트 5바이트로 길이가 다릅니다. 64비트로 표시된 mov 명령은 상수값이 그냥 숫자가 아니라 주소값이라는 의미입니다.



##fd가 0이어도 메세지 출력이 될까?

이런 소스를 만들어서 write 시스템콜이 fd가 0일때도 메세지를 출력하는지 확인해보겠습니다.

```
#include <unistd.h>
#include <sys/syscall.h>   /* For SYS_xxx definitions */

int main(void)
{
    syscall(1, 1, "hello", 6);    
    return 0;
}
$ gcc a.c
$ ./a.out
hello$ 
```

출력이 되네요.

커널 소스를 보니 확실하진않지만 stdin, stdout일 경우 번호와는 상관이 없이 동작하게 구현이 된것 같습니다. 어쨌든 되는건 되는거니까 그냥 구현상 그렇게 되도록 만들거라고 생각해야될것 같습니다.



##왜 세그먼트 폴트가 발생할까?

원래 블로그 저자의 의도는 exit 시스템콜을 호출해서 프로그램을 종료하는 것이었습니다. 그런데 BSD와 리눅스의 시스템콜 테이블의 차이를 생각하지않고 BSD에서만 동작하는 소스를 우리는 리눅스에서 실행시킨 것이었습니다. 그럼 exit 시스템콜이 없는 프로그램은 어떻게되나 한번 끝까지 실행시켜볼까요?

아래는 제가 세그먼트 폴트가 발생할때까지 ni를 실행한 결과입니다.
```
(gdb) disassemble _start
Dump of assembler code for function _start:
   0x00000000004000b0 <+0>:    movabs $0x1234567812345678,%rax
   0x00000000004000ba <+10>:	mov    $0x4,%eax
   0x00000000004000bf <+15>:	mov    $0x1,%edi
   0x00000000004000c4 <+20>:	movabs $0x6000e0,%rsi
   0x00000000004000ce <+30>:	mov    $0xd,%edx
   0x00000000004000d3 <+35>:	syscall 
   0x00000000004000d5 <+37>:	mov    $0x1,%eax
   0x00000000004000da <+42>:	xor    %rdi,%rdi
   0x00000000004000dd <+45>:	syscall 
   0x00000000004000df <+47>:	retq   
End of assembler dump.
(gdb) b *0x4000df
Breakpoint 1 at 0x4000df
(gdb) r
Starting program: /home/gohkim/assembly_64bit/p 
hello, world!
Breakpoint 1, 0x00000000004000df in _start ()
(gdb) info registers 
rax            0xd	13
rbx            0x0	0
rcx            0x4000df	4194527
rdx            0xd	13
rsi            0x6000e0	6291680
rdi            0x0	0
rbp            0x0	0x0
rsp            0x7fffffffdeb0	0x7fffffffdeb0
r8             0x0	0
r9             0x0	0
r10            0x0	0
r11            0x246	582
r12            0x0	0
r13            0x0	0
r14            0x0	0
r15            0x0	0
rip            0x4000df	0x4000df <_start+47>
eflags         0x246	[ PF ZF IF ]
cs             0x33	51
ss             0x2b	43
ds             0x0	0
es             0x0	0
fs             0x0	0
gs             0x0	0
(gdb) ni
0x0000000000000001 in ?? ()
(gdb) ni

Program received signal SIGSEGV, Segmentation fault.
0x0000000000000001 in ?? ()
(gdb) info registers 
rax            0xd	13
rbx            0x0	0
rcx            0x4000df	4194527
rdx            0xd	13
rsi            0x6000e0	6291680
rdi            0x0	0
rbp            0x0	0x0
rsp            0x7fffffffdeb8	0x7fffffffdeb8
r8             0x0	0
r9             0x0	0
r10            0x0	0
r11            0x246	582
r12            0x0	0
r13            0x0	0
r14            0x0	0
r15            0x0	0
rip            0x1	0x1
eflags         0x10246	[ PF ZF IF RF ]
cs             0x33	51
ss             0x2b	43
ds             0x0	0
es             0x0	0
fs             0x0	0
gs             0x0	0
```
마지막에서 두번째 ni를 실행하니
```
0x0000000000000001 in ?? ()
```
이런 메세지가 나타났습니다. ni 명령은 실행하고나면 다음에 실행할 주소를 알려줍니다. 그러면 이 메세지는 다음에 실행할 주소가 0x1번지라는 것입니다. 0x1번지에 프로그램이 있을까요?
```
(gdb) x 0x1
0x1:    Cannot access memory at address 0x1
```
0x1번지에 뭐가 있나 보려했더니 접근할 수 없는 메모리라고 나오네요.

수많은 버그들이 엉뚱한 주소를 접근하기 때문에 발생하는데 0x0이나 0x1이나 0x5123같이 일반적인 데이터를 잘못써서 포인터 변수에 넣기때문에 발생합니다. 그래서 리눅스등 많은 운영체제가 시스템의 첫번째 페이지, 즉 주소로 따지면 0x0 ~ 0xFFF번지에 접근하면 세그먼트 폴트가 나도록해서 자동으로 버그를 감지할 수 있도록 해줍니다. 지금도 그런 경우입니다.

그러면 왜 ret 명령을 실행했더니 rip값이 1이 되었을까요? ret 명령은 무슨 일을 하나요? 스택에서 다음에 실행할 명령의 주소를 꺼내오는 일을 합니다. rip가 1이 되었다는 것은 결국 스택에 1이 저장되어있었고, ret명령으로 그 값이 rip에 저장되었다는 것을 의미합니다.

그럼 gdb를 다시 실행해서 정말 스택에 1이 저장되어있었는지 확인해볼까요?
```
(gdb) disassemble 0x4000df
Dump of assembler code for function _start:
   0x00000000004000b0 <+0>:    movabs $0x1234567812345678,%rax
   0x00000000004000ba <+10>:	mov    $0x4,%eax
   0x00000000004000bf <+15>:	mov    $0x1,%edi
   0x00000000004000c4 <+20>:	movabs $0x6000e0,%rsi
   0x00000000004000ce <+30>:	mov    $0xd,%edx
   0x00000000004000d3 <+35>:	syscall 
   0x00000000004000d5 <+37>:	mov    $0x1,%eax
   0x00000000004000da <+42>:	xor    %rdi,%rdi
   0x00000000004000dd <+45>:	syscall 
=> 0x00000000004000df <+47>:	retq   
End of assembler dump.
(gdb) info registers rsp
rsp            0x7fffffffdeb0	0x7fffffffdeb0
(gdb) x 0x7fffffffdeb0
0x7fffffffdeb0:	0x00000001
```
rsp값을 확인해서 메모리를 읽어보니 정말 1이 들어있었습니다. 어디서 1을 저장한건지는 더 디버깅을 해봐야겠지만 어쨌든 잘못된 지점에서 함수가 종료되었다는 것을 알 수 있습니다.

나중에 개발하다가 이상한 주소로 점프하는 버그가 생긴다면 스택을 조사하고 함수의 종료가 제대로 처리되는지를 조사하면 된다는 것을 알 수 있습니다. 어디선가 다른 함수의 스택을 덮어쓰고, 그래서 함수가 종료될때 엉뚱한 주소로 점프하게되는게 보통의 경우입니다. 어디서 스택을 덮어쓰는지는 valgrind나 asan등의 툴을 이용하면 찾을 수 있습니다.

참고로 이상없이 동작하는 소스는 참고한 블로그의 댓글에 있습니다.



분량이 좀 길어졌지만 나름 실전적인 내용이 들어가게되서 재밌으셨을거라 믿습니다. 이제 대강 64비트의 맛을 봤으니 다음은 조금 원론적인 이야기를 하겠습니다.