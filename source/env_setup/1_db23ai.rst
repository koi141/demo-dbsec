##########################################
1. Oracle Database 23ai FREEの準備
##########################################

.. topic:: 実施内容

    + DB23ai FREEをインストールし、DBを作成する

*******************************************
DB23ai にてご利用いただける機能について
*******************************************
Oracleが提供する新しい無償版バージョン「23ai FREE」では従来のXE（Express Edition）と比較して、セキュリティ機能が多く解放されており、気軽に試しやすくなりました。 
TDE や Database Vault など、従来の無償版XEでは使用できなかった機能が使えるようになっていますので、ぜひ、お試しください。

使用可能な機能の一部は以下の通りです。
+ TDE
+ Data Redaction
+ Oracle Label Security
+ Virtual Private Vault
+ Database Vault
+ SQL Firewall
+ ファイングレイン監査

その他FREEで使用できる機能は `こちら <https://apex.oracle.com/database-features/>`__ からもご確認いただけます。


***********************************
DB23ai を使用できる環境について
***********************************

このサイトで紹介する手順の環境は主に無償版である23ai FREEを使用します。
23ai は以下の環境にてご使用いただけます。（2025年1月時点）

**Oracle Cloud環境**
    + Base Database Service
    + Exadata Database Service
    + Autonomous Database

**オンプレミス環境**
    + FREE(無償版) 
    + Oracle Database Appliance
    + Exadata Database Appliance

> ダウンロードサイト： https://www.oracle.com/jp/database/free/get-started/


******************************
DB23ai FREE の準備
******************************

FREEでは以下の3種類で提供しています。

+ コンテナ（docker, podman）
+ VMイメージ（VM VirtualBox）
+ rpmパッケージ（Oracle Linux 8,9, Windows）

各インストール手順については、 `こちらのドキュメント <https://docs.oracle.com/cd/G11854_01/xeinl/index.html>`__ に記載がありますので、お好みの方法で環境を構築してください。

rpmでインストールする場合、 `OCIチュートリアル <https://oracle-japan.github.io/ocitutorials/ai-vector-search/ai-vector102-23aifree-install>`__ でもインストール手順が紹介されていますので、ぜひ参考にしてみてください。OCIチュートリアルでの手順では表領域の作成手順は不要ですので、その前の手順までを実施してください。


.. admonition:: FREEのライセンス制限

    | FREEでは処理に使用されるCPUは2コアに自動的に制限されます。
    | RAMは最大2GBまでとなっており、使用可能な場合でも超過することはできません。
    | データの使用料は最大12GBであり、これを超えて増加すると「ORA-12954: リクエストが、最大許容データベース・サイズの12GBを超えています」というエラーになります。
    | 参考： `ライセンス制限 <https://docs.oracle.com/cd/G11854_01/xeinl/licensing-restrictions.html#GUID-A3BF7927-EC58-40FC-96B6-1A5E135D19BA>`__


******************************
関連リンク
******************************
+ `Oracle Database 23ai ドキュメント（日本語） <https://docs.oracle.com/cd/G11854_01/books.html>`__
+ `Oracle Database 23ai Overview 戦略とロードマップ - Speaker Deck <https://speakerdeck.com/oracle4engineer/oracle-database-23ai-overview>`__
+ `Oracle Database Freeインストレーション・ガイド, 23ai for Linux（日本語） <https://docs.oracle.com/cd/G11854_01/xeinl/index.html>`__