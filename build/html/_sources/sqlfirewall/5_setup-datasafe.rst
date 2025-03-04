############################################
5. Data Safeの準備
############################################


**実施内容**

+ DATASAFE$ADMINユーザーを作成
+ OCIコンソール

.. datasafeの説明を入れる


basedbまたはADBの場合はDatasafeは基本無償でお使いいただけますが、コンピュート上に23aiがある場合は課金が発生しますのでご注意ください。
この手順ではOCI上にDB23aiがあるものとしますが、basedbも同様の手順でいけるはずです。

ADBの場合は少し手順が異なりますので、以下を参考にしてみてください

- https://qiita.com/western24/items/b772d95148b8855b8bb0
- https://docs.oracle.com/cd/E83857_01/paas/data-safe/admds/target-database-registration.html

****************************
DATASAFE$ADMINユーザーの作成
****************************
登録方法についてのサイトはこちら
https://docs.oracle.com/cd/E83857_01/paas/data-safe/admds/target-database-registration.html


DBにはdatasafe用のDBユーザーとして、DATASAFE_ADMINを作成。

別に名前は自由だが、コンソールではDATASAFE$ADMINユーザーがデフォルトでセットされてるので。

ADBにはDS$ADMINという名前ですでに作成されているので、作成する必要はない。
ロックされているのでアンロックする必要はある。
（ADBが登録解除されると再度ロックされる）


▼ DATASAFE_ADMINユーザーの作成例
    CREATE USER DATASAFE$ADMIN identified by password
    DEFAULT TABLESPACE "DATA"
    TEMPORARY TABLESPACE "TEMP";
GRANT CONNECT, RESOURCE TO DATASAFE_ADMIN;
:警告: SYSTEMまたはSYSAUXをデフォルト表領域として使用しないでください。 これらの表領域を使用している場合は、データをマスキングできません。


PDBで作成


CREATE USER DATASAFE$ADMIN identified by <password>
DEFAULT TABLESPACE "USERS"
TEMPORARY TABLESPACE "TEMP";

CREATE USER DATASAFE$ADMIN identified by Welcome1#Welcome1#
DEFAULT TABLESPACE "USERS"
TEMPORARY TABLESPACE "TEMP";


******************************************
DATASAFE$ADMINユーザーにロールを追加する
******************************************


ロールを追加する
ADBとno-ADBで追加するロールが違う
https://docs.oracle.com/cd/E83857_01/paas/data-safe/admds/grant-roles-oracle-data-safe-service-account-your-target-database.html
Autonomous Databaseでは、DS$DATA_MASKING_ROLEを除くすべてのロールがすでにデフォルトで付与されています。

.. image:: ../_static/datasafe/ロール.png


non-ADBの場合、datasafe_privileges.sqlを流すことで、ロールを付与、または取り消しができる。
このスクリプトはOCIコンソールのターゲットDBの登録画面から入手可能

.. image:: ../_static/datasafe/権限スクリプトのDL.png

.. [メニューバー] → [Oracle Database] → [データ・セーフ - データベース・セキュリティ]

DBサーバーにスクリプトファイルを移し、以下の感じで実行すると、
すべての権限を付与し、すべてのOracle Data Safe機能が使用可能になる。

@datasafe_privileges.sql <DATASAFEユーザー> GRANT ALL -VERBOSE


SQL> @datasafe_privileges.sql DATASAFE$ADMIN GRANT ALL -VERBOSE
Enter value for USERNAME (case sensitive matching the username from dba_users)
Setting USERNAME to DATASAFE$ADMIN
Enter value for TYPE (grant/revoke)
Setting TYPE to GRANT
Enter value for MODE (audit_collection/audit_setting/data_discovery/masking/assessment/sql_firewall/all)
Setting MODE to ALL

Granting AUDIT_COLLECTION privileges to "DATASAFE$ADMIN" ...
CREATE ROLE "ORA_DSCS_AUDIT_COLLECTION"
GRANT CREATE SESSION to "ORA_DSCS_AUDIT_COLLECTION"
GRANT AUDIT_VIEWER TO "ORA_DSCS_AUDIT_COLLECTION"
GRANT READ ON SYS.DBA_AUDIT_MGMT_CLEANUP_JOBS TO "ORA_DSCS_AUDIT_COLLECTION"
GRANT READ ON SYS.V_$PWFILE_USERS TO "ORA_DSCS_AUDIT_COLLECTION"
GRANT READ ON SYS.DBA_TABLES TO "ORA_DSCS_AUDIT_COLLECTION"
GRANT SELECT ON SYS.DUAL TO "ORA_DSCS_AUDIT_COLLECTION"
GRANT READ ON SYS.V_$OPTION TO "ORA_DSCS_AUDIT_COLLECTION"
GRANT EXECUTE ON SYS.DEFAULT_JOB_CLASS TO "ORA_DSCS_AUDIT_COLLECTION"
GRANT EXECUTE ON SYS.DBMS_OUTPUT TO "ORA_DSCS_AUDIT_COLLECTION"
GRANT READ ON SYS.STMT_AUDIT_OPTION_MAP TO "ORA_DSCS_AUDIT_COLLECTION"
GRANT EXECUTE ON SYS.XMLTYPE TO "ORA_DSCS_AUDIT_COLLECTION"
...
GRANT AUDIT_VIEWER TO "ORA_DSCS_ASSESSMENT"
GRANT CAPTURE_ADMIN TO "ORA_DSCS_ASSESSMENT"
GRANT SELECT ON AUDSYS.AUD$UNIFIED TO "ORA_DSCS_ASSESSMENT"
GRANT "ORA_DSCS_ASSESSMENT" to "DATASAFE$ADMIN"
Disconnected from Oracle Database 23ai Free Release 23.0.0.0.0 - Develop, Learn, and Run for Free
Version 23.6.0.24.10


******************************************
DATASAFEにDBを登録する
******************************************



プライベートエンドポイントの作成
================================


[データ・セーフ] → [設定] → [ターゲット・データベース] → [プライベート・エンドポイント]
からプライベートエンドポイントを作成します。

