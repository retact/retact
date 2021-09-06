---
title: "TwitterのタイムラインをDiscordに表示させる"
date: 2021-09-07T01:28:40+09:00
draft: false
---

# はじめに  
 
 みなさんはDiscord使っていますでしょうか．  
 僕の使っているサーバでは[分報](https://craftsman-software.com/posts/56)というものがあり,みんなのやっていることを共有できてとても楽しかったのですが最近身の回りの人は沙汰がなく，Twitterの方が捗っている方が多い傾向にあります．  
 それだと少し寂しい気がしたので,みんながTweetしたらDiscordに表示させるようにしたらいいのではという思いから,雑ではありますが実装してみました．  
 
※今回使うTwitterStreamAPIの鍵垢のツイートの抽出はよくわからなかったので本文は公開垢のみの抽出とWebhooksを用いた出力について書かれています．  
 
実装したコードは[こちら](https://github.com/retact/twi-cord)に載せておきます．


# Twitterのツイートを読み込むために  
 
まずタイムラインを抽出するためにご自身のtwitterアカウントでAPIキーとアクセストークンを取得する必要があります．  
 [こちら](https://www.itti.jp/web-direction/how-to-apply-for-twitter-api/)を参考にTwitterのAPI利用申請を行えば(少しUIは異なるものの)問題なく行えるかと思います．
  

 
ここで入手した  
 ○ Consumer API key  
 ○ Consumer API secret key  
 ○ Access token  
 ○ Access token secret  
 の4つは後ほど使用します．
  

[Twitter APIを使用したOAuth](https://developer.twitter.com/ja/docs/basics/authentication/overview/oauth)  
 
# Twitter Streaming API  

今回はTwitter Streaming APIを使用するために[Tweepy](https://docs.tweepy.org/en/v3.5.0/index.html)ライブラリを使用させていただきました．
  


# Tweetをとりあえず出力する．  
 
config.pyに先ほど取得した  
 ○ Consumer API key  
 ○ Consumer API secret key  
 ○ Access token  
 ○ Access token secret  
 を記入してtwi-cord.pyを実行するとツイートが実行画面に表示されるかとおもいます．
  

[※ デフォルトのアクセスレベルでは、最大400のトラックキーワード、5000のフォローユーザーID、25のロケーションボックスが許可される](https://developer.twitter.com/en/docs/twitter-api/v1/tweets/filter-realtime/api-reference/post-statuses-filter)
  

```python:config.py
CONFIG = {
    "CONSUMER_KEY": "Consumer API keyを記入してください",
    "CONSUMER_SECRET": "Consumer API secret keyを記入してください",
    "ACCESS_TOKEN": "Access tokenを記入してください",
    "ACCESS_SECRET": "Access token secretを記入してください",
}
```
```python:twi-cord.py
import tweepy
from config import CONFIG

auth = tweepy.OAuthHandler(CONFIG["CONSUMER_KEY"], CONFIG["CONSUMER_SECRET"])
auth.set_access_token(CONFIG["ACCESS_TOKEN"], CONFIG["ACCESS_SECRET"])
# APIインスタンスを作成
api = tweepy.API(auth, wait_on_rate_limit=True, wait_on_rate_limit_notify=True)

# フォローしているユーザーのwatch_listを作成
followee_ids = api.friends_ids(screen_name=api.me().screen_name)
watch_list = [str(user_id) for user_id in followee_ids]
watch_list.append(str(api.me().id))

# デフォルトのアクセスレベルではfollowには5000人まで指定できるため
assert(len(watch_list) <= 5000)

class MyStreamListener(tweepy.StreamListener):
    def on_status(self, status):
	# @ツイートは表示しない．
        if "@" in status.text:
            pass
        else:
            print('-------------------------------------------')
            print('name:' + status.user.name)
            print(status.text)

myStreamListener = MyStreamListener()
myStream = tweepy.Stream(auth=api.auth, listener=myStreamListener)
myStream.filter(follow=watch_list)
```
  
プライバシーの関係上大部分を隠しておりますが以下のようにフォローしてる人のTweetが表示されると思います．
![](https://storage.googleapis.com/zenn-user-upload/beeaa93cc6ebb5afa8407df3.png)
  

# Webhookで雑にDiscordに投げる．
とりあえずDiscordにタイムラインを表示させたいだけなので今回はWebhookを用います．  

  
## DiscordでURLを生成
わからない方はこちらを参考にすれば問題なく生成できるかと思います．  

○ [Intro to Webhooks](https://support.discord.com/hc/en-us/articles/228383668-Intro-to-Webhooks)  
 こちらで生成されたURLは後ほど使います．  
 
Webhooksの構造について知りたい方は[こちら](https://discord.com/developers/docs/resources/webhook)が参考になるかと思います．  

##  DiscordにTweetを表示 

先ほど記述したコードにWebhookで送り込む部分を付け足します．
```diff python:config.py
CONFIG = {
    "CONSUMER_KEY": "Consumer API keyを記入してください",
    "CONSUMER_SECRET": "Consumer API secret keyを記入してください",
    "ACCESS_TOKEN": "Access tokenを記入してください",
    "ACCESS_SECRET": "Access token secretを記入してください",
+   "WEBHOOK_URL": "Webhook URLを記入してください ",
}
```
```diff python:twi-cord.py
import tweepy
+import requests
+import json
from config import CONFIG

auth = tweepy.OAuthHandler(CONFIG["CONSUMER_KEY"], CONFIG["CONSUMER_SECRET"])
auth.set_access_token(CONFIG["ACCESS_TOKEN"], CONFIG["ACCESS_SECRET"])
# APIインスタンスを作成
api = tweepy.API(auth, wait_on_rate_limit=True, wait_on_rate_limit_notify=True)

+webhook_url = CONFIG["WEBHOOK_URL"]

# フォローしているユーザーのwatch_listを作成
followee_ids = api.friends_ids(screen_name=api.me().screen_name)
watch_list = [str(user_id) for user_id in followee_ids]
watch_list.append(str(api.me().id))

# デフォルトのアクセスレベルではfollowには5000人まで指定できるため
assert(len(watch_list) <= 5000)

class MyStreamListener(tweepy.StreamListener):
    def on_status(self, status):
	# @ツイートは表示しない．
        if "@" in status.text:
            pass
        else:
+	    # ユーザーの情報を取得
+	    user_id = status.user.id
+           user = api.get_user(user_id)
+           main_content = {
+                  'username': status.user.name,
+                  'avatar_url': user.profile_image_url_https,
+                  'content': status.text
+           }
+           headers = {'Content-Type': 'application/json'}
+           response = requests.post(webhook_url, json.dumps(main_content), headers=headers)

            print('-------------------------------------------')
            print('name:' + status.user.name)
            print(status.text)

myStreamListener = MyStreamListener()
myStream = tweepy.Stream(auth=api.auth, listener=myStreamListener)
myStream.filter(follow=watch_list)
```
  
プライバシーの関係上大部分を隠しているのでみづらいですがこんな感じで表示されるはずです．
![](https://storage.googleapis.com/zenn-user-upload/d0e21054d0bf9ea23fd5aafe.png)




# おまけ
Twitterを目で見るのは疲れるのでTwitter読み上げソフトを探していたのですが，Discordにツイートを投げちゃえば，後は読み上げるBotを作るだけなので，[こちらの記事](https://qiita.com/Sashimimochi/items/aad6bcfa2d6096212d4c)を参考に実際にツイートを読み上げてもらうようにしました．
  
 
こちらはツイートではないですが，結構いい感じに読んでくれます．  
 

[実行動画](https://youtu.be/oMiyYVsAiI8)  




# おわりに
DiscordにTwitterタイムラインを実装できたけど，いいねもできないしリプライもできないのでTwitterとして利用するには別のBotを作る必要があります．  
 ただ実装はとても楽なのでTweetを音で読みたいだけの人にはちょうどいいのかなと思ったりはしました．  鍵垢のツイートははじかれていますが，プライバシー的に共用のサーバでタイムラインを垂れ流しにするのはよくない気がするので，使い方には気をつけてください．
 
# 参考文献
https://blog.hatena.ne.jp/login?blog=https%3A%2F%2Fkivantium.hateblo.jp%2Fentry%2F2020%2F04%2F15%2F235422


