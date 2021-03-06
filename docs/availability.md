---
title: Confluent関連 冗長性について 
summary: 冗長化について記載 
authors:
  - A.Tomita
date: 2019-06-10
---

# Confluentの冗長化機能について

Confluentは、コンポーネントごとに冗長化方法が異なります。

## zookeeperの冗長化

zookeeperは、3重化、5重化をサポートしていますが、
冗長台数が偶数となる場合（2台構成のHAを含む）の構成は組むことができません。
通常は、3台構成で冗長構成をとります。

過半数以上のサーバが動作している場合は、サービスを提供しますが、
過半数を下回る場合は、サービスを停止して復旧を待ちます。  
要は、3台冗長の場合、1台故障までシステム継続、2台故障で停止 
5台冗長の場合は、2台故障までシステム継続、3台故障で停止

!!! note 備考
    3台構成の場合、2台HAより稼働率が劣ります。詳細は稼働率計算を参照願います。

複数台故障の場合でも、保存されたデータは維持されるため、
全数障害にならない限り、データ損失が起きることはありません。


!!! note 備考
    データの保全性は、3台構成の方が冗長性が高くなります。



## zookeeper の稼働率計算

Zookeeperを冗長化した時の稼働率を計算するための式を以下に記載します。
\(x\)は、サーバ1台当たりの稼働率を入力します

###3重化

システムが稼働する確率  
\( 1-(1-x)^2*(1+2x) \)

データ破損が起きない確率  
\( 1-(1-x)^3 \)

### 5重化
システムが稼働する確率  
\(1-(1-x)^3*(5x^2+3x+1)\)

データ破損が起きない確率   
\(1-(1-x)^5\)

!!! note "備考"
    2重化のHA構成では、システム停止、データ破損共に\(1-(1-x)^2\)となります。
    この値は、3重化した時よりもシステム停止は起きにくいが、データ破損は
    起きやすいことになります。

例: 稼働率99%のサーバで構成した場合

|方式|システム稼働率|データ保全率|
|---|---|---|
|2重化|99.99%|99.99%|
|3重化|99.9702%|99.9999%|
|5重化|99.99911295%|99.99999999%|

* ハードウェア障害時の論理値です。不具合やその他の要因は考慮されていません。

## Kafka brokerの冗長化

Kafka brokerは3重化で冗長化を行います。
Zookeeperとは異なり、4台構成や5台構成なども組むことができますが
パーティション毎に3台が選択され利用します。

3台構成の場合
* 1台故障では、システム稼働を継続します。
* 2台故障で、システム稼働を停止します。

4台以上の構成の場合
* 故障した場合、故障したBroker内のパーティション情報を
  自動的に他のBrokerで再構築をしてシステム稼働を継続します。
* 稼働中のBrokerが2台になった場合は、縮退運用で稼働を継続します。

### 3台構成 VS 4台以上のクラスタ

3台ごとにクラスタを組む場合

メリット
* 障害時に縮退運用になるため、動作がわかりやすい

デメリット
* 縮退時に更なる障害があった場合は 動作停止を引き起こす
* コンシューマ、プロデューサーごとにクラスタを意識しないといけないため
  プログラムの管理が煩雑になる

4台以上の大きなクラスタを組む場合

メリット
* 障害時に自動的に再冗長化するため、安定性が高い

デメリット
* 障害時に再冗長化するためにパフォーマンス低下が発生する
* 障害復旧時に新たにトピックを作成するまで、新規ノードが使われることはない

!!! error "未検証事項"
    brokerクラスタは、zookeeperのルートパスを変えることで、複数の
    クラスタを組むことができます
    設定方法は server.propertiesの`zookeeper.connect`にpathを加えるだけになります。
