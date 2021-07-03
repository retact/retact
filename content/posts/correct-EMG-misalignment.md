---
title: "相互相関関数を用いてEMGのズレを推定する"
date: 2021-07-03T01:28:40+09:00
draft: false
---
2つのMyoデバイスを使ってそれぞれEMGを取得した際にこのようにデータのズレが生じてしまいます.  
EMGと同時に取得されるタイムスタンプなどで補正すれば問題はないのですが，同じ腕の信号を取得しているため今回は相互相関関数を使ってこの2つの信号のズレを推定してみたいと思います.  
また今回使用したコードは[こちら](https://github.com/retact/sEMG-Cross-correlation)にまとめてあります．
 ![EMG1](/img/correct-EMG-misalignment/Myo-EMG1.png)  


## 相互相関関数(Cross-correlation function)
相互相関関数は2つの信号の類似性を見るための関数であり，xとyの平均がそれぞれ0で2つの信号が連続信号であるとき以下のような式で表されます.  
  

 ![C-corr-f](/img/correct-EMG-misalignment/C-corr-f.png)  
  


同時に観測された二つの不規則信号に対して,x(t)と時間τだけずらしたy(t+τ)の相関を示します.  
このような性質からレーダーシステムなどで送信信号と受信信号を比較して目標物の位置を特定する際に利用され,整合フィルタ(Match Filter)とも呼ばれています.  
## Raw-EMGの相関関数で推定する.  
python で相互相関関数を利用する場合はnumpyのcorrelate関数やこれに基づいたmatplotlib.pyplotのxcorr関数を利用する方法などがありますが，今回はnumpyのcorrelate関数を利用します．  
[numpy.correlate](https://numpy.org/doc/stable/reference/generated/numpy.correlate.html)  
[matplotlib.pyplot.xcorr](https://matplotlib.org/stable/api/_as_gen/matplotlib.pyplot.xcorr.html)  
```python
import numpy as np 

np.correlate(a, v, mode='valid') 
```
また実際データに相互相関関数を利用する際にはデータを正規化する必要があります．今回は標準化の処理を行います．  

```python
dev1_emg1=(dev1_emg1-np.mean(dev1_emg1))/np.std(dev1_emg1)
dev2_emg1=(dev2_emg1-np.mean(dev2_emg1))/np.std(dev2_emg1)
```

そのようにして実際に実装すると以下のようなコードで以下のような結果を得ることができます.
```python
import numpy as np 
import myplotlib.pyplot as plt

#　Standardization
dev1_emg1=(dev1_emg1-np.mean(dev1_emg1))/np.std(dev1_emg1)
dev2_emg1=(dev2_emg1-np.mean(dev2_emg1))/np.std(dev2_emg1)

# Cross-correlation function
corr=np.correlate(dev1_emg1, dev2_emg1,"full")
estimated_delay = corr.argmax() - (len(dev2_emg1) - 1)
print("estimated delay is " + str(estimated_delay))

# Plotting
plt.figure(figsize=(15,8))
plt.subplot(2, 1, 1)
plt.plot(np.arange(len(dev1_emg1)), dev1_emg1)
plt.plot(np.arange(len(dev2_emg1)) + estimated_delay, dev2_emg1)
plt.xlabel('Number of data',fontsize=15)
plt.ylabel('EMG',fontsize=15)
plt.title("Myo raw-EMG signals (emg1)",fontsize=25)
plt.xlim([0, len(dev1_emg1)])
plt.subplot(2, 1, 2)
plt.plot(np.arange(len(corr)) - len(dev2_emg1) + 1, corr, color="r")
plt.xlabel('Delay',fontsize=15)
plt.ylabel("Cross-correlation",fontsize=15)
plt.xlim([0, len(dev1_emg1)])
plt.show()
```
![MyoEMG2](/img/correct-EMG-misalignment/Myo-EMG2.png)  

結果からズレが推定できておらず，2つの信号の差が少しは小さくなったものの補正できていないことがわかります．こうなってしまった原因として，Myoから取得しているデータがbyteデータであることが挙げられます．byteデータだと扱える値が-128-127であり，Myoの内部でそれより大きい値や小さい値はそれぞれ-128と127で置換されてしまうため，信号同士の相関関係を強く示す場所がいくつか出てしまいます．  
この結果を踏まえて取得しているデータを元にズレを推定するために今回は取得した信号をある一定の範囲で積分した値を算出し，その信号同士の相関関係でズレを推定してみたいと思います．  

## IEMGの相互相関関数で推定する．  
IEMGとはIntegrated ElectroMyoGraphyの略でEMGの時間的特徴量としてジェスチャー分類する際に用いられます．実際にはEMGを一定区間で積分した値で次のような式で表すことができます．   
![IEMG](/img/correct-EMG-misalignment/IEMG.png)  
これをpythonで実装すると以下のような関数で任意の信号に対して実装することができます．  
```python
import numpy as np 

IEMG1=[]

def to_IEMG(l, n):#l=signal,n=Segment data count 
    for i in range(0, len(l), n):
        yield np.sum([abs(x) for x in l[i:i + n]])

IEMG1=list(to_IEMG(dev1_emg1,5))
```
この関数を用いて2つの信号に対してIEMGを算出したあとに相互相関関数を用いてズレを推定していきます.

まずはそれぞれの信号をIEMGの関数に代入し積分していきます．  

```python 
IEMG1=[]
IEMG2=[]
dataf=10

# IEMG function
def to_IEMG(l, n):
    for i in range(0, len(l), n):
        yield np.sum([abs(x) for x in l[i:i + n]])

IEMG1=list(to_IEMG(dev1_emg1,dataf))
IEMG2=list(to_IEMG(dev2_emg1,dataf))

# Plotting
plt.figure(figsize=(15,3))
plt.plot(IEMG1)
plt.xlabel('Number of data',fontsize=15)
plt.ylabel('IEMG',fontsize=15)
plt.title("Myo IEMG(50ms) device1",fontsize=25)
plt.show()

plt.figure(figsize=(15,3))
plt.plot(IEMG2)
plt.xlabel('Number of data',fontsize=15)
plt.ylabel('IEMG',fontsize=15)
plt.title("Myo IEMG(50ms) device2",fontsize=25)
plt.show()
```
![IEMG graph 1](/img/correct-EMG-misalignment/Myo-EMG4.png)  
![IEMG graph](/img/correct-EMG-misalignment/Myo-EMG3.png)  

見た目も先ほどのrawデータより相関が出そうな気がします．  
この変換したIEMGに先ほどと同様の相互相関関数を用いてズレを推定していきます．  

```python
#　Standardization
IEMG1=(IEMG1-np.mean(IEMG1))/np.std(IEMG1)
IEMG2=(IEMG2-np.mean(IEMG2))/np.std(IEMG2)

# Cross-correlation function
corr=np.correlate(IEMG1, IEMG2,"full")
estimated_delay = corr.argmax() - (len(IEMG2)-1)
print("estimated delay is " + str(estimated_delay))

# Plotting
plt.figure(figsize=(15,8))
plt.subplot(2, 1, 1)
plt.plot(np.arange(len(IEMG1)), IEMG1)
plt.plot(np.arange(len(IEMG2)) + estimated_delay, IEMG2 )
plt.xlabel('Number of data',fontsize=15)
plt.ylabel('IEMG',fontsize=15)
plt.title("Myo IEMG signals (emg1)",fontsize=25)

plt.xlim([0, len(IEMG1)])
plt.subplot(2, 1, 2)
plt.plot(np.arange(len(corr)) - len(IEMG2) + 1, corr, color="r")
plt.xlabel('Delay',fontsize=15)
plt.ylabel("Cross-correlation",fontsize=15)
plt.xlim([0, len(IEMG1)])
plt.show()
```

![IEMG corr](/img/correct-EMG-misalignment/Myo-EMG5.png)  
図でもわかる通りいい感じに相関関係が表示できたかと思います．  
ここで算出したDelayをRawデータのデータ点数に変換してraw-EMGデータに反映して表示させてみます．
```python
EMGdelay=estimated_delay*dataf
plt.figure(figsize=(15,3))
# EMG1
plt.plot(np.arange(len(dev1_emg1)), dev1_emg1)
# EMG2 + EMGdelay 
plt.plot(np.arange(len(dev2_emg1))+ EMGdelay, dev2_emg1)
plt.xlabel('Number of data',fontsize=15)
plt.ylabel('EMG [-128-127](mV)',fontsize=15)
plt.title("Myo raw-EMG signals (emg1)",fontsize=25)
plt.xlim([0, len(dev1_emg1)])
plt.show()
```  
![rawEMG](/img/correct-EMG-misalignment/EMG6.png)   
 結果のとおり，いい感じにデータのズレを修正できたかと思います．  
しかし今回実際に相関関係を算出した際に利用したIEMGはraw-EMGを10点区間で積分した値なので，データのズレも+-10点ほど存在することになります．  
 そのためデータ点数のズレをできるだけ小さくしたい場合，相互相関関数においてはMyoで取得したEMGデバイスのデータのズレを推定し，修正することは難しいと思います．  
使用するデータの波形が綺麗でサンプリングレートが高いものであればもう少しうまくいっていたかもしれません．  
また今回取得したデータはEMGの計測部分が異なるため，筋肉の伝達の誤差と筋肉から発生する信号の大きさによって完璧な相互相関があるわけではないことから，少なからずデータに対しての誤差は生じることが考えられます．  
このようなことからの2つのMyoデバイスを使用して取得した際に発生するズレを補完する方法として相互相関関数は不向きであることがわかりました．  
取得するデータや他のシステムの遅延量を求める際には利用できるので，使用法を考えて利用していきたいと思います.  

  




