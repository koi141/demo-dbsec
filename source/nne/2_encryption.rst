###############################
2. 通信の暗号化を確認する
###############################


.. topic:: 実施内容

    + sqlnet.oraファイルを編集し、ネイティブ・ネットワーク暗号化を有効化する
    + Wiresharkでパケットが暗号化されているかを確認する


************************************
DBサーバー側で通信を待ち受ける
************************************

| まず、DBサーバー側でポート1521を開放し、通信を受け付ける準備をします。
| 次のコマンドで、ファイアウォールを設定してポート1521を解放します。

.. code-block:: bash

    # ポート1521を解放
    sudo firewall-cmd --permanent --add-port=1521/tcp
    sudo firewall-cmd --reload

    # インターフェース名と1521番ポートが解放されていることを確認
    $ sudo firewall-cmd --list-all
    public (active)
        target: default
        icmp-block-inversion: no
        interfaces: enp0s5
        sources:
        services: dhcpv6-client ssh
        ports: 1521/tcp
        protocols:
        forward: no
        masquerade: no
        forward-ports:
        source-ports:
        icmp-blocks:
        rich rules:

| 次に、tsharkコマンドを使ってネットワークインターフェースを確認し、パケットキャプチャを開始します。
| 先ほど確認したインターフェース名、enp0s5 を使用してTCPポート1521で通信をキャプチャします。

.. code-block:: bash

    # インターフェースの番号を確認
    $ sudo tshark -D
    Running as user "root" and group "root". This could be dangerous.
    1. enp0s5
    2. lo (Loopback)
    3. any
    4. bluetooth-monitor
    5. nflog
    6. nfqueue
    7. usbmon0
    8. usbmon1
    9. usbmon2
    10. usbmon3
    ...

    # TCP/1521で受け取る通信をキャプチャ
    $ sudo tshark -i 1 -Y 'tcp.port==1521' -x


これで、DBサーバー側での通信がキャプチャされるようになります。次に、クライアント側でDBに接続します。



************************************
クライアント側でDBに接続する
************************************

簡易接続子を使って、SQL*PlusでOracleデータベースに接続します。
::

    sqlplus hr/<パスポート>@<接続先ホスト名>:<ポート番号>/<サービス名>


以下のコマンドで、HRユーザーとして接続します。実行例ではパスワードを省略することで証跡にパスワードが残らないように接続しています。

.. code:: bash

    $ sqlplus hr@192.168.130.169:1521/freepdb1

    SQL*Plus: Release 21.0.0.0.0 - Production on Tue Nov 26 22:09:52 2024
    Version 21.11.0.0.0

    Copyright (c) 1982, 2022, Oracle.  All rights reserved.

    Enter password: <パスワードを入力>
    Last Successful login time: Tue Nov 26 2024 22:08:58 +09:00

    Connected to:
    Oracle Database 23ai Free Release 23.0.0.0.0 - Develop, Learn, and Run for Free
    Version 23.6.0.24.10

    -- 適当なSQLを発行
    SQL> select * from jobs;

    JOB_ID     JOB_TITLE                           MIN_SALARY MAX_SALARY
    ---------- ----------------------------------- ---------- ----------
    AD_PRES    President                                20080      40000
    AD_VP      Administration Vice President            15000      30000
    AD_ASST    Administration Assistant                  3000     6000
    FI_MGR     Finance Manager                           8200      16000
    FI_ACCOUNT Accountant                                4200     9000
    AC_MGR     Accounting Manager                        8200      16000
    ...


この時点で、DBサーバーでパケットキャプチャを実行している端末を見ると、jobs テーブルの内容が平文で送信されていることが確認できます。

.. code:: text

    （一部抜粋）
    ...
    0080  2c 01 04 05 41 44 5f 56 50 1d 41 64 6d 69 6e 69   ,...AD_VP.Admini
    0090  73 74 72 61 74 69 6f 6e 20 56 69 63 65 20 50 72   stration Vice Pr
    00a0  65 73 69 64 65 6e 74 03 c3 02 33 02 c3 04 07 2a   esident...3....*
    00b0  2c 01 04 07 41 44 5f 41 53 53 54 18 41 64 6d 69   ,...AD_ASST.Admi
    00c0  6e 69 73 74 72 61 74 69 6f 6e 20 41 73 73 69 73   nistration Assis
    00d0  74 61 6e 74 02 c2 1f 02 c2 3d 07 21 2c 01 04 06   tant.....=.!,...
    00e0  46 49 5f 4d 47 52 0f 46 69 6e 61 6e 63 65 20 4d   FI_MGR.Finance M
    00f0  61 6e 61 67 65 72 02 c2 53 03 c3 02 3d 07 1f 2c   anager..S...=..,
    0100  01 04 0a 46 49 5f 41 43 43 4f 55 4e 54 0a 41 63   ...FI_ACCOUNT.Ac
    0110  63 6f 75 6e 74 61 6e 74 02 c2 2b 02 c2 5b 07 24   countant..+..[.$
    0120  2c 01 04 06 41 43 5f 4d 47 52 12 41 63 63 6f 75   ,...AC_MGR.Accou
    0130  6e 74 69 6e 67 20 4d 61 6e 61 67 65 72 02 c2 53   nting Manager..S
    0140  03 c3 02 3d 07 26 2c 01 04 0a 41 43 5f 41 43 43   ...=.&,...AC_ACC
    0150  4f 55 4e 54 11 50 75 62 6c 69 63 20 41 63 63 6f   OUNT.Public Acco
    0160  75 6e 74 61 6e 74 02 c2 2b 02 c2 5b 07 20 2c 01   untant..+..[. ,.
    0170  04 06 53 41 5f 4d 41 4e 0d 53 61 6c 65 73 20 4d   ..SA_MAN.Sales M
    0180  61 6e 61 67 65 72 02 c3 02 04 c3 03 01 51 07 27   anager.......Q.'
    0190  2c 01 04 06 53 41 5f 52 45 50 14 53 61 6c 65 73   ,...SA_REP.Sales
    01a0  20 52 65 70 72 65 73 65 6e 74 61 74 69 76 65 02    Representative.
    01b0  c2 3d 04 c3 02 15 09 07 24 2c 01 04 06 50 55 5f   .=......$,...PU_
    01c0  4d 41 4e 12 50 75 72 63 68 61 73 69 6e 67 20 4d   MAN.Purchasing M
    01d0  61 6e 61 67 65 72 02 c2 51 03 c3 02 33 07 23 2c   anager..Q...3.#,
    01e0  01 04 08 50 55 5f 43 4c 45 52 4b 10 50 75 72 63   ...PU_CLERK.Purc
    01f0  68 61 73 69 6e 67 20 43 6c 65 72 6b 02 c2 1a 02   hasing Clerk....
    0200  c2 38 07 1e 2c 01 04 06 53 54 5f 4d 41 4e 0d 53   .8..,...ST_MAN.S
    ...


| この状態では、データが平文でネットワーク上に送信されており、暗号化が行われていないことが確認できます。
| それでは、次の手順で通信の暗号化を設定します。

************************************
通信の暗号化設定を行う
************************************

DBサーバーの ``$ORACLE_HOME/network/admin`` にある ``sqlnet.ora`` ファイルを編集し、通信の暗号化を有効化します。


まず、 ``sqlnet.ora`` ファイルの場所を確認します。


.. code:: bash

    $ ls $ORACLE_HOME/network/admin
    listener.ora  samples  shrept.lst  sqlnet.ora  tnsnames.ora

次に、 ``sqlnet.ora`` を編集して、暗号化を有効にします。

.. code-block:: bash
    :emphasize-lines: 5,6

    $ vi $ORACLE_HOME/network/admin/sqlnet.ora
    NAMES.DIRECTORY_PATH= (TNSNAMES, EZCONNECT)

    # 以下の2行を追加
    SQLNET.ENCRYPTION_SERVER = REQUIRED
    SQLNET.ENCRYPTION_TYPES_SERVER = (AES256, AES192, AES128)                                    

この設定により、データベースとの通信が暗号化されるようになります。

sudo rpm -ivh oracle.mgmt_agent.241023.2127.Linux-x86_64.rpm --preserve-env=JAVA_HOME

************************************
暗号化設定後、再びDBに接続する
************************************

クライアント側で再びDBに接続し、先ほどと同様に問い合わせを行います。

.. code:: bash

    $ sqlplus hr@192.168.130.169:1521/freepdb1

    SQL*Plus: Release 21.0.0.0.0 - Production on Tue Nov 26 22:09:52 2024
    Version 21.11.0.0.0

    Copyright (c) 1982, 2022, Oracle.  All rights reserved.

    Enter password: <パスワードを入力>
    Last Successful login time: Tue Nov 26 2024 22:08:58 +09:00

    Connected to:
    Oracle Database 23ai Free Release 23.0.0.0.0 - Develop, Learn, and Run for Free
    Version 23.6.0.24.10

    -- 適当なSQLを発行
    SQL> select * from jobs;

    JOB_ID     JOB_TITLE                           MIN_SALARY MAX_SALARY
    ---------- ----------------------------------- ---------- ----------
    AD_PRES    President                                20080      40000
    AD_VP      Administration Vice President            15000      30000
    AD_ASST    Administration Assistant                  3000     6000
    FI_MGR     Finance Manager                           8200      16000
    FI_ACCOUNT Accountant                                4200     9000
    AC_MGR     Accounting Manager                        8200      16000
    ...

Wiresharkでのキャプチャを確認すると、データが平文ではなく、暗号化されていることが確認できます。

.. code:: text

    （一部抜粋）
    ...
    0140  41 d7 65 ec 53 50 c4 46 10 a4 08 e4 30 9b 68 7f   A.e.SP.F....0.h.
    0150  2c 19 28 2c 5d 58 09 f4 7a c6 16 5f 8e 6a 42 1d   ,.(,]X..z.._.jB.
    0160  e0 7a 69 f9 eb 86 96 44 36 5e c6 81 e8 cb 48 61   .zi....D6^....Ha
    0170  44 e1 02 61 ca 47 ce d0 58 df 2c be 3b cb 02 36   D..a.G..X.,.;..6
    0180  05 a8 1f 4f e8 d9 be 71 da ac 67 36 e0 65 9d 4b   ...O...q..g6.e.K
    0190  e3 8f ef d3 5f 30 fa 74 24 fa 6c b8 5e 87 71 c1   ...._0.t$.l.^.q.
    01a0  67 af 2e e5 b3 e4 4a e8 c7 c6 dc 61 11 22 b4 69   g.....J....a.".i
    01b0  68 42 f3 e9 da 80 ad 5e d6 7b 75 f3 10 53 04 77   hB.....^.{u..S.w
    01c0  8c 62 68 2c e8 ab 9b 4e 6f 54 4b ec fa fb 5a ed   .bh,...NoTK...Z.
    01d0  61 63 97 27 74 a5 d5 1f ce 54 b8 56 d8 a3 c0 e1   ac.'t....T.V....
    01e0  a4 1e ca 5f 39 0c 1e ec 3e 23 57 a1 a4 a2 c6 2b   ..._9...>#W....+
    01f0  c1 27 04 07 4f 0e 10 ac 13 ff b0 ae 31 48 a8 26   .'..O.......1H.&
    0200  df 6e 68 dc fd 17 36 db 4d 07 ad 11 af e3 ff 80   .nh...6.M.......
    0210  d1 5e cf ab 61 f9 57 36 6f 98 52 d0 49 29 13 57   .^..a.W6o.R.I).W
    0220  8d 21 32 1d 40 cb d0 a2 96 70 db 13 ae 35 0d ec   .!2.@....p...5..
    ...

このように通信が確実に暗号化されていることが確認できます。  

以上で、ネイティブ・ネットワーク暗号化のデモは終了です。