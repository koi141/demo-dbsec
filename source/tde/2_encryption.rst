###########################
2. 表領域の暗号化を行う
###########################

.. topic:: 実施内容

    + 表領域に対して暗号化を行い、データファイルを暗号化
    + OSコマンドでデータファイルを開き、データが暗号化されていることを確認する

****************************
データファイルの確認
****************************

作成したHRスキーマを使用します。手順ではUSERS表領域に作成していますが、念のため実際に確認を行います。

.. code-block:: sql
   :caption: FREEPDB1で実行 (SYSユーザー)

    -- 結果をcsv形式で出力
    SQL> set markup csv on

    -- HRスキーマの表領域がUSERS表領域であることを確認
    SQL> select username, default_tablespace from dba_users where username ='HR';
    "USERNAME","DEFAULT_TABLESPACE"
    "HR"      ,"USERS"

    -- USER表領域が格納されるデータファイルのパスを確認
    SQL> select tablespace_name, file_name from dba_data_files where tablespace_name = 'USERS';
    "TABLESPACE_NAME","FILE_NAME"
    "USERS"          ,"/opt/oracle/oradata/FREE/FREEPDB1/users01.dbf"


OSコマンドでUSERS表領域のデータファイルの中身を確認します。

.. code-block:: sql

    SQL> !strings /opt/oracle/oradata/FREE/FREEPDB1/users01.dbf
    ...
    CJOHNSON
    44.1632.960034
    SA_REP
    O       Kimberely
    Grant
    KGRANT
    44.1632.960033
    SA_REP
    Jack
    Livingston
    JLIVINGS
    44.1632.960032
    ...


このように、TDEで暗号化されていない場合、stringsコマンドでデータの内容が確認できてしまうことが分かります。



****************************
表領域の暗号化を行う
****************************

次に、HRスキーマが格納されるUSERS表領域を暗号化します。

.. code-block:: sql
    :caption: FREEPDB1で実行 (SYSユーザー)

    -- 実行時間を計測
    SQL> set timing on

    -- USERS表領域を暗号化
    SQL> alter tablespace users encryption online using 'AES256' encrypt;
    
    Tablespace altered.

    Elapsed: 00:00:07.02

暗号化が完了しましたので、先ほどと同様のOSコマンドでデータファイルの中身を確認します。

.. code-block:: sql

    SQL> !strings /opt/oracle/oradata/FREE/FREEPDB1/users01.dbf
    ...
    mK=.
    qM$;
    /eUr
    N5Y9N
    ZSaKE
    y!Ac
    oH/P
    <ar3:oii
    /`_S
    }l d%
    ...

このように、データファイルが暗号化され、内容が解読できない形式に変わったことを確認できます。  
また、SQLクエリが問題なく実行されることも確認します。

.. code-block:: sql

    SQL> select * from hr.jobs;
    "JOB_ID","JOB_TITLE","MIN_SALARY","MAX_SALARY"
    "AD_PRES","President",20080,40000
    "AD_VP","Administration Vice President",15000,30000
    "AD_ASST","Administration Assistant",3000,6000
    ...
    "SH_CLERK","Shipping Clerk",2500,5500
    "IT_PROG","Programmer",4000,10000
    "MK_MAN","Marketing Manager",9000,15000
    "MK_REP","Marketing Representative",4000,9000
    "HR_REP","Human Resources Representative",4000,9000
    "PR_REP","Public Relations Representative",4500,10500

    19 rows selected.

    Elapsed: 00:00:00.02



****************************
暗号化された表領域を復号する
****************************

オンラインで暗号化を行いましたが、同様にオンラインで復号も行うことができます。

.. code-block:: sql
    :caption: FREEPDB1で実行 (SYSユーザー)

    -- USERS表領域の復号を行う
    SQL> alter tablespace users encryption online decrypt;

    -- 復号されていることを確認する
    SQL> !strings /opt/oracle/oradata/FREE/FREEPDB1/users01.dbf
    ...
    Geneve
    Rua Frei Caneca 1360    01307-002       Sao Paulo       Sao Paulo
    Schwanthalerstr. 7031
    80925
    Munich
    Bavaria
    9702 Chester Road
    09629850293     Stretford
    Manchester
    (Magdalen Centre, The Oxford Science Park


| 復号後、データファイルの内容が再び人間が読める形式で表示されることが確認できます。  
| また他の端末にて、オンライン暗号化または復号処理を実行中に ``ls`` コマンドを実行することで、暗号化・復号処理中のファイルの状況を確認することができます。

.. code-block:: bash

    $ ls -l
    total 1077780
    -rw-r-----. 1 oracle oinstall 597696512 Nov 28 14:09 sysaux01.dbf
    -rw-r-----. 1 oracle oinstall 314580992 Nov 28 14:08 system01.dbf
    -rw-r-----. 1 oracle oinstall  20979712 Nov 27 22:00 temp01.dbf
    -rw-r-----. 1 oracle oinstall 104865792 Nov 28 14:09 undotbs01.dbf
    -rw-r-----. 1 oracle oinstall  75505664 Nov 28 14:10 users01.dbf
    -rw-r-----. 1 oracle oinstall  75505664 Nov 28 14:10 users01.dbf_new

| 以上の結果より ``users01.dbf_new`` という同じサイズの新しいデータファイルが作成されていることが分かります。  
| このようにオンライン処理では元のデータファイルと同じサイズの新しいファイルが作成されます。そのため、オンライン暗号化を行う際には、対象データファイルと同じサイズの空き容量を確保しておく必要があります。