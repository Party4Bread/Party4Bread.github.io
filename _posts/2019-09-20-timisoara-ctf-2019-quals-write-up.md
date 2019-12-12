---
layout: post
title: "Timisoara CTF 2019 Quals Write-UP"
date: 2019-09-20 14:24:34
image: '/assets/img/tim2019/prev.png'
description: 티미소아라 CTF 2019 예선 문제 풀이
category: 'Write UP'
tags: 
- Cryptography
- CTF
- Reversing
- Algorithm
twitter_text:
use_math: true
introduction: 이미 사라진 대회입니다.
---
# Timisoara CTF 2019 Quals Write UP

Timisoara CTF 2019 예선이 끝났습니다. 이번에도 2등으로 본선에 올라가게 되었네요. 

~~대회가 없어졌지만요~~

이번에는 올솔을 했네요. 난이도가 기간에 비해 쉬운점이 있어서인지 팀원이였던 `howdayz`군이 잘해준 덕인지 가능했던거 같습니다. 이번에는 많은 문제를 풀지는 못했습니다. 최근 개발쪽에 좀 더 공부를 해서인지 감이 좀 떨어졌더라구요. 이번에는 기억나는 문제들만 추려서 라업을 써보도록 하겠습니다.

## Don’t trust time (200pts)

`time_crypt.cpp`에 의하면 파일을 [AES](https://ko.wikipedia.org/wiki/%EA%B3%A0%EA%B8%89_%EC%95%94%ED%98%B8%ED%99%94_%ED%91%9C%EC%A4%80)-[CBC](https://ko.wikipedia.org/wiki/%EB%B8%94%EB%A1%9D_%EC%95%94%ED%98%B8_%EC%9A%B4%EC%9A%A9_%EB%B0%A9%EC%8B%9D#%EC%95%94%ED%98%B8_%EB%B8%94%EB%A1%9D_%EC%B2%B4%EC%9D%B8_%EB%B0%A9%EC%8B%9D_(CBC)) 알고리즘을 사용해서 암호화하는데 암호화 할때 쓸 키를 time() 함수의 반환 바이트를 [SHA1](https://ko.wikipedia.org/wiki/SHA)로 해싱한 결과를 상위16바이트만 사용하여서 만들고, NULL IV를 사용하여 암호화 합니다.

복호화를 위해서는 time()함수의 반환값을 예측하여야 하는데 `enc_flag.txt` time() 함수의 반환값은 `enc_flag.txt`의 생성시간에서 얻을수 있습니다.

따라서 복호화 소스는 다음과 같습니다.

```python
from Crypto.Cipher import AES
from pathlib import Path
import hashlib

p=Path("enc_flag.txt")
aaa=p.open("rb").read()

key = hashlib.sha1(int(p.stat().st_mtime).to_bytes(4,'little')).digest()
iv = '\x00'*16
encryptor = AES.new(key[:16], AES.MODE_CBC, iv)
print(encryptor.decrypt(aaa))
```

## 3 magic numbers (300pts)

Description에서 굳이 중국인을 언급한걸을 보아서 중국인 나머지 정리를 써야 할꺼 같네요. 

중국인 나머지 정리는 다음과 같습니다. 소수인 p1, p2 ,p3과 자연수 c1, c2, c3가 주어지고, 다음과 같은 관계라면

$$c_1 = m \pmod {p_1} \\ c_2 = m \pmod {p_2} \\ c_3 = m \pmod {p_3}$$

m값을 알아낼수가 있다는겁니다.

하지만 이 문제에서 준 값은 

$$t_1=t\pmod{p_1} \\ t_2=t^{2019}\pmod{p_2} \\ t_3=t^{2019^{2019}}\pmod{p_3}$$

이라서 지수를 제거해야 CRT(중국인 나머지 정리)를 적용할수 있습니다.

지수를 없애는 법은 다음과 같습니다.

오일러의 정리가 다음과 같기 때문에

$$t^{\phi(n)}\equiv 1 \pmod n$$

아래의 식이 성립함을 증명할수 있습니다.

$$ed \equiv 1 \pmod {\phi(n)} \\ t^{ed}\equiv t \pmod n$$

n과 t가 소인수라고 가정하면

$$ed = 1 + h\phi(n)\space(h\ge0)\\ t^{ed}=t^{1+h\phi(n)}=t(t^{h\phi(n)})\\ t(t^{h\phi(n)}) \equiv t(1)^h \pmod n$$

이고

$$t^{ed}\equiv t \pmod n$$

이게 됩니다.

그래서 만약 $$2019d_2\equiv 1 \pmod {\phi({p_2}^4)}$$ 인 `d2`와 $$2019d_3\equiv 1 \pmod {\phi({p_3}^4)}$$ 인 `d3`를 구할수 있다면 지수를 없앨수 있습니다. 그렇다면 원래의 메세지인 `t`도 구할수 있겠죠

그래서 원본 메세지는 다음과같은 소스를 이용해서 해독할수 있습니다.

```python
p1 = 492876863
p2 = 472882049
p3 = 573259391

from functools import reduce
def chinese_remainder(n, a):
    sum = 0
    prod = reduce(lambda a, b: a*b, n)
    for n_i, a_i in zip(n, a):
        p = prod // n_i
        sum += a_i * mul_inv(p, n_i) * p
    return sum % prod

def mul_inv(a, b):
    b0 = b
    x0, x1 = 0, 1
    if b == 1: return 1
    while a > 1:
        q = a // b
        a, b = b, a%b
        x0, x1 = x1 - q * x0, x0
    if x1 < 0: x1 += b0
    return x1

t1 = 53994433445527579909840621536093364
t2 = 36364162229311278067416695130494243
t3 = 31003636792624845072184744558108878

d2=mul_inv(2019,(p2-1)*(p2**3))
d3=mul_inv(2019,(p3-1)*(p3**3))
print(d2,d3)

mt1=t1
mt2=pow(t2,d2,p2**4)
mt3=pow(t3,d3**2019,p3**4)

m=chinese_remainder([p1**4,p2**4,p3**4],[mt1,mt2,mt3])

try:
    print(("0%x"%m).decode('hex'))
except:
    print(("%x"%m).decode('hex'))
```

## Secure file transfer

문제에서 주는 패킷에서 10.0.2.2 에서 오는 TCP 패킷을 보면 PE 헤더를 볼수 있습니다.

패킷을 덤프를 뜨면 PE의 헤더가 상당히 손상되어있습니다. 따라서 다음과 같은것들을 수정 해주어야 합니다.

- MZ magic의 DOS헤더 추가
- PE의 OPTIONAL HEADER Magic을 PE32에서 PE64로 변경
- PE OPTIONAL HEADER 의 checksum 재계산
- **PE FILE HEADER 의 Characteristics flag중 excutable 을 true로**

그러면 유효한 PE 파일이 되고, 실행하면 플래그를 줍니다.

## Linear recurrence

이 문제는 무작위의 선형 점화식을 던져줍니다. 그리고 k번째 수열의 값을 666013으로 나눈 값을 반환해야 합니다.

k값의 최댓값이 99999999 이고 시간 제한이 1초라서 $$O(k)$$의 시간 복잡도를 갖는 알고리즘은 아마 잘 동작하지 못할껍니다.

근데, 문제에서 분명 점화식의 항 갯수인 N은 k보다 작다고 하였지만 사실은 2,3,4만 들어옵니다.

그래서 선형 점화식 계산을 행렬곱셈으로 바꿔주면 문제를 풀기에는 충분한 $$O(N^3\lg k)$$ 알고리즘을 만들수 있습니다.

거의 대부분의 선형 점화식은 행렬식으로 바뀔수 있습니다. 예를 들어, 피보나치 수열을 보자면

$$F_n = F_{n−1} + F_{n−2}$$ 인데 이는 선형적입니다. 따라서 이 식은 다음과 같은행렬로 바뀔수 있습니다.

$$\begin{bmatrix} F_{n+1} \\ F_{n+2} \end{bmatrix} = \begin{bmatrix} 0 & 1 \\ 1 & 1 \end{bmatrix} \begin{bmatrix} F_n \\ F_{n+1} \end{bmatrix}$$

이 식은 이렇게도 바꿔 쓸수있습니다.

$$\begin{bmatrix} F_{n+1} \\ F_{n+2} \end{bmatrix}=\begin{bmatrix} 0 & 1 \\ 1 & 1 \end{bmatrix}^n\begin{bmatrix} F_1 \\ F_2 \end{bmatrix}$$

다른 선형 점화식은 어떻게 이렇게 나타 낼수 있을까요?

간단합니다, 위에있는 $$\begin{bmatrix} 0 & 1 \\ 1 & 1 \end{bmatrix}$$ 행렬을 T행렬이라고 치면

우선, 결과 항과 입력 항은 1xN크기이고 T행렬은 NxN행렬이 될껍니다.

따라서 통상적으로는 이런 형태가 되겠죠.

$$\begin{bmatrix}F_{n+1} \\ F_{n+2} \\ \vdots \\ F_{n+N} \end{bmatrix} = T^n \begin{bmatrix} F_1 \\ F_2 \\ \vdots \\ F_N \end{bmatrix}$$

T를 만들면 되겠네요!

T만드는 것도 쉽습니다. 만약 점화식이 $$a_{n+1}=c_na_n+c_{n-1}a_{n-1}+c_{n-2}a_{n-2}+\cdots+c_1a_1$$ 형태라면 T행렬은 다음과 같은 꼴이 됩니다.

$$T= \begin{bmatrix} 0 & 1 & 0 & 0 & 0 & \cdots & 0 \\ 0 & 0 & 1 & 0 & 0 & \cdots & 0 \\ 0 & 0 & 0 & 1 & 0 & \cdots & 0 \\ 0 & 0 & 0 & 0 & 1 & \cdots & 0 \\ \vdots&\vdots& \vdots &\vdots&\vdots& \ddots & \vdots \\ c_1&c_2&c_3&c_4&c_5&\cdots&c_N \end{bmatrix}$$

따라서 전체 식은 다음과 같이 되겠네요!

$$\begin{bmatrix}F_{n+1} \\ F_{n+2} \\ F_{n+3} \\ F_{n+4} \\ \vdots \\ F_{n+N} \end{bmatrix} = \begin{bmatrix} 0 & 1 & 0 & 0 & 0 & \cdots & 0 \\ 0 & 0 & 1 & 0 & 0 & \cdots & 0 \\ 0 & 0 & 0 & 1 & 0 & \cdots & 0 \\ 0 & 0 & 0 & 0 & 1 & \cdots & 0 \\ \vdots&\vdots& \vdots &\vdots&\vdots& \ddots & \vdots \\ c_1&c_2&c_3&c_4&c_5&\cdots&c_N \end{bmatrix}^n \begin{bmatrix} F_1 \\ F_2 \\ F_3 \\ F_4 \\ \vdots \\ F_N \end{bmatrix}$$

이것만 보면 그냥 $$O(k)$$ 보다 더 큰 $$O(N^3k)$$ 짜리의 끔직한 알고리즘입니다. 행렬곱이 회당 $$O(N^3)$$의 복잡도를 갖기 때문이죠. 

하지만 [exponential by squaring](https://en.wikipedia.org/wiki/Exponentiation_by_squaring) 을 사용하면 $$O(k)$$가  $$O(\lg k)$$로 됩니다. 와!

이제 이 알고리즘의 시간복잡도는 $$O(N^3\lg k)$$네요 *(혹은 $$O(N^{2.372}\lg k)$$)* 

```python
import numpy as np
from pwn import *

N=666013

def exponentiation(bas, exp): 
    if (exp == 0): 
        return 1; 
    if (exp == 1): 
        return bas % N; 
    t = exponentiation(bas, int(exp / 2))
    t = (t * t) % N
    if (exp % 2 == 0): 
        return t; 
    else: 
        return ((bas % N) * t) % N; 

r = remote("89.38.208.143",22022)

for i in range(10):
	print(r.readline())
	print(r.readline())
	n,k=map(int,r.readline().split())
	hm=list(map(int,r.readline().split()))
	matlist=[[1 if i==j else 0 for j in range(n)] for i in range(1,n)]
	matlist.append(list(reversed(hm[::2])))
	transmat = np.matrix(matlist)
	initmat = np.matrix([[i] for i in reversed(hm[1::2])])
	res = (exponentiation(transmat,k-n)*initmat)%N
	r.sendline(str(int(res[n-1][0])))
r.interactive()
```

`TIMCTF{Matrix_multiplication_OP_please_n3rf}`