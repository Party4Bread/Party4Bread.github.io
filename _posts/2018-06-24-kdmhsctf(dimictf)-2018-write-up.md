---
layout: post
title: "KDMHSCTF(dimictf) 2018 Write UP"
date: 2018-06-24 17:00:07
image: '/assets/img/nani.png'
description: dimictf Write UP
category: 'Write UP'
tags:
- CTF
- Reversing
- Guessing
twitter_text:
introduction: dimictf 2018 8th Write UP
---
# KDMHSCTF(dimictf) 2018 W-P

`5시간 참여 / 고등부 8위 / 5620P / Played as two steps from haven `


##PWN

`??? : 말도안돼 P4B가 포너블을 풀다니`

###init(580p)

바이너리도 안보고 풀었다.

NC 서버에 접속한뒤 문제 메뉴를 테스트 해보려고 쳐본 명령어에서 플래그가 나왔다.

```
breadparty@ubuntu ~ nc 121.170.91.17 9901
Do you want to do?
[R]ead
[W]rite
[E]xit
>>> W
length: 70
hA0k�����ꛋ�Udimi{A110cAt3_1s_$o_1mp0rt@n7}�@2��Do you want to do?
[R]ead
[W]rite
[E]xit
>>> None !
Do you want to do?
[R]ead
[W]rite
[E]xit
>>> 
```

![엩....](/assets/img/seemslegit.png)

그렇구나 ㅎ....

### BabyCpp(880p)

이번에도 바이너리는 안봤다.... ~~이쯤되면 프로 guesser~~

메뉴보고 대충 Use After Free겠지... 한것이 맞은것이다....

```
breadparty@ubuntu ~ nc 121.170.91.17 7777
There are two very vulnerable objects
And all type can be declared ( character, string, integer, etc.. )
You just do new, delete and call
[I]nteger
[D]ouble
[C]haracter
[V]ulnteer
[S]hell
[E]xit
>>> V
[N]ew
[C]all
[D]elete
>>> N
[I]nteger
[D]ouble
[C]haracter
[V]ulnteer
[S]hell
[E]xit
>>> V
[N]ew
[C]all
[D]elete
>>> D
[I]nteger
[D]ouble
[C]haracter
[V]ulnteer
[S]hell
[E]xit
>>> S
[N]ew
>>> N
Shell address: 0x560f0cf5b998
[I]nteger
[D]ouble
[C]haracter
[V]ulnteer
[S]hell
[E]xit
>>> V
[N]ew
[C]all
[D]elete
>>> C
id
uid=1000(babycpp) gid=1000(babycpp) groups=1000(babycpp)
cat /home/babycpp/flag
dimi{Cpp_1s_e@5y_and_d1ff1cult}
```

그냥 포지션을 Reverser, Crypto에서 다시 Guesser로 바꿔야하나 생각이들었다.

----------

## REV

### EZPZ(500p)

1번문제여서 strings를 먼저 시도해 보았다.

```
breadparty@ubuntu ~/Downloads strings -e l EZPZ.exe|grep _  
Welcome_reversing
VS_VERSION_INFO
```

`Welcome_reversing`이 수상해서 `dimi{Welcome_reversing}` 인증을 시도했더니 풀렸다.

## mm(930p)

헥스레이가 죽어서 애먹기 시작했다... 그래서 그냥 어셈으로 풀기로 했다,

바이너리가 고정된 seed인 6051로 rand를 하면서 입력한 값과  어떤 연산을 해서 배열에 넣고 일괄적으로 어떤 테이블과 비교하고있었다.

그래서 python을 1바이트씩 브포하는 스크립트를 만들어서 돌렸다.

```python
from ctypes import CDLL

libc=CDLL('libc.so.6')

libc.srand(6051)

randtable=[]
origin=[0x73A8, 0x39CC, 0x4E0A, 0x8D85, 
0xD1F2, 0x7776, 0x272E, 0xAB31, 
0x8F34, 0x4659, 0xE7AC, 0xA308,
0x154D, 0x7D9F, 0x7123, 0xF8DB,
0x49C4, 0x5BB8, 0x2274, 0xDD76,
0xC29D, 0x7048, 0x52AE, 0x1361,
0xC98C, 0x73A6, 0x870A, 0x8870,
0x748D, 0x0669, 0x8C8F, 0xE8A9,
0x40B1, 0xDABF, 0x76C7, 0x133D, 
0x52B2, 0x9E59, 0xBE76, 0xE248,
0xE4DD, 0xA6C5, 0x856E, 0xFAB7, 
0x2465, 0xF6F7, 0xF41C, 0x6E93, 
0x535A, 0x16DA, 0x4C54, 0x166D, 
0x87A4, 0x9F0F, 0x29DD, 0x51A3, 
0x1327, 0xB13A]

import string
for i in origin:
    tmp=(libc.rand()&0xFFFF)
    randtable.append(tmp)
    for j in string.printable:
        iy=ord(j)
        iy*=tmp
        it=tmp+1
        if i==iy%it:
            print(j,end="")
            break
print()
```

결과는 다음과같다

```
breadparty@ubuntu ~/dimictf python3 mmhand.py
dimi{ca1cul4t3d_inv3rs3?_0r_us3d_z3?_0h_y0u_ar3_4_F0Ol_;)}
```

다른 더 쉬운 방법이 있던걸까.... ㅡㅡ

###Challenger(980p)

디스어셈블리도 제대로 되질않아서 그냥 입력 값을 넣는 메모리주소에 모두 하드웨어 브레이크 포인트를 rwx로 걸고 천천히 디버깅 해나가니 약간씩 소스코드가 보이기 시작했다.

고정된 0 seed를 사용하는 rand 값과 입력값을 xor해나가면서 비교하는것을 보고 빠르게 파이썬 소스코드를 만들었다.

```python
from ctypes import cdll

libc = cdll.msvcrt

libc.srand(0)
sometable=[0x5e,0x1F,0xC0,0xB1,
 0xAD,0x5D,0xC8,0x6F,
 0xB7,0xCB,0x9E,0xAB,
 0x1B,0x7C,0x4A,0x6E,
 0xB0,0xF2,0xFB,0x1D,
 0xA7,0xCC,0xD4,0x5A,
 0x78,0x47,0xBD,0xA5,
 0x93,0x71,0xEB,0x3C]
print("".join([(chr(i^(libc.rand()&0xFF))) for i in sometable]))
```

결과는 다음과같다.

```
C:\Users\BREADY\py3_5_deep\Scripts\python.exe D:/AngstromCTF/CHallengerhandray.py
x864:Here_Comes_A_NEW_Challenger
```

플래그 형식에 맞춰서 인증하니 됬다.

힌트로 나온 링크는.... 봤긴했지만...

![전혀 모르겠다](/assets/img/idkrlry.png)

## MISC

### MIC CHECK(10p)

어...음?

`dimi{Hello, DIMIGO!} `

### Win RSP(830p)

안드로이드 앱을 주기에 nox에 넣고 게임가디언으로 값을 980413으로 조작해서 ~~더러운 툴키디~~ 1번만 더 이기고 플래그를 얻었습니다.

`flag{Are_you_Genius_or_Stupid?}`

### guess(910p)

flag.png 라는 파일가 있는데..... 당신 누구야?

~~**???:ida에서 열었는데 소스가안보여**~~ 

![와 고건 몰랐네....](/assets/img/wellidkthat.png)

아무튼 메모장으로 여니까 `8Xk4HCc` 라는게 나와서 준것도 이미지파일이고해서 이미지 공유 사이트가면 있지않을까해서 imgur에 넣어보니...

![아니 이게 플래그라고?](https://i.imgur.com/8Xk4HCc.png)

.....

-----------

## 후기

토요일 대회면 참 좋았을텐뎅......

그리고 진짜 제대로된 포너블 풀어보고싶다....

그리고 H3X0R떄 끝나고 8분뒤에 하나 풀려서 설마 또 그러진 겠지 했지만

아니나 다를까 dimictf에서도 끝나고 6분위에 what을 풀어서.....

RSA글도 끝내야하는데....