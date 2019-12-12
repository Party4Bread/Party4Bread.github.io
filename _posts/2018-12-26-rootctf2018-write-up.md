---
layout: post
title: "ROOTCTF2018 Write UP"
date: 2018-12-26 05:31:25
image: '/assets/img/root2018/prev.png'
description: ROOTCTF 2018 Write UP
category: 'Write UP'
tags:
- CTF
- Reversing
- Guessing
twitter_text:
introduction: ROOTCTF 2018 Write UP
---



# ROOTCTF2018

```
DS 앱잼안하고 뭐해?
```

Klevitreidous : 4896P : 6th

앱잼하면서 하느라 리버싱문제를 길게 보지 못해서 아쉽다. 앱잼을 하느라 대부분의 문제를 게싱으로 풀었다. 시간만 좀더 있었다면 립싱 올클도 가능했을꺼같은데.... ~~시간만 있으면 못하는게 뭐냐 CTF는 시간줄이는게 관건인데~~

# Misc

## MIC DROP

`FLAG{M1c..Dr0p..M1c..Dr0p..Welc0me_T0_R00T_CTF_2018!!}`

설명이 필요한가요? 그렇다면 당신은 -- 입니다.

## HubGIT

.git을 준다.

.git/objects에 커밋수에 비해 뭔가 많아서 다 뒤지면 뭔가 나올꺼같아서 다음과 같은 스크립트를 짜서 풀었다.

```python
import glob,zlib
for i in glob.glob(".git/objects/*/*"):
    print(zlib.decompress(open(i,"rb").read()))
```

`FLAG{GIT_8rob1em_7h@t_C4n_b3_50lv3d_in_O63_M1nu7e!}`

## Encoded_Code

코드를 준다.

맨위에 두 문자열이 왠지 custom table base64같아서 다음과 같은 코드를 짜서 풀었다.

```python
import string,base64
table="lksOm9BteESFLuDaNiUho1dwnRG0yc+jAKpXgYrH6xIQPW4vM7q32TbfZC8/5VJz"
orige=string.ascii_uppercase+string.ascii_lowercase+string.digits+"+/"
tb=str.maketrans(table,orige)
print(base64.b64decode("ig7kifWsutu9w3n2w214n3kgLoCHwTiojN==".translate(tb)))
```

`FLAG{B4sE_64_Enc0d1Ng_TT}`

## FindMe!

hxd같은 에디터로 읽고 FLAG{로 검색하면 플래그가 나온다. 사실 문제가 처음 나왔을때는 파일의 링크가 잘못되어있어서 그 링크를 게싱해서 먼저 파일을 받았는데 그게 문제의 핵심인줄 알았다.

*학교에서 받기엔 파일이 너무커서 플래그는 적지 않겠습니다*

## Format

~~고마워 혁주야!~~

손상된 PNG파일이 주어진다.

여러 PNG구조에서 특히나 사이즈 같은것들이 비정상적을 큰걸 알수있다. 따라서 널바이트여야 하는곳이 바뀐것이라 추측할수있고 풀수있는 문제를 내기 위해선 `0x00->X`면 `X->0x00`이 되어야 한다는걸 추측할수 있다. 이 문제는 `0x00<->0x10` 이였고 다음과 같은 코드로 PNG를 복구했다 .

```python
t=bytearray(open("Format.PNG","rb").read())
t10=filter(lambda x:x is not None, [i if j == 0x10 else None for i, j in enumerate(t)])
t00=filter(lambda x:x is not None, [i if j == 0x00 else None for i, j in enumerate(t)])
for i in t10:
    t[i]=0x00
for i in t00:
    t[i] = 0x10
open("Fixed.png","w+b").write(t)
```

`FLAG{NU;L?_ZeR0?_}`

# Web

## Secret_chest

단순 JS분석 문제이다.

아래 코드를 문제사이트에서 실행하면 FLAG가 나온다.

```javascript
lv=10
sessionStorage.setItem("lv",lv);
$.ajax({
	type:"post",
	data:{"gang":sessionStorage.getItem("lv")},
	dataType:"text",
	url:"./",
	success:function(result){
    	console.log('ok');
	}
});
location.href="?open";
```

`FLAG{1!tTle_2sy_3Asy_4l@g}`

# Reversing

## ROOT_Process_1

자기자신의 프로세스명이 특정 문자열이면 Correct가 나오는 바이너리이다.

그 문자열은 아래 파이선 코드로 복구할수있다.

```python
t=bytearray("Very_easy_Reversing!".encode("utf8"))
c=[31,41,66,15,58,50,40,29,23,49,19,21,71,87,65,69,71,11,31,68]
print("".join([chr(i^j) for i,j in zip(t,c)]))
```

~~지하철에서 풀어서 기억이 이거밖에 안납니다~~

## ROOT_Process_2

In memory execution을 하는듯한 바이너리가 주어진다.

execute되는 바이너리는 쉽게 덤프 할 수 있고 해당 바이너리는 argv를 비교하여 Correct를 출력하는데 단순히 seed가 지정된 rand값들과 xor하고 비교하는 루틴이라 쉽게 flag를 얻을수 있다.

```python
import ctypes
libc=ctypes.CDLL("msvcrt.dll")
libc.srand(1)
tt="6F 78 2E 13 0C 35 00 7A 72 0F 44 20 62 5A 54 2E 3E 35 4E 08 7B"
tt=list(map(lambda x:int(x,16),tt.split()))
for i in tt:
    print(chr(i^libc.rand()%0x7F),end='')
```

## CrackMe

base64에 추가적 연산을 섞어서 동작하길래 분석 할 시간이 없어서 얍삽이로 풀었다. 사이드 채널 만세!

```python
import r2pipe,os,binascii,string

txt="39xs8IzeMEDEsLl/N+ps8yHgRGB9osPSRwD1RwC7QHVdr0xx5qp/fFMCRMFFccMaRGAEcHSEnFVs8w5yI7QlRwDzleMtykO2mbSEqDRvYhfcRwCbi7hsMiJvzbSEebSEzj1yBC9sMomuwHV7K+Zs8ypBUbVdko9oRGzsR+tsMhp8msHV4c2QEP4CGKFsMsps8H7kuclvIBwRRGzRrKyi+akVRwCAIXAAdswNntZ5OIzic1LbHoP/ibd/xdw7FWBsMmfi6VmryjVs8zhSNeZYRGYmRoZmfpskXa8qOrV7IdDQRwCWXL1s8GlpFrSEL/3rRwD0UJCQ6iJs8vG4LDLW2bSEBBeoRwF9eS1sModdQDUnh+cOxBZ8qGts8uhs8CnKKXV7jHSEXFts8yN4RGz7CtJ="

r2 = r2pipe.open('CrackMe')
charset = string.ascii_letters + string.digits + "_-{}.!,"

def encodeit(ipt):
	r2.cmd("ood")
	r2.cmd("dcu 0x0040078a")
	r2.cmd("wao nop")#read함수 역활
	r2.cmd("dr rax="+str(len(ipt)+1))
	r2.cmd('wx '+(ipt+'\n\0').encode('hex')+' @ rsi')
	r2.cmd("dcu 0x00400d67")
	res=r2.cmd("psz 410 @ r13")#암호화된 입력값
	return res 

opt=""
base="FLAG{"
while txt!=opt and encodeit(base+'\n')!=txt:
	ssapable=[]	
	able=[]
	maybe=[]
	for i in charset:
		opt=encodeit(base+i)
		if txt[:len(opt)]==opt[:len(opt)]:
			ssapable.append(i)
		elif txt[:len(opt)-4]==opt[:len(opt)-4]:
			able.append(i)
		elif txt[:len(base)*8+8]==opt[:len(base)*8+8]:
			maybe.append(i)
	print(ssapable,able,maybe)
	base+=raw_input("PICK:")[0]
	print base
print base

r2.quit()
```

위와같이 암호화되서 나오는 구간에서 값을 받고 수동으로 값을 추정해서 넣는 방식으로 한자리씩 게싱했다.

`FLAG{I_w1ll_G0_t0_Y0u_lik3_The_F1rst_Sn0w}`

-----------

문제의 퀄이 다 조금씩 올라갔다. 아무래도 경제님의 영향이 있지않을까.