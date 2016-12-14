## gcc 최적화 옵션 실험

이제 컴파일러의 최적화에 관한 실험을 몇가지 해보겠습니다. 컴파일러의 최적화 기술은 커널이나 데이터베이스만큼 방대하므로 우리는 바닷가 모래에서 한삽 정도 떠보겠습니다.



이 예제를 한번 빌드해보시기 바랍니다. 소스를 저장하고 gcc -c 옵션으로 빌드하면 됩니다. -O2 옵션을 추가했을 때의 .o 파일과 없을 때의 .o파일을 만들어서 objdump -S 명령으로 비교해도 됩니다. 아니면 main함수를 만들어서 gdb로 디스어셈블해도 됩니다. 편하신대로 하세요.
```
void delay(uint64_t count)
{
	for (;count > 0; count--)
		continue;
}
```
차이는 한눈에 알아볼 수 있습니다.

O2 최적화
```
0000000000000000 <delay>:
repz retq 
nopl   0x0(%rax)
nopw   %cs:0x0(%rax,%rax,1)
```

최적화 없음
```
0000000000000000 <delay>:
push   %rbp
mov    %rsp,%rbp
mov    %rdi,-0x8(%rbp)
jmp    f <delay+0xf>
subq   $0x1,-0x8(%rbp)
cmpq   $0x0,-0x8(%rbp)
jne    a <delay+0xa>
nop
pop    %rbp
retq 
```
  
최적화를 하면 루프가 날아가버립니다. 컴파일러 입장에서는 왜 사용하지도 않는 변수의 값을 바꾸는지 이해를 못한 것입니다. 컴파일러의 입장을 이해하지못하고 최적화를 시도했다가는 이런 문제가 발생하게됩니다.

만약 delay()함수를 쓰는 코드를 만들었다가 최적화 옵션을 추가했더니 프로그램 실행 시간이 줄었다고 좋아했다면, delay()가 필요한 곳에 문제가 생긴걸 몰라서일 것입니다. 강좌의 첫 머리에서 이야기했듯이 알고리즘의 최적화가 가장 먼저요, 그 다음에 컴파일러 최적화를 잘 활용하는 것입니다. 이 예제는 컴파일러의 최적화를 이해못한 것입니다.

그렇다고 어설프게 volatile int i로 변수를 만들어서 강제로 딜레이를 만들면 어떻게 될까요? 싱글 스레드 프로그램이라면 상관없겠지만 멀티스레드 프로그램이라면 딜레이함수가 전체 쓰레드의 성능에 영향을 줄 수 있습니다. volatile때문에 메모리 접근이 크게 늘어나면서 다른 쓰레드의 메모리 접근을 방해하게 되니까요. 한 쓰레드가 메모리 버스를 독점해버리는 것입니다. 하나의 쓰레드만 지연시키려다가 전체 프로그램이 지연됩니다. 리눅스 커널은 그래서 이런 루프가 아니라 디바이스 포트를 읽는 등의 방식을 사용합니다.



다음은 메모리 복사에 대한 실험을 해보겠습니다. 이것도 최적화 옵션을 줬을 때와 안줬을 때 각각의 코드를 디스어셈블 해보시기 바랍니다.
```
struct ll {
    uint64_t pad[8];
};

struct ll_big {
	uint64_t pad[32];
};

void copy_struct_ll(struct ll *dst, struct ll *src)
{
	*dst++ = *src++;
}

void copy_struct_ll_big(struct ll_big *dst, struct ll_big *src)
{
	*dst++ = *src++;
}
```

copy_struct_ll은 8*8 바이트를 복사합니다. copy_struct_ll_big은 8*32 바이트를 복사합니다. 이런 크기 차이가 컴파일에도 영향을 줄까요?

우선 좀더 깔끔하고 이해하기 쉬운 최적화 옵션이 있을때의 결과를 보겠습니다.
```
0000000000000010 <copy_struct_ll>:
  10:	48 8b 06             	mov    (%rsi),%rax
  13:	48 89 07             	mov    %rax,(%rdi)
  16:	48 8b 46 08          	mov    0x8(%rsi),%rax
  1a:	48 89 47 08          	mov    %rax,0x8(%rdi)
  1e:	48 8b 46 10          	mov    0x10(%rsi),%rax
  22:	48 89 47 10          	mov    %rax,0x10(%rdi)
  26:	48 8b 46 18          	mov    0x18(%rsi),%rax
  2a:	48 89 47 18          	mov    %rax,0x18(%rdi)
  2e:	48 8b 46 20          	mov    0x20(%rsi),%rax
  32:	48 89 47 20          	mov    %rax,0x20(%rdi)
  36:	48 8b 46 28          	mov    0x28(%rsi),%rax
  3a:	48 89 47 28          	mov    %rax,0x28(%rdi)
  3e:	48 8b 46 30          	mov    0x30(%rsi),%rax
  42:	48 89 47 30          	mov    %rax,0x30(%rdi)
  46:	48 8b 46 38          	mov    0x38(%rsi),%rax
  4a:	48 89 47 38          	mov    %rax,0x38(%rdi)
  4e:	c3                   	retq   
  4f:	90                   	nop

0000000000000050 <copy_struct_ll_big>:
  50:	b9 20 00 00 00       	mov    $0x20,%ecx
  55:	f3 48 a5             	rep movsq %ds:(%rsi),%es:(%rdi)
  58:	c3                   	retq   
```

두 함수가 완전히 다릅니다. 각각 dst가 rdi에, src가 rsi에 저장된 것은 동일합니다만 복사하는 명령어를 완전히 다른걸 사용합니다. copy_struct_ll은 src에서 rax로 8바이트를 읽고 읽은 8바이트를 dst에 복사합니다. mov 명령을 씁니다. 그런데 copy_struct_ll_big은 rep movs라는 연속된 메모리를 복사하는 명령을 씁니다. 왜 작은 크기 복사에서는 mov를 쓰고 큰 크기 복사에는 rep movs를 쓸까요?



이번에는 최적화안된 디스어셈블 코드를 보겠습니다.
```
0000000000000019 <copy_struct_ll>:
  19:	55                   	push   %rbp
  1a:	48 89 e5             	mov    %rsp,%rbp
  1d:	48 89 7d f8          	mov    %rdi,-0x8(%rbp)
  21:	48 89 75 f0          	mov    %rsi,-0x10(%rbp)
  25:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
  29:	48 8d 50 40          	lea    0x40(%rax),%rdx
  2d:	48 89 55 f8          	mov    %rdx,-0x8(%rbp)
  31:	48 8b 55 f0          	mov    -0x10(%rbp),%rdx
  35:	48 8d 4a 40          	lea    0x40(%rdx),%rcx
  39:	48 89 4d f0          	mov    %rcx,-0x10(%rbp)
  3d:	48 8b 0a             	mov    (%rdx),%rcx
  40:	48 89 08             	mov    %rcx,(%rax)
  43:	48 8b 4a 08          	mov    0x8(%rdx),%rcx
  47:	48 89 48 08          	mov    %rcx,0x8(%rax)
  4b:	48 8b 4a 10          	mov    0x10(%rdx),%rcx
  4f:	48 89 48 10          	mov    %rcx,0x10(%rax)
  53:	48 8b 4a 18          	mov    0x18(%rdx),%rcx
  57:	48 89 48 18          	mov    %rcx,0x18(%rax)
  5b:	48 8b 4a 20          	mov    0x20(%rdx),%rcx
  5f:	48 89 48 20          	mov    %rcx,0x20(%rax)
  63:	48 8b 4a 28          	mov    0x28(%rdx),%rcx
  67:	48 89 48 28          	mov    %rcx,0x28(%rax)
  6b:	48 8b 4a 30          	mov    0x30(%rdx),%rcx
  6f:	48 89 48 30          	mov    %rcx,0x30(%rax)
  73:	48 8b 52 38          	mov    0x38(%rdx),%rdx
  77:	48 89 50 38          	mov    %rdx,0x38(%rax)
  7b:	90                   	nop
  7c:	5d                   	pop    %rbp
  7d:	c3                   	retq   

000000000000007e <copy_struct_ll_big>:
  7e:	55                   	push   %rbp
  7f:	48 89 e5             	mov    %rsp,%rbp
  82:	48 89 7d f8          	mov    %rdi,-0x8(%rbp)
  86:	48 89 75 f0          	mov    %rsi,-0x10(%rbp)
  8a:	48 8b 55 f8          	mov    -0x8(%rbp),%rdx
  8e:	48 8d 82 00 01 00 00 	lea    0x100(%rdx),%rax
  95:	48 89 45 f8          	mov    %rax,-0x8(%rbp)
  99:	48 8b 45 f0          	mov    -0x10(%rbp),%rax
  9d:	48 8d 88 00 01 00 00 	lea    0x100(%rax),%rcx
  a4:	48 89 4d f0          	mov    %rcx,-0x10(%rbp)
  a8:	48 89 c6             	mov    %rax,%rsi
  ab:	b8 20 00 00 00       	mov    $0x20,%eax
  b0:	48 89 d7             	mov    %rdx,%rdi
  b3:	48 89 c1             	mov    %rax,%rcx
  b6:	f3 48 a5             	rep movsq %ds:(%rsi),%es:(%rdi)
  b9:	90                   	nop
  ba:	5d                   	pop    %rbp
  bb:	c3                   	retq   
```

스택에 지역변수를 만들기 위해 rbp 값을 저장하고, 함수 인자들을 저장하는 등 함수가 해야 할 일들이 들어가있습니다. 사실 필요한 코드지만 우리가 만든 예제 코드가 워낙 간단한 코드이기 때문에 지역변수를 만들거나 rbp를 저장할 필요가 없으니 그런 코드들을 제거하는 것도 최적화하는 방법일 것입니다.

그리고 메모리 복사에 mov를 쓰고 rep movs를 쓰는 차이는 없어지지 않습니다. 그런걸 봐서 최적화의 차이가 아니라 복사하는 메모리 크기의 차이인 것이 확실해졌습니다. rep movs가 뭔지 왜 좀더 큰 메모리를 복사하는데 쓰는 것인지, 뭔가 더 큰 메모리 복사에 적당하니까 쓰는 것이겠지만 왜 그런지 등은 다음 장에서 리눅스 커널에서 메모리 복사를 어떻게 구현했는지를 보면서 생각해보겠습니다.

rep movs 로 구글링을 해보세요. 이미 우리가 가진 의문들에 대해서 먼저 의문을 가진 사람들이 있습니다.