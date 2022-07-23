---
title: "筋電義手"
date: 2022-07-20T01:28:40+09:00
draft: false
---
{{< youtube xli2Wd_343k >}}
  
開発期間: 2年  
使用言語: python3.7      
論文: [Deep Learning for Gesture Recognition based on Surface EMG Data](https://ieeexplore.ieee.org/document/9661533)  
ソースコード1: [リアルタイム分類システム](https://github.com/retact/real-time-myo)  
ソースコード2: [Myo Logger](https://github.com/retact/Myo-Logger)  
[論文収集](https://ps.retact.net/)  

# 概要  
腕の筋電図(筋肉の信号)から3層CNNモデルにより，52動作のジェスチャー分類と制御を可能にした義手の研究です.  
また，リアルタイムシステムの構築によって，平均195ms以下のレスポンスタイムで義手に反映させることに成功しました．  

# 目的
日本における能動義手の普及率が各国と比較して低い状態にある原因として，ジェスチャが限られていることや，操作性が低いことがあげられます．ここで深層学習モデルを用いた分類を行うことによって，ジェスチャの動作数を増やし，リアルタイムシステムを構築することによって300ms(人間が知覚して違和感の感じない制限時間)以下のレスポンスタイムで動作をする機能性を実現する必要があります．  


