#리눅스 커널 디버깅

https://codeascraft.com/2012/03/30/kernel-debugging-101/

이 포스팅을 약간만 설명하겠습니다. 그냥 눈으로만 읽어보세요. 리눅스 커널 디버깅이 이렇게 간단하게 해결되는 일은 거의 없습니다. 엄청난 테스트를 하고 릴리즈가 되니까요. 어셈블리를 알아야 이렇게 디버깅을 할 수 있다는걸 보여드리기위한 예제로 소개하는 것이니 보기만 하셔도 좋습니다.

어느날 리눅스상에서 게임을 하던 중 커널에서 이런 메세지를 출력하면서 시스템이 죽어버렸습니다.
```
Pid: 0, comm: swapper Not tainted <kernel> #1 Supermicro X8DTT-H/X8DTT-H
RIP: 0010:[<ffffffff8142bb40>]  [<ffffffff8142bb40>] __netif_receive_skb+0x60/0x6e0
RSP: 0018:ffff88034ac83dc0  EFLAGS: 00010246
RAX: 0000000000000060 RBX: ffff8805353896c0 RCX: 0000000000000000
RDX: ffff88053e8c3380 RSI: 0000000000000286 RDI: ffff8805353896c0
RBP: ffff88034ac83e10 R08: 00000000000000c3 R09: 0000000000000000
R10: 0000000000000001 R11: 0000000000000000 R12: 0000000000000000
R13: 0000000000000015 R14: ffff88034ac93770 R15: ffff88034ac93784
FS:  0000000000000000(0000) GS:ffff88034ac80000(0000) knlGS:0000000000000000
CS:  0010 DS: 0018 ES: 0018 CR0: 000000008005003b
CR2: 0000000000000060 CR3: 000000010e130000 CR4: 00000000000006e0
DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
DR3: 0000000000000000 DR6: 00000000ffff0ff0 DR7: 0000000000000400
Process swapper (pid: 0, threadinfo ffff880637d18000,
task ffff880337eb9580)
Stack:
 ffffc90013e37000 ffff880334bdc868 ffff88034ac83df0 0000000000000000
<0> ffff880334bdc868 ffff88034ac93788 ffff88034ac93700 0000000000000015
<0> ffff88034ac93770 ffff88034ac93784 ffff88034ac83e60 ffffffff8142c25a
Call Trace:
 <IRQ>
 [<ffffffff8142c25a>] process_backlog+0x9a/0x100
 [<ffffffff814308d3>] net_rx_action+0x103/0x2f0
 [<ffffffff81072001>] __do_softirq+0xc1/0x1d0
 [<ffffffff810d94a0>] ? handle_IRQ_event+0x60/0x170
 [<ffffffff8100c24c>] call_softirq+0x1c/0x30
 [<ffffffff8100de85>] do_softirq+0x65/0xa0
 [<ffffffff81071de5>] irq_exit+0x85/0x90
 [<ffffffff814f4dc5>] do_IRQ+0x75/0xf0
 [<ffffffff8100ba53>] ret_from_intr+0x0/0x11
 <EOI>
 [<ffffffff812c4b0e>] ? intel_idle+0xde/0x170
 [<ffffffff812c4af1>] ? intel_idle+0xc1/0x170
 [<ffffffff813fa027>] cpuidle_idle_call+0xa7/0x140
 [<ffffffff81009e06>] cpu_idle+0xb6/0x110
 [<ffffffff814e5ffc>] start_secondary+0x202/0x245
Code: 00 44 8b 1d cb be 79 00 45 85 db 0f 85 61 06 00 00 f6 83 b9
      00 00 00 10 0f 85 5d 04 00 00 4c 8b 63 20 4c 89 65 c8 49 8d
      44 24 60 <49> 39 44 24 60 74 44 4d 8b ac 24 00 04 00 00 4d
      85 ed 74 37 49
RIP  [<ffffffff8142bb40>] __netif_receive_skb+0x60/0x6e0
 RSP <ffff88034ac83dc0>
CR2: 0000000000000060
```

이럴때 뭘 해야할까요?

일단 어떤 명령을 실행하다가 죽었는지 RIP의 값을 봅니다.
```
__netif_receive_skb+0x60
```

라네요.

gdb에서 disassemble __netif_receive_skb를 하셔서 어떤 명령어인지 찾아보면 됩니다. 어떤 명령어일지는 커널 버전, 빌드 옵션 등등 실제로 부팅된 커널이 뭐냐에 따라 다르겠지요. 어쨌든 저자가 찾아보니 이런 명령어였답니다.
```
0xffffffff8142bb40 <__netif_receive_skb+96>:    cmp    %rax,0x60(%r12)
```

보아하니 rax에 저장된 값과 r12+0x60 위치에 저장된 값을 비교하는 명령입니다. 그럼 이 명령이 실행되는 순간에 레지스터 값들이 뭐였는지를 보면 되겠네요. 위의 커널 패닉 메세지에 나온 레지스터 값들이 바로 이 명령이 실행되는 순간의 레지스터 값들입니다. 
```
RAX: 0000000000000060 RBX: ffff8805353896c0 RCX: 0000000000000000
RDX: ffff88053e8c3380 RSI: 0000000000000286 RDI: ffff8805353896c0
RBP: ffff88034ac83e10 R08: 00000000000000c3 R09: 0000000000000000
R10: 0000000000000001 R11: 0000000000000000 R12: 0000000000000000
```
rax는 0x60인데 읽고있는 메모리 주소 값이 r12+0x60=0x60번지입니다. Canonical address space에서 말씀드렸듯이 커널 주소는 F로 시작해야합니다. 뭔가 주소가 잘못되었습니다. 포인터 에러입니다.

gdb나 crash같은 툴을 써서 __netif_receive_skb+0x60 위치가 C 코드로 어딘가 찾아봤더니 netpoll_receive_skb 함수의 리스트를 읽는 부분이었습니다.

이때부터는 순수하게 커널 코드를 알거나 해당 드라이버 코드를 알아야만 추적이 가능한 부분입니다. 왜 리스트를 읽다가 포인터 값이 잘못될 수 있는지를 알려면 이 리스트가 어디서 왜 사용되는 리스트인지, 어디서 잘못될 가능성이 있는지 등등 커널 지식을 동원해야합니다.

이렇게 리눅스 커널의 문제를 디버깅할 때 어셈블리를 알아야합니다. C로 대규모 프로젝트를 한다면 사실 어셈블리를 모르고서는 디버깅이 안되지요.



