#######################################
1. ネイティブ・ネットワーク暗号化の準備
#######################################

.. topic:: 実施内容
    
    + DBクライアントの用意
    + wiresharkのインストール

*******************************
DBクライアントの用意
*******************************

クライアントとデータベース間の通信を確認するために、クライアントソフトが必要です。

Oracleデータベースに接続するためのクライアントソフトとしては、主に以下の2つがあります

+ SQL Developer
+ SQL*Plus

いづれかのツールを用いてFREEPDB1に接続できる環境をご準備ください。  
本手順ではSQL*Plusを使用しますが、SQL Developerもご利用いただけます。  
また、SQL Developerは、VS Codeの拡張機能としても提供されていますので、ぜひご活用ください。

サービス名など、接続のための情報はDBサーバーで ``lsnrctl status`` コマンドを実行することで確認できます。

.. code:: bash

    $ lsnrctl status

    LSNRCTL for Linux: Version 23.0.0.0.0 - for Oracle Cloud and Engineered Systems on 28-NOV-2024 14:54:00

    Copyright (c) 1991, 2024, Oracle.  All rights reserved.

    Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=xxx.xxx.xxx.xxx)(PORT=1521)))
    STATUS of the LISTENER
    ------------------------
    Alias                     LISTENER
    Version                   TNSLSNR for Linux: Version 23.0.0.0.0 - for Oracle Cloud and Engineered Systems
    ...
    Instance "FREE", status READY, has 1 handler(s) for this service...
    Service "FREE" has 1 instance(s).
    Instance "FREE", status READY, has 1 handler(s) for this service...
    Service "FREEXDB" has 1 instance(s).
    Instance "FREE", status READY, has 1 handler(s) for this service...
    Service "freepdb1" has 1 instance(s).
    Instance "FREE", status READY, has 1 handler(s) for this service...
    The command completed successfully

ここで、サービス名や接続先ホスト、ポート番号などの情報が確認できます。  
この情報をもとに、クライアントソフトからDBに接続してください。


*******************************
wiresharkのインストール
*******************************

DBサーバーで受信した通信をキャプチャし、暗号化が行われているかどうかを確認します。

パケットキャプチャにはWiresharkを使用するため、DBサーバーにインストールします。

.. code:: bash

    $ sudo dnf -y install wireshark

    -- インストールを確認
    $ tshark --version
    TShark (Wireshark) 2.6.2 (v2.6.2)

    Copyright 1998-2018 Gerald Combs <gerald@wireshark.org> and contributors.
    License GPLv2+: GNU GPL version 2 or later <http://www.gnu.org/licenses/old-licenses/gpl-2.0.html>
    This is free software; see the source for copying conditions. There is NO
    warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
    ...


これで、Wiresharkのインストールが完了し、通信のキャプチャ準備が整いました。



