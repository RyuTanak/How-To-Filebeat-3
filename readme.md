# Filebeatの使いかた(3章)  

[1章](https://github.com/RyuTanak/How-To-Filebeat-1)では、データ取得、データ送信についてやりました。  
[2章](https://github.com/RyuTanak/How-To-Filebeat-2)では、データ加工の方法について説明しました。  
3章では、もう少し複雑なログの分解をやってみたいと思います。  

## 目次  
[入力条件](#content1)  
[moduleの作成](#content2)  
[](#content3)  
[](#content4)  

<h2 id="content1">入力条件</h2>  

例えばWebサーバのログであれば、出力されるログは
Request、Responseだったり、サーバ自体にログインしたりと様々なログが出現する。  
このように1つの機器から複数種類のログが出力されることがあり、その場合でもFilebeatは対応することが可能である。  

対応方法は様々あるが、1つとして以下の方法を紹介する。  

入力となるログは以下とする
```  
2023-01-01 10:00:00 hp00001aa[12334] Response: 199.20.11.44 ms0000000 501 success 10ms  
2023-01-01 11:00:00 hp00001aa[543] Response: 199.20.11.44 ms0000001 404 fail 100ms error connect  
2023-01-01 12:00:00 hp00001aa[7890] login: 199.20.11.43 rtanaka OK
2023-01-02 10:00:00 hp00001ab[111111] Request: 199.20.12.43 ms0000001 503 success 12 https://github.com
2023-01-02 11:00:00 hp00001ab[232] Request: 199.20.12.21 ms0000003 501 success 90ms https://github.com/Ryu.tanak
2023-01-02 12:00:00 hp00001ab[3543] login: 199.20.12.11 rtanaka NG Password faiid
2023-01-03 10:00:00 hp00001ac[23123] Response: 199.20.13.34 ms0000002 501 success 80ms
2023-01-03 11:00:00 hp00001ac[647676] Request: 199.20.13.44 ms0000004 501 success 70ms https://github.com
2023-01-03 12:00:00 hp00001ac[454777] Response: 199.20.13.54 ms0000005 503 success 60ms  
```

登録先のindexを以下のように定義する。  
index名：webServer_index  
|@timestamp|host_name|process_id|log_type|dstip|module_name|status_code|status|access_time|message|  
|-|-|-|-|-|-|-|-|-|-|  
|2023-01-01 10:00:00|hp00001aa|12334|Response|199.20.11.44|ms0000000|501|success|10|-|
|2023-01-01 11:00:00|hp00001aa|543|Response|199.20.11.44|ms0000001|404|fail|100|error connect|
|2023-01-03 10:00:00|hp00001ac|23123|Response|199.20.13.34|ms0000002|501|success|80|-|
|2023-01-03 12:00:00|hp00001ac|454777|Response|199.20.13.54|ms0000005|503|success|60|-|  

※statusがsuccessの場合、messageは空  

|@timestamp|host_name|process_id|log_type|srcip|module_name|status_code|status|access_time|message|  
|-|-|-|-|-|-|-|-|-|-|  
|2023-01-02 10:00:00|hp00001ab|111111|Request|199.20.12.43|ms0000001|503|success|12|https://github.com|
|2023-01-02 11:00:00|hp00001ab|232|Request|199.20.12.21|ms0000003|501|success|90|https://github.com/Ryu.tanak|
|2023-01-03 11:00:00|hp00001ac|647676|Request|199.20.13.44|ms0000004|501|success|70|https://github.com|

|@timestamp|host_name|process_id|log_type|srcip|login_name|result|message|
|-|-|-|-|-|-|-|-|
|2023-01-01 12:00:00|hp00001aa|7890|login|199.20.11.43|rtanaka|OK|-|
|2023-01-02 12:00:00|hp00001ab|3543|login|199.20.12.11|rtanaka|NG|Password faiid|

※resultがOKの場合、messageは空  

上記のように、1つのindexに複数種類あるWebサーバのログを登録する。  

<h2 id="content2">moduleの作成</h2>  

moduleを用意することで、1つのFilebeatで複数機器のログ取集が行えます。  
（今回はWebサーバの1機器であるが、紹介としてmoduleを作成する。）  

フォルダ構成は以下のようになります。  

├etc  
│└filebeat  
│　├filebeat.yml  
│　└modules.d  
│　　├webserver.yml　←追加  
│　　├・・・  
├usr  
│└share  
│　└filebeat  
│　　└module  
│　　　└webserver　←ここから下全部追加  
│　　　　└log  
│　　　　　├config  
│　　　　　│└log.yml  
│　　　　　├ingest  
│　　　　　│└pipeline.yml  
│　　　　　└manifest.yml  

