# 리눅스 커널에서 ACCESS_ONCE 이해하기 (volatile 키워드)

https://lwn.net/Articles/508991/

변수 한개를 멀티쓰레드에서 공유할때는 lock을 사용하지 않고 atomic 연산으로 처리하기도 합니다. 그런데 최근 리눅스 커널에서 atomic연산을 빼고 ACCESS_ONCE()로 대체하는 경우가 있었습니다. 뭔가 해서 조사해봤더니 위의 문서에 설명이 있었습니다.

사실 C언어를 기준으로 설명해놔서 말은 길고 장황한데 잘 이해가 안됩니다. 대강 volatile을 써서 변수를 루프안에서 꼬박꼬박 다시 읽어오도록 만든다는 것인데, 왜 volatile이 뭐길래 그런 효과가 나는 걸까요? volatile을 아는 분들은 사실 변수를 읽을 때마다 메모리에서 다시 읽어온다라고 외운 경우가 많을건데요 디스어셈블을 해서 어떤 효과가 있는건지를 보겠습니다.

C로 ACCESS_ONCE를 쓰는 경우와 안쓰는 경우를 만들어보겠습니다.

```
#include <stdio.h>
#include <stdlib.h>
 
struct data
{
    int a;
    int b;
    int c;
};
int count[100];
 
#define __ACCESS_ONCE(x) ({(volatile typeof(x) *)&(x); })
#define ACCESS_ONCE(x) (*__ACCESS_ONCE(x))
 
int try_access(struct data *ddd)
{
    int v;
 
    while (1) {
        v = ACCESS_ONCE(ddd->a);
        if ((v == 1) && (ddd->b == 1)) {
            break;
        }
    }
}
 
int try_read(struct data *ddd)
{
    int v;
 
    while (1) {
        v = ddd->a;
        if ((v == 1) && (ddd->b == 1)) {
            break;
        }
    }
}
 
int try_reg(struct data *ddd)
{
    int v;
 
    while (1) {
        asm volatile ("" : : : "memory");
        v = ddd->a;
        if ((v == 1) && (ddd->b == 1)) {
            break;
        }
    }
    return v;
}
 
int main(void)
{
    int i;
    struct data testdata = {1, 2, 3};
 
    testdata.a = (random() % 6) + ((random() % 6) * 6);
    try_access(&testdata);
    try_read(&testdata);
    try_reg(&testdata);
 
    for (i = 0; i < 10; i++)
        printf("%d=>%d\n", i, count[i]);
    return 0;
}
```

소스가 긴데 봐야할 부분은 try_access()와 try_read()에서 v = ddd->a 입니다. 메모리의 특정 위치를 읽어서 그 값이 1인가 비교하는 것입니다. ddd 구조체는 여러 쓰레드에서 공유되는 데이타입니다. 즉 한 쓰레드에서 값이 ddd->a 값이 1인가 확인하면서 루프를 돌고, 다른 쓰레드는 어떤 일을 완료한 후 완료했다는 표시로 ddd->a에 1을 씁니다. spinlock을 가상으로 만든 것입니다.

gcc -S -O2 명령으로 빌드하면 디스어셈블 소스를 만들어줍니다.
```
~/assembly_64bit $ gcc -O2 -S access_once.c 
~/assembly_64bit $ cat access_once.s
```

디스어셈블된 소스가 긴데 우리가 볼 부분은 각 함수의 루프부분입니다.

```
try_access:
.LFB38:
    .cfi_startproc
    movl    4(%rdi), %edx
    .p2align 4,,10
    .p2align 3
.L2:
    movl    (%rdi), %eax
    cmpl    $1, %eax
    jne .L2
    cmpl    $1, %edx
    jne .L2
    rep ret
    .cfi_endproc
 
try_read:
.LFB39:
    .cfi_startproc
    movl    (%rdi), %eax
    movl    4(%rdi), %edx
    .p2align 4,,10
    .p2align 3
.L8:
    cmpl    $1, %eax
    jne .L8
.L12:
    cmpl    $1, %edx
    jne .L12
    rep ret
    .cfi_endproc
```

try_access에서 .L2 루프를 보면 메모리 주소 rdi에서 값을 읽어서 eax에 저장하고 1과 비교합니다. 즉 다른 쓰레드에서 메모리에 값을 쓰면, 현재 쓰레드에서도 값이 바뀐 것을 알 수 있습니다.

try_read의 .L8 루프를 보면 어떤가요?

try_reg는 메모리 베리어를 사용한 예입니다. ACCESS_ONCE()같이 volatile을 쓴 것과 메모리 베리어를 쓴 것이 어떤 차이가 있는지 확인해보세요. 사실 소스가 너무 간단해서 차이가 안보일 수도 있습니다만, 소스가 복잡해지면 점점 차이가 생깁니다. try_reg 함수의 베리어 앞뒤로 여러가지 계산식을 추가해보면 메모리 베리어가 무엇인지 실체를 보실 수 있습니다.

컴파일러가 이렇게 한다느니 저렇게 한다느니 알아야할게 많습니다. 메모리 베리어가 그런 예입니다. 많은 분들이 그냥 그렇다고 알고 외우고 넘어갑니다. 하지만 그 실체는 결국 기계어를 생성할 때 어떻게 만들것이냐입니다. 디스어셈블해서 이해할 수 있다면 그냥 알아야할게 아니라 눈으로 볼 수 있는 것입니다.

