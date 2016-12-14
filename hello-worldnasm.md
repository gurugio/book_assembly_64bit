#nasm 어셈블러로 Hello, world! 출력하기

##nasm 설치

"태초의 프로그래밍 언어 어셈블리"강좌에서는 emu8086이라는 툴이 있어서 정말 쉽게 실습해볼 수 있었습니다. 감사한 일이지요. 개발회사가 사라져서 얼마나 아쉬운지 모릅니다. 언젠가 제가 유사한 툴을 오픈소스로 만들 기회가 있기를 바랍니다.

64비트 환경에서는 에물레이터가 필요없습니다. 그냥 64비트 컴퓨터에서 그대로 실행하면 되지요. 혹시 지금 32비트 윈도우에서 이 강좌를 보고계시다면 64비트 리눅스 환경을 꾸미시기 바랍니다. 64비트 리눅스를 기준으로 설명하겠습니다. 윈도우에서 MASM으로 개발해본게 너무 오래전이라 지금 되새김질하기에는 시간이 아까워서요. VitrualBox같은 가상화 툴을 사용하시면 64비트 리눅스를 설치하실 수 있으실 겁니다. 리눅스 배포판 중에서도 우분투를 기준으로 설명하겠습니다. 어떤 배포판을 골라야할지 모르시겠으면 그냥 우분투 설치하세요. 무조건 골수 오픈소스 개발자가 되고싶으시면 데비안이구요. 설치 방법은 많은 자료가 있으니 설치하신걸로 생각하고 진행하겠습니다.

그럼 언제나 좋은 친구 오픈소스 진영에 어떤 어셈블러가 있는 확인해보겠습니다. 그냥 간단하게 다음 명령을 실행해보시면 됩니다.
```
$ apt-cache search assembler | grep assembler

gurugio@giohnote:~$ apt-cache search assembler | grep assembler
bin86 - 16-bit x86 assembler and loader
binutils - GNU assembler, linker and binary utilities
binutils-doc - Documentation for the GNU assembler, linker and binary utilities
binutils-source - GNU assembler, linker and binary utilities (source)
libasm1 - library with a programmable assembler interface
nasm - General-purpose x86 assembler
yasm - modular assembler with multiple syntaxes support
abyss - de novo, parallel, sequence assembler for short reads
a56 - Motorola DSP56001 assembler
as31 - Intel 8031/8051 assembler
avra - assembler for Atmel AVR microcontrollers
binutils-arm-none-eabi - GNU assembler, linker and binary utilities for ARM Cortex-A/R/M processors
binutils-hppa64-linux-gnu - GNU assembler, linker and binary utilities targeted for hppa64-linux
crasm - Cross assembler for 6800/6801/6803/6502/65C02/Z80
d52 - Disassembler for 8052, 8048/8041, and Z80/8080/8085 code
dis51 - Disassembler for 8051 code in Intel Hex format
fasm - fast assembler for the x86 and x86-64 architectures
fcml - single-line assembler and disassembler
flasm - assembler and disassembler for Flash (SWF) bytecode
gnusim8085 - Graphical Intel 8085 simulator, assembler and debugger
idba - iterative De Bruijn Graph De Novo short read assembler for transcriptome
jasmin-sable - Java class (.class) file assembler
libdisasm-dev - disassembler library for x86 code (development files)
libdisasm0 - disassembler library for x86 code
libdistorm3-3 - powerful disassembler library for x86/AMD64 binary streams
libdistorm3-dev - powerful disassembler library for x86/AMD64 binary streams (development files)
libdistorm64-1 - ultimate disassembler library for x86 code
libdistorm64-dev - ultimate disassembler library for x86 code - header files
libhsdis0-fcml - HotSpot disassembler plugin using FCML
libjnr-x86asm-java - Pure java x86 and x86_64 assembler
minia - short-read biological sequence assembler
mira-assembler - Whole Genome Shotgun and EST Sequence Assembler
mira-doc - documentation for the mira assembler
mira-examples - files to experiment with the mira assembler
palbart - Enhanced version of the PAL PDP8 assembler
pasmo - easy to use Z80 cross-assembler
python-distorm3 - powerful disassembler library for x86/AMD64 binary streams (Python bindings)
ray-doc - focumentation for ray parallel de novo genome assembler
ssake-examples - example data for SSAKE, a genomic assembler of short reads
velvet - Nucleic acid sequence assembler for very short reads
velvet-example - Example data for the Velvet sequence assembler
velvet-long - Nucleic acid sequence assembler for very short reads, long version
velvet-tests - Test data for the Velvet sequence assembler
xa65 - cross-assembler and utility suite for 65xx/65816 processors
z80asm - assembler for the Zilog Z80 microprocessor
z80dasm - disassembler for the Zilog Z80 microprocessor
z88dk - Z80 processor assembler and SmallC+ cross compiler
```

놀랍지 않으세요? 8051, 6800, Z80 등 옛날 호랑이 담배피던 시절에 우리 조상님들께서 일하시던 유물들의 어셈블러도 있습니다. 오픈소스 개발자들은 쉬지도 않고 누가쓸지 모르겠을 모든 것을 다 개발하고 있습니다. 순전히 재미있으니까 하는 것이지 (물론 돈도 같이 벌리면 더 좋지요) 돈만 바로보고는 이렇게 못할거라는 생각이 듭니다.

그중에서 우리가쓰는 x86을 지원하는 어셈블러는 nasm과 fasm이 있는데 제가 좀더 익숙한 nasm을 쓰겠습니다. 둘다 유닉스/윈도우즈를 지원하고 32비트/64비트를 지원하므로 약간의 지시어만 다를뿐 대동소이할 것으로 생각됩니다.

설치는 간단합니다.
```
$ sudo apt-get install nasm
```
아래와 같이 nasm이 실행되면 설치가 완료된 것입니다.
```
gurugio@giohnote:~$ nasm
nasm: error: no input file specified
type `nasm -h' for help
```

nasm -h 명령도 실행해 보세요. 앞으로 쓸 옵션들과 설명들이 나옵니다.

##nasm으로 Hello, world! 출력하기

이쯤되면 무조건 해야되는게 있지요. Hello, world!를 출력해보겠습니다.

두가지 방법이 있습니다. 순전히 어셈블리로만 실행파일을 만들 수도 있고, 특정 함수만 어셈블리로 만들고, C 코드에서 어셈블리로 구현된 함수를 호출할 수도 있습니다.

먼저 해볼것은 함수 하나만 어셈블리로 만들어보는 것입니다. 다음 파일을 hello.nasm으로 저장합니다.

```
; To create executable:
; nasm -f elf64 hello.asm
; gcc -o greet hello.o main.c
 
segment .data
 
outmsg db    "Hello, world!", 0
 
segment .bss
 
segment .text
    global  asm_main        ; other modules can call asm_main
    extern printf
 
asm_main:
    push rdi
 
    mov rdi, outmsg
    call printf
     
    pop rdi
    ret
```
다음 파일은 main.c 입니다.

```
void asm_main(void);
 
int main(void)
{
    asm_main();
    return 0;
}
```
일단 지금은 빌드해서 돌아가는 것만 보겠습니다. 자세한 분석은 gdb로 한줄한줄 실행해가면서 알아보겠습니다.
```
$ nasm -f elf64 hello.nasm
$ gcc -o greet hello.o main.c
$ ls
greet  hello.nasm  hello.o  main.c  README.md
$ ./greet
Hello, world!$ 
```
nasm으로 빌드할 때 반드시 지정해야하는 옵션이 elf64입니다. 우리는 현재 리눅스에서 실행될 프로그램을 빌드하고 있으므로 elf포맷으로 빌드해야하고 64비트이므로 elf64로 지정합니다. 기타 옵션들은 nasm 홈페이지에 있는 메뉴얼을 참고하시면 됩니다.

 

다음으로는 어셈블리 코드만으로 실행파일을 만들어보겠습니다. 이 소스는 다음 블로그에 있는 소스입니다.

https://thebrownnotebook.wordpress.com/2009/10/27/native-64-bit-hello-world-with-nasm-on-freebsd/

```
section .data
 
message:
    db      'hello, world!', 0
 
section .text
 
global _start
_start:
    mov     rax, 4
    mov     rdi, 1
    mov     rsi, message
    mov     rdx, 13
    syscall
 
    mov     rax, 1
    xor     rdi, rdi
    syscall
```
빌드 방법도 블로그에 있습니다.

```
~$ nasm -f elf64 -o p.o p.nasm
~$ ld -o p p.o
~$ ./p
```

마찬가지로 elf64 포맷으로 object파일을 만들고 링커로 실행파일을 만듭니다.

 

2가지 방식으로 실행파일을 만들었는데 무슨 차이일까요?

제가 보여드리고 싶은 것은

1. 64비트 레지스터 사용 방법
1. 64비트 환경에서 함수를 호출하는 방법
1. 사실 main함수가 프로그램의 진짜 시작 지점이 아니라는 것
1. ./p 프로그램을 실행하면 hello, world!는 출력되지만 세그먼트 폴트로 발생한다는 것
1. 사실 위 4가지 내용이 이 강좌의 전부라는 것입니다.

그럼 어셈블러 사용법을 알았고 소스를 빌드해봤으니 emu8086의 64비트 버전이라고 할 수 있는 gdb를 가지고 실험을 해보겠습니다.

 