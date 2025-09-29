############################################
1. SQL Firewallの準備
############################################

SQL Firewallを有効化します。環境はFREEPDB1です。

.. topic:: 実施内容

    + SQL Firewallの有効化


****************************
SQL Firewallを有効化する
****************************

ステータスを確認します。


.. code-block:: sql

    SQL> set markup csv on
    SQL> select * from DBA_SQL_FIREWALL_STATUS;
    "STATUS"  ,"STATUS_UPDATED_ON"                  ,"EXCLUDE_JOBS"
    "DISABLED","06-DEC-24 05.50.51.263690 AM +00:00","Y"

DISABLEDとなっており、無効化されていますので、次のコマンドで有効化します。

.. code-block:: sql
    
    EXEC DBMS_SQL_FIREWALL.ENABLE;

    -- 有効化されたことを確認
    SQL> select * from DBA_SQL_FIREWALL_STATUS;
    "STATUS" ,"STATUS_UPDATED_ON"                  ,"EXCLUDE_JOBS"
    "ENABLED","06-DEC-24 05.52.21.098865 AM +00:00","Y"


