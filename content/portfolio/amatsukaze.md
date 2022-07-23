---
title: "Amatsukaze-ロケット"
date: 2022-07-23T01:28:40+09:00
draft: false
---
{{< youtube MzzBfFVT36c >}}
  
開発期間: 3ヶ月  
使用言語: C++ (Mbed compiler)    
担当: サブPM, 電装, ロガー  
[ソースコード](https://os.mbed.com/teams/CORE/code/Logger/)  


# 概要  
  
 ![system](/img/amatsukaze/Picture1.jpg)  
点火から機体回収における全てのシーケンスを実行すること，またフライトデータおよびロケットを回収し，解析結果を取得するを目的としたロケット  


# ロガー

![system](/img/amatsukaze/Picture2.jpg)
![system](/img/amatsukaze/Picture3.jpg)  
 
 ロケットのフライトデータを記録するためのロガーの製作を行った．基盤の設計はAUTODESK社のEAGLEにて行った．  
 ロガーの特徴として，気圧センサ(LPS22HB)，9軸センサ(MPU9250)を搭載し，これらのセンサデータはMicroSDカードに記録される．  
 マイコンはSTM303K8T6を使用し，さらに無線機を搭載によりダウンリンクを行える設計にした．マイコンへの書き込み用端子としてミニUSB端子，Serial通信を行うためのマイクロUSBを分けることで，誤動作の帽子を行った．  
また，GNDと5Vの配線をUSB標準の規格に合わせることによって謝ってusb接続をしてもロガーが破損しないように工夫を行った．  

