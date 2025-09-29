############################################
3. Database Vaultの動作を確認する
############################################

レルムのアクセス制御、SYSユーザーの制限、そしてIPアドレス制限を使用したアクセス制御の動作を確認します。

.. topic:: 実施内容

    + SYSユーザーのアクセス制限確認
    + HRユーザーおよびSALES_APPユーザーのアクセス確認
    + APPユーザーのIPアドレス制限確認



********************************
SYSユーザーのアクセス確認
********************************

Database Vaultが有効化された環境では、SYSユーザーはユーザーを作成できません。
この操作は、C##DVACCTMGR(アカウント管理者)ユーザーに委任されます。

.. code-block:: sql

    SQL> create user test;
    create user test
    *
    ERROR at line 1:
    ORA-01031: insufficient privileges
    Help: https://docs.oracle.com/error-help/db/ora-01031/


レルム内のオブジェクトにアクセスできないことを確認します。

.. code-block:: sql

    SQL> select * from hr.regions;
    select * from hr.regions
                    *
    ERROR at line 1:
    ORA-01031: insufficient privileges
    Help: https://docs.oracle.com/error-help/db/ora-01031/


******************************************************
HRユーザーおよびSALES_APPユーザーのアクセス確認
******************************************************
レルム認可を行ったHRユーザーまたはSALES_APPユーザーからは、SYSユーザーではアクセスできなかったREGIONS表にアクセスできることを確認します。

.. code-block:: sql
    :caption: HRユーザーまたはSALES_APPユーザー

    SQL> select * from hr.regions;

    REGION_ID REGION_NAME
    ---------- -------------------------
            10 Europe
            20 Americas
            30 Asia
            40 Oceania
            50 Africa


********************************
APPユーザー
********************************

APPユーザーにはIPアドレスによる制限付きで認可が付与されています。
この設定に基づき、許可されたIPアドレスからのみアクセス可能であることを確認します。

許可されたIPアドレスからのアクセスの場合
==============================================

.. code-block:: sql
    :caption: APPユーザー

    SQL> set markup csv on
    SQL> select SYS_CONTEXT('USERENV','IP_ADDRESS');
    "SYS_CONTEXT('USERENV','IP_ADDRESS')"
    "xxx.xxx.xxx.xxx"

    SQL> select * from hr.regions;
    "REGION_ID","REGION_NAME"
    10,"Europe"
    20,"Americas"
    30,"Asia"
    40,"Oceania"
    50,"Africa"


許可されていないIPアドレスからのアクセスの場合
==============================================

.. code-block:: sql
    :caption: APPユーザー

    SQL> set markup csv on
    SQL> select SYS_CONTEXT('USERENV','IP_ADDRESS');
    "SYS_CONTEXT('USERENV','IP_ADDRESS')"
    "yyy.yyy.yyy.yyy"

    SQL> select * from hr.regions;
    select * from hr.regions
                    *
    ERROR at line 1:
    ORA-47306: 20000: DV_Error: Can only be accessed from a specific IP address
    Help: https://docs.oracle.com/error-help/db/ora-47306/

エラーメッセージに、レルム認可時に設定したカスタムエラーメッセージが表示されていることも分かります。

