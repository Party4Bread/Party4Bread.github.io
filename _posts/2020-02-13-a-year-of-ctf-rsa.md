---
date: '2020-02-14 01:09:13'
layout: post
title: A year of CTF RSA
subtitle: 1년정도 CTF 뉴비로 있으면서 (아직도 뉴비지만) 겪었던 RSA 문제들의 유형을 대략 정리했습니다.
description: 1년정도 CTF 뉴비로 있으면서 (아직도 뉴비지만) 겪었던 RSA 문제들의 유형을 대략 정리했습니다.
image: /assets/img/uploads/somrresult4.png
category: blog
tags:
  - CTF
  - RSA
  - Cryptography
author: party4bread
paginate: false
---
몇 년간 CTF를 했었을때 꽤 많은 RSA문제를 풀었던거같은데, 그 유형들을 한번 정리해보았습니다. ~~뻔하고 지루한 유형을 못나오게 해버리자구요. 아 글고 RSACTFTools 좀 그만 써보자구요. 출제자들도 힘빠질꺼아녜요.~~ 이런 문제들과 그 공격이 동작하는 방법을 공부하는 것은  매우 흥미롭습니다. 부디 이 글이 더 깊은 공부의 좋은 시발점이 되기를 바랍니다.

## Wrong Implementation of Generator

뭔가 생성기가 잘못된 경우입니다.

### p와 q가 상당히 인접할 경우

페르마의 인수 분해법을 사용하면됩니다.

페르마의 인수 분해법은 어떤 홀수 $N$을  $N=a^2-b^2$처럼 [정수 제곱수의 차로 표현](https://en.wikipedia.org/wiki/Difference_of_two_squares) 할 수 있는것에 기반하고 있는데 이때 이 식을 한번 더 펼쳐서 $N=(\frac {c+d}2)^2-(\frac {c-d}2)^2$로 만들면 $N =cd$임을 알 수 있습니다.

페르마의 인수 분해법은 다음과 같습니다.

1. $n$이 완전 제곱수이면 끝내고, 아니면 $a$를 $\sqrt n$보다 큰 가장 작은 수로 설정합니다.
2. $a^2-n$이 완전 제곱수이면 $n=a^2-b^2$ 인것이므로 끝내고 아니믄 $a$를 1 증가시키고 1로 돌아간다.

정말 간단하죠?

구현도 간단합니다.

```python
def factor(n):
	x = math.ceil(math.sqrt(n))
	y = x**2 - n
	while not math.sqrt(y).is_integer():
		x += 1
		y = x**2 - n
	return x + math.sqrt(y), x - math.sqrt(y)
```

### p와 q가 상당히 작은경우

그냥 인수 분해기 돌리세요.

[Pollard Rho]([https://en.wikipedia.org/wiki/Pollard%27s_rho_algorithm](https://en.wikipedia.org/wiki/Pollard's_rho_algorithm))돌려도 되기는 하는데 그냥 인수 분해기 돌리세요.

제가 추천할만한 인수 분해기는... 음....

[yafu](https://sourceforge.net/projects/yafu/) : GNFS,SNFS,페르마 인수분해법,SIQS,타원곡선,Pollard Rho 등등 지원

[CADE-NFS](http://cado-nfs.gforge.inria.fr/) : 엄청 빠르고 클러스터도 구축 가능하던거로 기억합니다. 다만 NFS만 됩니다. 제법 최근까지도 관리 되었습니다. yafu보다 좀 더 큰 수에서 유리합니다.

[msieve](https://sourceforge.net/projects/msieve/) : SIQS,GNFS의 구현체고 yafu한테 붙여서 쓸수도 있습니다. 근데 yafu가 보통 더 빠릅니다.

[GMP-ECM](https://gforge.inria.fr/projects/ecm/) : ECM과 후술할 p-1, p+1알고리즘이 구현 되어 있습니다.

[factordb](http://factordb.com/) : 수많은 소수의 db입니다. 제법 쓸만 합니다.

제가 안쓰지만 좋은 프로그램들은 메르센 포럼에 [좋은 스레드](https://www.mersenneforum.org/showthread.php?t=3255)에 있습니다.

대부분의 위 프로그램들은 상당히 오랫동안 관리가 된적이 없어서 직접 컴파일 하셔서 CPU최적화를 하시고 쓰는것을 추천하고 아니면 관련 포럼에서 CUDA같은 병렬화 지원을 찾아보는것을 추천드립니다.

소인수 분해는 무려 [포럼](https://www.mersenneforum.org/forumdisplay.php?f=19)도 있습니다. 한번 확인해 보시면 더 빠르게 인수 분해 하는 법도 종종 나오고는 합니다.

### p,q에 인접한 수가 smooth수 일 경우

[RBTree님의 글](http://www.secmem.org/blog/2019/10/20/Smooth-number-and-Factorization/
) 에 잘 나와있습니다.

요약을 하자면 $p-1$이나 $q-1$중 하나가 [B-powersmooth](https://en.wikipedia.org/wiki/Smooth_number#Powersmooth_numbers)면 Pollard’s p-1 알고리즘을 사용하면 효율적으로 인수 분해 하는것이 가능하고, $p+1$이나 $q+1$중 하나가 [smooth](https://en.wikipedia.org/wiki/Smooth_number)하면 Williams's p + 1 알고리즘으로 비교적 효율적으로 풀어낼 수 있습니다.

### pq에 0이 많을 경우

N값에 0이 많으면 합리적 의심으로 두 소수중 적어도 하나는 100000.....000+z 꼴일 가능성을 생각해 볼수있습니다. 이런 경우에 n진법이라 하면 각 자릿수를 하나씩 지워보면서 gcd를 하여 1이 아닌게 나오면 인수 분해가 됩니다. 2진수인 경우에도 해당 될수 있고 7진수, 16진수 구분없이 0이 많다면 각 자릿수를 뒤에서 부터 지워보면서 gcd를 하면 빠르게 p,q를 알아낼수도 있습니다.

### 왠지 d가 작을꺼 같은 경우

Wiener 공격이나 Boneh-Durfee 공격이 있는데 후자가 좀더 적용가능한 d가 넓어서 그냥 후자를 찔러 봅시다. Boneh-Durfee 공격은 [mimoo](https://github.com/mimoo/RSA-and-LLL-attacks)님의 구현체를 참고하는것도 좋습니다.

## Misuse of RSA

이곳은 거의다 Coppersmith 교수님의 방법이 장악했습니다.

간단한 insight는 [eyebrowmoon님의 글](https://eyebrowmoon.github.io/hacking/crypto/rsa/2019/05/23/RSA_Attack_Using_LLL.html)에서 얻을수 있습니다.

### e가 좀 작은데 같은 M이 다른 e개의 N으로 암호화 된 경우

Håstad's broadcast attack을 씁시다. 

### 손상된 p와 q가 주어진 경우

상위 비트를 충분히 안다면 [mimoo](https://github.com/mimoo/RSA-and-LLL-attacks)님의 구현을 참고해보세요.

무작위 bit가 주어진 p,q의 복구 에 관한건 [RBTree님의 글](http://www.secmem.org/blog/2019/11/15/On-Factoring-Given-Any-Bits/) 을 참조하면 됩니다.

### e=3인데 메세지가 2개 주어지고 둘의 선형 관계가 주어진 경우

Franklin-Reiter related-message attack를 하시면 됩니다.

### 같은 키를 여러번 사용하는 Oracle이 주어진 경우

C를 가지고 있다면 선택 암호문 공격을 할수 있습니다.

C의 복호화를 금하는 경우 m이 절묘하게 큰 경우가 아니라면 $2^eC  \pmod N$ 를 하여 $c'$을 만들고 Oracle에게 복호화를 시도하면 $2m \pmod N$을 얻을 수 있습니다.

### e=2n 일 경우

그것은 아마도 RSA가 아닙니다 Rabin Cryptosystem일 가능성이 매우 높습니다.

### 아무래도 m^e가 N과 큰 차이가 없을꺼 같은 경우

특히나 e 가 작은데 N이 과도하게 크다면 의심해 봅시다.

c+(N*i) 에서 i값을 적절히 증가 시키면서 e의 제곱근이 정수인지를 살펴봐 줍시다.

### 메세지의 MSB를 알 경우

Stereotyped Messages 공격을 해봅시다.

자세한것은 [mimoo](https://github.com/mimoo/RSA-and-LLL-attacks)님의 레포를 참고 합시다. 마찬가지로 Coppersmith method가 사용되므로 위에있는 글들을 읽어보면 큰 도움이 됩니다.

### PKCS#1 v1.5 가 뜬끔없이 나올경우

일단 Bleichenbacher’s Low-Exponent Attack이 성립될지 한번 고민해 봅시다.

### 같은 메세지가 다른 e값 같은 N값으로 암호화된 경우

흑마법을 부리면 됩니다. 확장 유클리드 알고리즘을 이용해서

$\gcd(e_B,e_C)=1$이고 $e_Bs_1+e_cs_2=1$인 $s1$과 $s2$를 구하고

$C^{s_1}_B∗C^{s^2}_C=(M^{e_B})^{s_1}∗(M^{e_C})^{s_2}=M^{e_Bs_1}∗M^{e_Cs_2}=M^{e_Bs_1+e_Cs_2}=M^1=M$

짜잔!

## 기타 공격

그 외에 것은 흔하지는 않지만 가끔 가끔 나오는 것들 입니다.

### p,q 가 PRP였지만 P는 아니다

소수 판별이 잘못 구현 되어있을경우 생각보다 인수분해가 쉽게 되어버릴수가 있습니다. Multi Prime RSA를 생각해서 풀어주시면 됩니다.

### 부채널 공격

보통 시간차나 전력차를 이용하는데 적절히 누수되는 정보를 있는데로 모은뒤에 위에있던 방법들과 함께 잘 풀어 나가면 됩니다.

### LSB에 대한 Oracle 이 있을경우

[Myria님](https://xerxes-break.tistory.com/455)의 예시를 보는것이 제가 이해한것보다도 정확할껍니다. 정확한 수학적 설명은 [Crypto StackExchange](https://crypto.stackexchange.com/questions/11053/rsa-least-significant-bit-oracle-attack)의 답변에 잘 나와있습니다.

~~죄송합니다. 좀 귀찮았습니다~~

### 오류 주입 공격

제가 만나본적은 없지만 이런게 존재는 한다고 합니다.

### ROCA

정말 드물지만 [CVE-2017-15361](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-15361)가 문제로 나옵니다. PoC를 찾아서 풉시다.
