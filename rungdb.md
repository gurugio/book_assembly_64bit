# gdb로 어셈블리 코드의 실행 확인

emu8086은 정말 편리한 툴이지만 64비트 코드는 실행하지 못합니다. 64비트 코드를 실제 우리 데스크탑에서 실행시키면서 동시에 에물레이터가 아닌 진짜 하드웨어의 상태 정보를 보여주는 툴이 있는데 바로 그 유명한 gdb입니다.

일단 gdb를 실행합니다. (만약 실행이 안되면 구글링하셔서 설치해주세요.) 우리가 만든 2개의 프로그램중에 먼저 greet 실행파일의 어셈블리 코드를 확인해보겠습니다.
```
$ gdb greet
```
(gdb)라는 프롬프트가 나타납니다. 그 전에 (no debugging symbols found) 라는 메세지가 나오는데 우리는 기계어를 쓰는 사람보다 기계에 가까운 사람이므로 사람들만 쓰는 심볼같은 정보따위는 필요없습니다. 

gdb가 C코드 부분은 건너뛰고 asm_main부터 실행하도록 b asm_main 명령과 r 명령을 실행합니다. 각 명령에 대한 설명은 하지 않겠습니다. gdb 사용법에 대한 문서는 흘러 넘치니까요.

r 명령을 실행하면 우리는 asm_main의 시작 지점에 서있게 됩니다. 진짜 그런가 확인하려면 어떻게 해야할까요?

disassemble /m asm_main 명령을 실행하면 gdb가 친절하게 => 표시를 해줍니다. 바로 그 지점이 프로세서가 실행할 지점이라는 의미입니다.
```
(gdb) b asm_main
Breakpoint 1 at 0x400540
(gdb) r
Starting program: /home/gohkim/assembly_64bit/greet 

Breakpoint 1, 0x0000000000400540 in asm_main ()
(gdb) disassemble /m asm_main
Dump of assembler code for function asm_main:
=> 0x0000000000400540 <+0>:    push   %rdi
   0x0000000000400541 <+1>:    movabs $0x601040,%rdi
   0x000000000040054b <+11>:    callq  0x400410 <printf@plt>
   0x0000000000400550 <+16>:    pop    %rdi
   0x0000000000400551 <+17>:    retq   
End of assembler dump.
```
r 명령으로 프로그램을 실행하면 0x0000000000400540 지점에서 멈췄다는 설명이 나옵니다. => 표시가 된 지점의 주소를 읽어보세요. 진짜로 gdb가 해당 지점에서 멈췄다는 것을 알 수 있습니다.

disassemble 명령으로 출력한 코드가 우리가 작성한 코드와 형식이 좀 다릅니다. 이것은 gcc나 gdb가 사용하는 at&t라는 문법입니다. nasm은 표준 intel 문법을 사용합니다. 뭐가 다르냐면

1. 레지스터 이름 앞에 % 표시하기, 숫자에는 $ 표시하기

2. 인자 순서가 반대: nasm에서는 mov rdi, 0x1234 라고 써야할 것을 at&t문법으로는 mov $0x1234, %rdi가 됩니다.

3. 명령어끝에 q, abs, l 같은 레지스터 크기 정보 출력하기: movq는 64비트, movl은 32비트, movb는 8비트 같은 식으로 각 명령어가 어떤 크기의 레지스터나 상수 값을 사용할 것인지를 명령어에 같이 표시합니다.

복잡하게 생각하지마세요. 그냥 우리가 nasm으로 빌드한 코드와 비교하면서 다른 점만 확인하면 됩니다. 표기만 다르니 어렵게 생각할 필요없습니다.

이 시점에서 해야할 일이 하나 있습니다. info registers 명령을 실행해보세요. 앞으로 실행될 명령의 주소를 가지고있는 레지스터가 ip 레지스터라고 알고계실겁니다. info registers 명령으로 레지스터 값들을 출력했습니다. 그중에 ip 레지스터가 있나요? 있긴 있습니다. rip라고 이름이 살짝 바뀌었습니다. 결국 우리가 아는 다른 레지스터들도 이름에 r 표시가 붙었습니다. 64비트가 되면서 어떻게 바뀌었는지 감이 오지 않으세요? 레지스터 이름이 바꼈습니다. 이름이 완전히 바뀐게 아니고 이름 앞에 r만 붙었습니다. 지금은 일단 그정도만 알고 넘어가겠습니다. 다음에 64비트와 32비트, 16비트의 차이만 자세히 설명하겠습니다. 지금은 gdb로 우리가 만든 코드를 확인하는 방법을 아는게 중요합니다. 그래야 다음에 32비트, 16비트 코드를 만들어서 직접 비교하고 실험할 수 있으니까요.

rip값을 확인하셨나요? 지금 실행되야될 지점이 맞나요?

그럼 다음 명령인 push rdi 명령을 실행해보겠습니다. gdb에서 C 코드를 디버깅할 때는 다음 줄로 넘어갈때 next 혹은 n이라고 입력했습니다. 어셈블리에서는 nexti나 ni라고 입력하면 됩니다.


```
(gdb) nexti
0x0000000000400541 in asm_main ()
(gdb) disassemble /m 0x0000000000400541
Dump of assembler code for function asm_main:
   0x0000000000400540 <+0>:    push   %rdi
=> 0x0000000000400541 <+1>:    movabs $0x601040,%rdi
   0x000000000040054b <+11>:    callq  0x400410 <printf@plt>
   0x0000000000400550 <+16>:    pop    %rdi
   0x0000000000400551 <+17>:    retq   
End of assembler dump.
```

nexti명령을 실행하면 새로운 실행 지점의 주소가 나타납니다. 그리고 그 주소에 뭐가 있는지는 disassemble 명령으로 확인할 수 있습니다.

어떤가요? push rdi 명령이 실행됐나요?

push 명령이 하는 일이 뭔지 기억이 나시나요? 스택에 레지스터 값을 저장하는 명령입니다. 스택에 뭔가 저장되면 어떤 일이 생기나요? 스택 포인터 레지스터값이 바뀝니다.

그럼 스택 포인터 값이 바꼈는지 info registers 명령으로 확인해보시기 바랍니다. 해보세요. 제가 일일이 제 실험 결과를 여기에 복사해넣지 않겠습니다. 어떤 값에서 어떤 값으로 바꼈는지 확인해보세요. 그리고 값이 얼마 차이가 나는지 생각해보세요. 우리가 스택에 몇 바이트를 집어넣었나요? 스택 포인터의 값 차이가 얼마인가요? 우리는 몇 비트 환경에서 동작하고 있나요?

남은 일은 nexti, disassemble, info registers 명령을 반복하는 것입니다. emu8086으로 실습할 때도 한줄한줄 실행하면서 레지스터 값이 어떻게 바뀌는지를 봤습니다. 똑같은 실험을 하는 것입니다. 단지 GUI가 없고 터미널에서 동작하는 차이뿐입니다. 솔직히 지금 우리가 에물레이터로 실행하는 것인지, 진짜 프로세서에서 동작하는 결과를 보는 것인지도 헷갈립니다.

GUI로 뭔가 보고싶으신 분은 DDD라는 전통있는 툴을 사용하시면 됩니다. emu8086처럼 화면에 레지스터 값들을 띄어놓고 값이 바뀌는 것을 확인할 수 있습니다. 구글로 검색해보시고 관련 문서들을 읽어보세요.

아래는 제가 실험한 결과입니다. 참고하세요.
```
$ gdb greet
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
Reading symbols from greet...(no debugging symbols found)...done.
(gdb) l
No symbol table is loaded.  Use the "file" command.
(gdb) b asm_main
Breakpoint 1 at 0x400540
(gdb) r
Starting program: /home/gohkim/assembly_64bit/greet 

Breakpoint 1, 0x0000000000400540 in asm_main ()
(gdb) disassemble /m asm_main
Dump of assembler code for function asm_main:
=> 0x0000000000400540 <+0>:    push   %rdi
   0x0000000000400541 <+1>:	movabs $0x601040,%rdi
   0x000000000040054b <+11>:	callq  0x400410 <printf@plt>
   0x0000000000400550 <+16>:	pop    %rdi
   0x0000000000400551 <+17>:	retq   
End of assembler dump.
(gdb) info registers 
rax            0x400552	4195666
rbx            0x0	0
rcx            0x0	0
rdx            0x7fffffffdec8	140737488346824
rsi            0x7fffffffdeb8	140737488346808
rdi            0x1	1
rbp            0x7fffffffddd0	0x7fffffffddd0
rsp            0x7fffffffddc8	0x7fffffffddc8
r8             0x7ffff7dd4dd0	140737351863760
r9             0x7ffff7de99d0	140737351948752
r10            0x833	2099
r11            0x7ffff7a2f950	140737348041040
r12            0x400440	4195392
r13            0x7fffffffdeb0	140737488346800
r14            0x0	0
r15            0x0	0
rip            0x400540	0x400540 <asm_main>
eflags         0x246	[ PF ZF IF ]
cs             0x33	51
ss             0x2b	43
ds             0x0	0
es             0x0	0
fs             0x0	0
gs             0x0	0
(gdb) ni
0x0000000000400541 in asm_main ()
(gdb) disassemble /m asm_main
Dump of assembler code for function asm_main:
   0x0000000000400540 <+0>:	push   %rdi
=> 0x0000000000400541 <+1>:	movabs $0x601040,%rdi
   0x000000000040054b <+11>:	callq  0x400410 <printf@plt>
   0x0000000000400550 <+16>:	pop    %rdi
   0x0000000000400551 <+17>:	retq   
End of assembler dump.
(gdb) info registers 
rax            0x400552	4195666
rbx            0x0	0
rcx            0x0	0
rdx            0x7fffffffdec8	140737488346824
rsi            0x7fffffffdeb8	140737488346808
rdi            0x1	1
rbp            0x7fffffffddd0	0x7fffffffddd0
rsp            0x7fffffffddc0	0x7fffffffddc0
r8             0x7ffff7dd4dd0	140737351863760
r9             0x7ffff7de99d0	140737351948752
r10            0x833	2099
r11            0x7ffff7a2f950	140737348041040
r12            0x400440	4195392
r13            0x7fffffffdeb0	140737488346800
r14            0x0	0
r15            0x0	0
rip            0x400541	0x400541 <asm_main+1>
eflags         0x246	[ PF ZF IF ]
cs             0x33	51
ss             0x2b	43
ds             0x0	0
es             0x0	0
fs             0x0	0
gs             0x0	0
(gdb) ni
0x000000000040054b in asm_main ()
(gdb) disassemble /m asm_main
Dump of assembler code for function asm_main:
   0x0000000000400540 <+0>:	push   %rdi
   0x0000000000400541 <+1>:	movabs $0x601040,%rdi
=> 0x000000000040054b <+11>:	callq  0x400410 <printf@plt>
   0x0000000000400550 <+16>:	pop    %rdi
   0x0000000000400551 <+17>:	retq   
End of assembler dump.
(gdb) info registers 
rax            0x400552	4195666
rbx            0x0	0
rcx            0x0	0
rdx            0x7fffffffdec8	140737488346824
rsi            0x7fffffffdeb8	140737488346808
rdi            0x601040	6295616
rbp            0x7fffffffddd0	0x7fffffffddd0
rsp            0x7fffffffddc0	0x7fffffffddc0
r8             0x7ffff7dd4dd0	140737351863760
r9             0x7ffff7de99d0	140737351948752
r10            0x833	2099
r11            0x7ffff7a2f950	140737348041040
r12            0x400440	4195392
r13            0x7fffffffdeb0	140737488346800
r14            0x0	0
r15            0x0	0
rip            0x40054b	0x40054b <asm_main+11>
eflags         0x246	[ PF ZF IF ]
cs             0x33	51
ss             0x2b	43
ds             0x0	0
es             0x0	0
fs             0x0	0
gs             0x0	0
(gdb) ni
0x0000000000400550 in asm_main ()
(gdb) disassemble /m asm_main
Dump of assembler code for function asm_main:
   0x0000000000400540 <+0>:	push   %rdi
   0x0000000000400541 <+1>:	movabs $0x601040,%rdi
   0x000000000040054b <+11>:	callq  0x400410 <printf@plt>
=> 0x0000000000400550 <+16>:	pop    %rdi
   0x0000000000400551 <+17>:	retq   
End of assembler dump.
(gdb) info registers 
rax            0xd	13
rbx            0x0	0
rcx            0x400	1024
rdx            0x7ffff7dd5970	140737351866736
rsi            0x7ffff7ff5000	140737354092544
rdi            0x7ffff7ff500d	140737354092557
rbp            0x7fffffffddd0	0x7fffffffddd0
rsp            0x7fffffffddc0	0x7fffffffddc0
r8             0xffffffff	4294967295
r9             0x0	0
r10            0x22	34
r11            0x246	582
r12            0x400440	4195392
r13            0x7fffffffdeb0	140737488346800
r14            0x0	0
r15            0x0	0
rip            0x400550	0x400550 <asm_main+16>
eflags         0x206	[ PF IF ]
cs             0x33	51
ss             0x2b	43
ds             0x0	0
es             0x0	0
fs             0x0	0
gs             0x0	0
(gdb) ni
0x0000000000400551 in asm_main ()
(gdb) disassemble /m asm_main
Dump of assembler code for function asm_main:
   0x0000000000400540 <+0>:	push   %rdi
   0x0000000000400541 <+1>:	movabs $0x601040,%rdi
   0x000000000040054b <+11>:	callq  0x400410 <printf@plt>
   0x0000000000400550 <+16>:	pop    %rdi
=> 0x0000000000400551 <+17>:	retq   
End of assembler dump.
(gdb) ni
0x000000000040055b in main ()
(gdb) info registers 
rax            0xd	13
rbx            0x0	0
rcx            0x400	1024
rdx            0x7ffff7dd5970	140737351866736
rsi            0x7ffff7ff5000	140737354092544
rdi            0x1	1
rbp            0x7fffffffddd0	0x7fffffffddd0
rsp            0x7fffffffddd0	0x7fffffffddd0
r8             0xffffffff	4294967295
r9             0x0	0
r10            0x22	34
r11            0x246	582
r12            0x400440	4195392
r13            0x7fffffffdeb0	140737488346800
r14            0x0	0
r15            0x0	0
rip            0x40055b	0x40055b <main+9>
eflags         0x206	[ PF IF ]
cs             0x33	51
ss             0x2b	43
ds             0x0	0
es             0x0	0
fs             0x0	0
gs             0x0	0
(gdb) q
A debugging session is active.

    Inferior 1 [process 18702] will be killed.

Quit anyway? (y or n) y
```

한가지 안한게 있습니다. 스택에 rdi 레지스터의 값을 저장해놓고 스택 메모리에 진짜 rdi 값이 들어갔는지를 확인하지 않았습니다. gdb에서 메모리 값을 출력하는 명령은 x 입니다. x <주소값> 을 입력하면 됩니다. 입력해보세요. rdi에 어떤 값이 있었는데 어떤 값이 출력되었나요? 주소값을 바꿔가면서 어떤 값이 스택에 있는지를 확인해보세요.

