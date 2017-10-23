#64비트 함수 호출 규약

대부분의 언어들이 함수를 호출할 때 반환값과 인자값이 있습니다. 컴파일러가 최종 기계코드를 만들 때 반환값과 인자값을 어디에 저장해야할지도 규칙이 있습니다. 규칙이 없으면 같은 64비트 리눅스라고 해도 배포판마다 컴파일러가 달라야할 수도 있으니까요.

nasm메뉴얼에는 사실 지금까지 제가 강좌로 쓴 내용들 대부분이 포함되어있습니다. 언어가 영어고 초보자가 보기에 쉽게 쓴게 아니라 참고하기쉽게 메뉴얼 형식으로 쓴 거라 한눈에 들어오지 않을 뿐입니다. 마찬가지로 함수 호출 규약에 대해서도 설명이 있습니다.

http://www.nasm.us/xdoc/2.11.08/html/nasmdo11.html

여기 나오는 내용을 간단하게 설명하고 예제 코드를 만들어보겠습니다. 사실 처음에 이미 예제 코드를 만들었었습니다. Hello, world!를 출력하기위해 시스템콜도 호출하고 printf도 호출했었는데요 그때 인자값들을 레지스터에 저장했었는데 이게 아무렇게나한게 아니라 이 규약을 따랐던 것입니다.

함수 호출 규약을 다시 한번 정리하면

* nasm이냐 at&t냐 같은 어셈블러나 문법에 상관없이 운영체제가 만든 규약이다
함수 인자가 6개 이내면 RDI, RSI, RDX, RCX, R8, R9에 순서대로 저장한다
6개가 넘으면 스택에 저장한다
* RAX, R10, R11는 호출되는 함수가 맘대로 써도 된다. 따라서 함수 호출전에 스택에 저장하고 함수 호출 후에 스택에서 원래 값을 꺼내는 절차가 필요하다.
* 반환값은 RAX, RDX에 순서대로 저장한다. C언어는 반환값이 1개이므로 보통 RAX를 사용합니다. 어셈블리로 만든 함수를 어셈블리에서 호출하면 반환값을 2개이상 만들 수 있겠네요.
  * 컴파일러에 따라 128비트 반환값을 지원하기도 합니다. 그러면 RAX만으로 표현이 안되니까 두개의 레지스터를 사용합니다. RAX:RDX를 사용할 수도 있고, 그건 컴파일러마다 다르겠지요.
* 부동소수점 인자를 전달하는데도 규칙이 있다. 어떤 규칙인지는 일일이 설명안해도 아시겠지요?
* void foo(long a, double b, int c) 이런 함수를 호출하게되면 a는 RDI에, b는 XMM0, c는 ESI에 저장된다. c가 왜 esi냐면 int가 32비트라서 그렇습니다.

이제 Hello, world! 예제를 조금 고쳐서 printf에 3개의 인자를 넘기는 예제 소스를 만들어보겠습니다.

call.asm 소스입니다.
```
; To create executable:
; nasm -f elf64 call.asm
; gcc -o call call.o main_call.c

segment .data

    outmsg db "Hello, %s:%d!", 0xa, 0
	nation db "Germany", 0
	year   dd 2016

segment .bss

segment .text
	global  asm_main ; other modules can call asm_main
	extern printf

asm_main:
	push rdi
	push rsi
	push rdx

	mov rdi, outmsg
	mov rsi, nation
	mov edx, dword [year]
	call printf

	pop rdx
	pop rsi
	pop rdi
	ret	
```

call_main.c 소스입니다.


```
#include <stdio.h>

int asm_main(void);

int main(void)
{
    int ret;
	ret = asm_main();
	printf("print %d-bytes\n", ret);
	return 0;
}
```

별다른거 없습니다. 잘 이해가 안되시면 C소스로 printf("Hello, %s:%d!", "Germany", 2016); 을 생각해보세요. 3개의 인자가 각각 어디로 저장되야하는지 생각해보세요. 그리고 asm_main함수의 반환값은 printf의 반환값 그대로라고 생각하시면 됩니다. gdb로 한줄씩 실행해가면서 레지스터 값들이 어떻게 바뀌는지 확인해보세요.


```
(gdb) disassemble main
Dump of assembler code for function main:
   0x0000000000400536 <+0>:    push   %rbp
   0x0000000000400537 <+1>:	mov    %rsp,%rbp
   0x000000000040053a <+4>:	sub    $0x10,%rsp
   0x000000000040053e <+8>:	callq  0x400570 <asm_main>
   0x0000000000400543 <+13>:	mov    %eax,-0x4(%rbp)
   0x0000000000400546 <+16>:	mov    -0x4(%rbp),%eax
   0x0000000000400549 <+19>:	mov    %eax,%esi
   0x000000000040054b <+21>:	mov    $0x400624,%edi
   0x0000000000400550 <+26>:	mov    $0x0,%eax
   0x0000000000400555 <+31>:	callq  0x400410 <printf@plt>
   0x000000000040055a <+36>:	mov    $0x0,%eax
   0x000000000040055f <+41>:	leaveq 
   0x0000000000400560 <+42>:	retq   
End of assembler dump.
(gdb) x/s 0x400624
0x400624:    "print %d-bytes\n"
(gdb) 
```

main함수의 어셈블리 코드도 볼 필요가 있습니다.

가장 먼저 하는 일이 rbp를 저장하는 것입니다. 그리고 스택에 16바이트 공간을 만듭니다. 그 말은 지역변수의 전체 크기가 16바이트라는 것입니다. asm_main함수의 반환값 eax를 지역변수에 저장하네요. ret 변수가 -0x4(rbp)에 위치한다는 것을 알았습니다. 지역변수에 대해서는 "태초의 프로그래밍 언어 어셈블리"강좌에서 자세하게 설명했습니다. 스택에 저장하는 단위 크기가 64비트로 커진 것 외에는 달라진게 없으므로 더 설명하지 않겠습니다.

printf를 호출하기 위해 edi에 0x4000624를 저장하네요. 문자열의 시작 주소일 것입니다. x/s 명령으로 문자열을 출력해보니 역시나 문자열이 맞습니다.

우리는 최적화 옵션을 지정하지 않았습니다. 컴파일러는 최적화를 하나도 하지않은 코드를 생성했습니다. 그러다보니 main함수가 스택을 16바이트나 쓰기도하고 eax를 ret에 저장했다가 ret를 다시 eax에 읽어오는 불필요한 코드도 있습니다. 이런 코드를 줄여주는게 바로 컴파일러의 최적화중 하나입니다.

다음은 컴파일러가 최적화를 하면 기계코드가 어떻게 달라지는지를 실험해보겠습니다.
