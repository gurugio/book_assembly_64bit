#gdb로 어셈블리 코드 디버깅하기

이전에 실행했던 p 프로그램이 메세지는 출력하는데 세그먼트 폴트가 났던걸 기억하실겁니다. 뭐가 문제일까요? 어셈블리 코드를 디버깅하는 실습을 해보겠습니다.

```
(gdb) disassemble _start
```
_start 라벨은 프로그램의 시작 지점으로 우리가 지정한 위치입니다.
```
_start:
```
코드에 이렇게 표기한 지점입니다. 이 라벨은 가장 가까운 명령어와 같은 위치를 표시해주는 역할을 합니다. jmp나 call 명령으로 점프할 위치를 알려주는 일을 한다는 것은 이전 강좌에서 설명했습니다. 그 외에도 지금처럼 디버거가 인식해서 주소값대신 위치를 표시할 수도 있습니다.

우리가 만든 어셈블리 코드가 어떤 주소에 저장되고 실행될지는 운영체제의 로더가 결정해주기 전에는 모릅니다. 따라서 disassemble 명령으로 내 코드를 보고싶을 때 어떤 주소값을 지정해야할지는 모릅니다. 항상 이렇게 라벨을 지정해야합니다. 함수 이름도 라벨에 한 종류입니다. 주소를 써도 됩니다만 가장 처음 디버깅을 시작할 때 아무런 정보도 없을 때는 _start나 main 라벨로 시작 주소를 알아낼 수 밖에 없습니다.

gdb가 디스어셈블한 코드를 한줄씩 보겠습니다.
```
   0x00000000004000b0 <+0>:    mov    $0x4,%eax
```
가장 앞에는 주소값이 보입니다. 그리고 +0 표시는 출력한 라벨에서 몇바이트 떨어진 위치인가를 말해줍니다. eax라는 레지스터에 4를 저장합니다. eax는 ax의 32비트 범위를 나타내는 이름입니다. rax는 64비트입니다. 원래 64비트 크기의 rax레지스터가 있고, 이 레지스터의 어떤 위치에 값을 저장할건가를 이름을 바꿔서 지정하는 것입니다. 16비트 ax레지스터에서 8비트 부분을 ah, al로 지정하는 것의 확장판입니다.

만약 64비트 전체 영역에 값을 쓰고 싶다면
```
mov rax, 0x1234567812345678
```
이렇게하면 rax레지스터 전체에 값을 씁니다. 그리고
```
mov eax, 0xFFFFFFFF
```
를 실행하면 rax 레지스터에는 0x12345678FFFFFFFF 값이 저장될 것입니다.
```
mov ax, 0x0
```
을 실행하면 rax레지스터의 값은 0x12345678FFFF0000 이 됩니다.



다음 명령은 
```
   0x00000000004000b5 <+5>:    mov    $0x1,%edi

   0x00000000004000ba <+10>:    movabs $0x6000d8,%rsi
```
1을 edi에 저장하고 rsi에는 0x6000D8을 저장합니다. movabs는 명령어 mov에 abs라는 postfix가 붙은 것입니다. 64비트 값을 레지스터에 쓰겠다는 의미입니다.

그리고 +5 +10 을 볼 수 있습니다. 시작 지점으로부터 5바이트 떨어진 위치라는 의미이니까 이전 명령어인 mov eax, 0x4가 5바이트 크기의 명령이라는 것을 알 수 있습니다. mov edi, 0x1도 5바이트 크기가 되네요.

의문이 생깁니다. 왜 0x6000D8은 64비트 레지스터에 쓰고 1은 32비트 레지스터에 쓸까요? 
```
   0x00000000004000c4 <+20>:    mov    $0xd,%edx
```
한줄 더 보겠습니다. 또 0xD 값을 32비트 레지스터에 씁니다.
```
   0x00000000004000c9 <+25>:    syscall
```
다음 줄은 syscall 명령입니다. syscall 명령은 한번 검색해볼 필요가 있습니다. 검색해서 무슨 일을 하는 것인지 확인해보세요. 그냥 syscall로 검색하면 C 라이브러리의 syscall 함수에 대한 설명이 더 많을 것입니다. man 페이지를 보면 ABI라는 설명이 나오고 레지스터 이름들이 나옵니다. assembly syscall로도 검색해보시고 위키피디아에서도 검색해보시기 바랍니다.

설명만 들어서는 잘 이해가 안되실 겁니다. syscall 명령은 시스템콜을 호출하는 명령입니다. 호출할 시스템 콜의 번호는 eax에 저장합니다. 시스템콜의 인자는 차례대로 rdi, rsi, rdx, r10...에 저장해야합니다.

우리가 호출할 시스템콜의 번호는 4번입니다. 그리고 함수 인자는 1, 문자열의 주소값, 0xd가 됩니다. 아직 시스템콜 4번이 어떤 함수인지는 찾지마시기 바랍니다. C 코드를 생각하면 메세지를 출력하는 write가 되야할것 같은데 정말 그런지 계속 실행하면서 알아보겠습니다.

(왜 write인가 printf가 아닌가 생각되시는 분들은 C 시스템 프로그래밍을 좀더 하시거나 아니면 그냥 그러려니 하시기 바랍니다. 이 강좌에서 필요한 내용은 아닙니다.)

ni (nexti) 명령을 실행하면 syscall을 실행하고 그 다음 줄로 넘어갑니다. 만약 제대로된 시스템 콜을 선택하고 제대로 인자를 전달했다면 화면에 메세지가 출력되야합니다.
```
(gdb) ni
0x00000000004000b5 in _start ()
(gdb) ni
0x00000000004000ba in _start ()
(gdb) ni
0x00000000004000c4 in _start ()
(gdb) ni
0x00000000004000c9 in _start ()
(gdb) ni
0x00000000004000cb in _start ()
(gdb) disassemble _start
Dump of assembler code for function _start:
   0x00000000004000b0 <+0>:    mov    $0x4,%eax
   0x00000000004000b5 <+5>:    mov    $0x1,%edi
   0x00000000004000ba <+10>:    movabs $0x6000d8,%rsi
   0x00000000004000c4 <+20>:    mov    $0xd,%edx
   0x00000000004000c9 <+25>:    syscall 
=> 0x00000000004000cb <+27>:    mov    $0x1,%eax
   0x00000000004000d0 <+32>:    xor    %rdi,%rdi
   0x00000000004000d3 <+35>:    syscall 
   0x00000000004000d5 <+37>:    retq   
End of assembler dump.
```




분명히 syscall을 실행했는데도 hello, world!가 출력되지 않습니다. 분명히 프로그램을 바로 실행했을 때 메세지가 출력되었으므로 어딘가 메세지를 출력하는 명령이 있을 것입니다. 그럼 계속 실행해보겠습니다.
```
(gdb) ni
0x00000000004000d0 in _start ()
(gdb) ni
0x00000000004000d3 in _start ()
(gdb) disassemble _start
Dump of assembler code for function _start:
   0x00000000004000b0 <+0>:    mov    $0x4,%eax
   0x00000000004000b5 <+5>:    mov    $0x1,%edi
   0x00000000004000ba <+10>:    movabs $0x6000d8,%rsi
   0x00000000004000c4 <+20>:    mov    $0xd,%edx
   0x00000000004000c9 <+25>:    syscall 
   0x00000000004000cb <+27>:    mov    $0x1,%eax
   0x00000000004000d0 <+32>:    xor    %rdi,%rdi
=> 0x00000000004000d3 <+35>:    syscall 
   0x00000000004000d5 <+37>:    retq   
End of assembler dump.
(gdb) ni
hello, world!0x00000000004000d5 in _start ()
(gdb) disassemble _start
Dump of assembler code for function _start:
   0x00000000004000b0 <+0>:    mov    $0x4,%eax
   0x00000000004000b5 <+5>:    mov    $0x1,%edi
   0x00000000004000ba <+10>:    movabs $0x6000d8,%rsi
   0x00000000004000c4 <+20>:    mov    $0xd,%edx
   0x00000000004000c9 <+25>:    syscall 
   0x00000000004000cb <+27>:    mov    $0x1,%eax
   0x00000000004000d0 <+32>:    xor    %rdi,%rdi
   0x00000000004000d3 <+35>:    syscall 
=> 0x00000000004000d5 <+37>:    retq   
End of assembler dump.
(gdb) 
```


다음 syscall 명령까지 실행했습니다. 이번 syscall에서 hello, world! 메세지가 출력되었습니다. 이번에 실행한 시스템콜은 몇번인가요? eax에 1을 저장했으니 1번 시스템콜이 호출될 것입니다. 1번 시스템 콜이 무엇인지 이번에는 검색해볼까요? "linux system call table" 을 검색해보시기 바랍니다. 주의할 것은 32비트 버전 리눅스 커널과 64비트 버전 리눅스 커널의 시스템콜 번호가 다르다는 것입니다. x86_64 버전을 찾으시면 64비트 버전 리눅스 커널에 해당하는 내용입니다.

x86은 인텔 프로세서를 말하는 것이고 _64는 64버전이라는 표시입니다. AMD64라고도 표시하는데 왜냐면 AMD에서 64비트 아키텍쳐를 만들었기 때문입니다. 인텔이 Itanium이라는 새로운 아키텍쳐를 시도할 때 AMD는 기존 32비트 아키텍쳐를 발전시킨 AMD64를 발표했습니다. 당연히 새로운 아키텍쳐보다는 기존 것들이 호환되는 아키텍쳐가 더 선호되겠지요. Itanium의 성능이 그리 좋지 않기도 했다고 합니다. 그래서 결국 AMD64가 시장 표준이 되었고, 인텔도 같은 아키텍쳐를 따라가서 x86_64로 통일하게 되었습니다. 그래서 x86이라고하면 보통 32비트를 부르고 x86_64나 AMD64라고하면 64비트 버전을 말하게 되었습니다.

결과적으로 1번 시스템 콜은 write였습니다. 아까 4번 시스템콜은 stat이었습니다. 그러니 메세지 출력이 안되었던 것입니다. write 함수의 선언을 확인해볼까요. 터미널에서 man 2 write 명령을 실행하면 볼 수 있습니다.
```
ssize_t write(int fd, const void *buf, size_t count);
```
시스템콜의 인자가 순서대로 rdi, rsi, rdx에 저장되야한다고 설명했습니다. 즉 fd를 rdi에 저장하고 메세지의 포인터를 rsi에 저장하고 메세지 길이를 rdx에 저장해야 합니다.

우리 코드는 어떤까요? rdi, rsi, rdx 에 뭐가 저장된 상태에서 두번째 syscall이 호출된 걸까요? 확인을 위해서 다시한번 gdb를 실행시킵니다. 이번에는 두번째 syscall이 호출되기 전에 gdb가 멈추도록 b *<주소> 명령을 먼저 실행하고 r을 실행합니다.


```
(gdb) b *0x4000d3
Breakpoint 1 at 0x4000d3
(gdb) r
Starting program: /home/gohkim/assembly_64bit/p 

Breakpoint 1, 0x00000000004000d3 in _start ()

(gdb) disassemble 0x4000d3
Dump of assembler code for function _start:
   0x00000000004000b0 <+0>:    mov    $0x4,%eax
   0x00000000004000b5 <+5>:    mov    $0x1,%edi
   0x00000000004000ba <+10>:    movabs $0x6000d8,%rsi
   0x00000000004000c4 <+20>:    mov    $0xd,%edx
   0x00000000004000c9 <+25>:    syscall 
   0x00000000004000cb <+27>:    mov    $0x1,%eax
   0x00000000004000d0 <+32>:    xor    %rdi,%rdi
=> 0x00000000004000d3 <+35>:    syscall 
   0x00000000004000d5 <+37>:    retq   
End of assembler dump.
```


지금 상태에서 레지스터 값들을 보겠습니다.
```
(gdb) info registers 
rax            0x1    1
rbx            0x0    0
rcx            0x4000cb    4194507
rdx            0xd    13
rsi            0x6000d8    6291672
rdi            0x0    0

......
```
시스템콜 번호 rax는 1이고, fd값은 0, 메세지 주소는 0x6000d8, 길이는 0xd가 전달됩니다. 결국 write가 실행될때 메세지 주소가 제대로 넘어가긴 하니까 메시지가 출력되긴했습니다.

저는 이 프로그램을 디버깅하면서 3가지 의문이 생겼습니다.

1. 어셈블리 프로그래밍은 mov rax, 4로 구현했는데 왜 gdb에서는 mov  eax, 1이 되서 32비트 레지스터에 1을 저장하고있을까?
2. fd 값이 0이면 stdin을 의미해서 메세지가 출력이 안되야될것 같은데 왜 출력이 될까?
3. 왜 세그먼트 폴트가 발생할까?

분량이 너무 길어졌으니 다음 장에서 계속하겠습니다.




```
gohkim@ws00837:~/assembly_64bit$ gdb p
GNU gdb (Ubuntu 7.10-1ubuntu2) 7.10
Copyright (C) 2015 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from p...(no debugging symbols found)...done.
(gdb) disassemble 
No frame selected.
(gdb) list
No symbol table is loaded.  Use the "file" command.
(gdb) ni
The program is not being run.
(gdb) disassemble _start
Dump of assembler code for function _start:
   0x00000000004000b0 <+0>:    mov    $0x4,%eax
   0x00000000004000b5 <+5>:	mov    $0x1,%edi
   0x00000000004000ba <+10>:	movabs $0x6000d8,%rsi
   0x00000000004000c4 <+20>:	mov    $0xd,%edx
   0x00000000004000c9 <+25>:	syscall 
   0x00000000004000cb <+27>:	mov    $0x1,%eax
   0x00000000004000d0 <+32>:	xor    %rdi,%rdi
   0x00000000004000d3 <+35>:	syscall 
   0x00000000004000d5 <+37>:	retq   
End of assembler dump.
(gdb) b _start
Breakpoint 1 at 0x4000b0
(gdb) r
Starting program: /home/gohkim/assembly_64bit/p 

Breakpoint 1, 0x00000000004000b0 in _start ()
(gdb) ni
0x00000000004000b5 in _start ()
(gdb) ni
0x00000000004000ba in _start ()
(gdb) ni
0x00000000004000c4 in _start ()
(gdb) ni
0x00000000004000c9 in _start ()
(gdb) ni
0x00000000004000cb in _start ()
(gdb) ni
0x00000000004000d0 in _start ()
(gdb) ni
0x00000000004000d3 in _start ()
(gdb) ni
hello, world!0x00000000004000d5 in _start ()
(gdb) disassemble _start
Dump of assembler code for function _start:
   0x00000000004000b0 <+0>:	mov    $0x4,%eax
   0x00000000004000b5 <+5>:	mov    $0x1,%edi
   0x00000000004000ba <+10>:	movabs $0x6000d8,%rsi
   0x00000000004000c4 <+20>:	mov    $0xd,%edx
   0x00000000004000c9 <+25>:	syscall 
   0x00000000004000cb <+27>:	mov    $0x1,%eax
   0x00000000004000d0 <+32>:	xor    %rdi,%rdi
   0x00000000004000d3 <+35>:	syscall 
=> 0x00000000004000d5 <+37>:	retq   
End of assembler dump.
(gdb) ni
0x0000000000000001 in ?? ()
(gdb) quit
A debugging session is active.

	Inferior 1 [process 21641] will be killed.

Quit anyway? (y or n) y
[1]+  Done                    emacs p.nasm
```

gdb에 출력되는 주소값등 몇몇 숫자는 시스템 환경에 따라 다를 수 있습니다.