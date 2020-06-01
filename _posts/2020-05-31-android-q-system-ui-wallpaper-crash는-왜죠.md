---
date: 2020-06-01T00:10:57.000Z
layout: post
title: Android Q System UI Wallpaper Crash는 왜죠
subtitle: TL;DR 색변환이 요상한 ICC Profile과 하모니를 이뤄 의도치 않은 값을 만들어 냅니다.
description: Android Q System UI Wallpaper Crash가 왜 일어났는지 알아봅시다
image: /assets/img/uploads/aasdfa.png
category: blog
tags:
  - Android
  - Crash analyze
author: party4bread
paginate: false
---
갑자기 재미있어 보여서 분석하게 됬습니다.

## Crash

크래시는 특정이미지를 배경화면으로 설정하면 납니다. 이 이미지를 접한것은 [이 트위터 글](https://twitter.com/UniverseIce/status/1266943909499826176)이였습니다.

실제로 Nexus 5 API 29와 Pixel 2 API 29에서 동작하는것을 확인했고 바로 logcat을 켰습니다.

```
FATAL EXCEPTION: AsyncTask #2
Process: com.android.systemui, PID: 6396
...
Caused by: java.lang.ArrayIndexOutOfBoundsException: length=256; index=256
	at com.android.systemui.glwallpaper.ImageProcessHelper$Threshold.getHistogram(ImageProcessHelper.java:139)
	at com.android.systemui.glwallpaper.ImageProcessHelper$Threshold.compute(ImageProcessHelper.java:105)
...
```

## Log analysis

해당 오류는 [ImageProcessHelper의 getHistogram함수](https://android.googlesource.com/platform/frameworks/base/+/master/packages/SystemUI/src/com/android/systemui/glwallpaper/ImageProcessHelper.java#139d)에서 발생하고 있고 해당 함수는 다음과 같이 생겼습니다.

```java
private int[] getHistogram(Bitmap grayscale) {
    int width = grayscale.getWidth();
    int height = grayscale.getHeight();
    // TODO: Fine tune the performance here, tracking on b/123615079.
    int[] histogram = new int[256];
    for (int row = 0; row < height; row++) {
        for (int col = 0; col < width; col++) {
            int pixel = grayscale.getPixel(col, row);
            int y = Color.red(pixel) + Color.green(pixel) + Color.blue(pixel);
            histogram[y]++;
        }
    }
    return histogram;
}
```

histogram배열이 256길이인데 257번째 요소를 접근하려해서 발생하는 크래시 입니다.

## Source analysis

getHistogram 함수가 크래시가 발생할때는 [toGrayscale 함수](https://android.googlesource.com/platform/frameworks/base/+/master/packages/SystemUI/src/com/android/systemui/glwallpaper/ImageProcessHelper.java#115)를 거쳐서 그레이스케일로 변환된 이미지를 처리하는데 RGB값의 총합이 256이 될때 발생합니다. 이 toGrayscale 함수는 [LUMINOSITY_MATRIX](https://android.googlesource.com/platform/frameworks/base/+/master/packages/SystemUI/src/com/android/systemui/glwallpaper/ImageProcessHelper.java#47)를 통해 ColorFilter를 하여 원본 이미지를 그레이스케일로 만듭니다. 이렇게 변환된 이미지에서 만약 RGB의 합이 256이 된다면 버그를 재현할 수 있습니다.

## Reproduction

원본 이미지에서 문제의 픽셀을 찾아본 결과 RGB가 (255,255,243)일때 Grayscale이 (55,183,18)이 되어서 발생하는데 직접 만든 이미지로 (255,255,243)를 넣었을때는 크래시가 생기지 않고 Grayscale이 (54,182,18)이나 255이 넘지않게 되어 버그가 발생하지 않지만 문제가 된 이미지의 ICC Profile인 `Google/Skia/E3CADAB7BD3DE5E3436874D2A9DEE126`로 색을 다시 매핑했더니 Grayscale 이 (55,183,18)되어 크래시가 발생했습니다.

![이미지](https://pbs.twimg.com/media/EZVUeHLUwAAj9GJ?format=jpg&name=small)

해당 이미지는 [이 곳](https://drive.google.com/file/d/1o2WgY6tBOR30GdJyLfrTHJaFNoJtwuXu/view) 에서 받아 볼수도 있습니다.

## Conclusion

ICC profile에 따라서 해석되는 색값의 실수 오차 때문에 ColorMatrix를 적용하여 그레이 스케일로 변환 할때 예상치 못한 값이 나온것 같습니다.  예상치 못한 값때문에 histogram을 구할때 Out of Index가 발생했고 이로 인한 크래시 인것으로 보입니다. 부트로더가 밀린다던가 OS가 고장난다던가는 아마 착각이 아닐까 싶고 Android Q에서 생기는 오류인것 같습니다. 

다만 호기심에 설정하면 System UI가 크래시가 나고 재시작되어 Wallpaper를 띄울때 또 크래시가 나는 순환에 걸리게 되므로 에뮬레이터로만 테스트 해보시길 바랍니다.

## Comment

 ICC Profile 이나 색 변환 때문에 이런 오류가 생기는건 흥미롭네요. 패치는 아마 변환 행렬을 수정하거나 histogram을 만들때 최댓값을 제한하는 코드만 넣으면 될듯하니 빠르게 되지않을까 생각합니다.