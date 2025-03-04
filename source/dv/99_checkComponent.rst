############################################
（おまけ）各コンポーネントの確認クエリ
############################################


各コンポーネントを確認するコマンドを示します。

ルールやルール・セット、レルム等多くのコンポーネントが出てきますので、自身で作成したものの確認や、削除忘れがないかなどの確認にご活用下さい。

参照するビューについての詳細はこちらも参考にしてください
    https://docs.oracle.com/cd/G11854_01/dvadm/oracle-database-vault-data-dictionary-views.html



********************************
レルム
********************************

関連するビューは以下の通りです。

:``DBA_DV_REALMビュー``: レルム一覧  
:``DBA_DV_REALM_AUTHビュー``: レルム内にアクセスできるユーザー一覧  
:``DBA_DV_REALM_OBJECTビュー``: レルム内にて保護されているオブジェクト一覧  

特定のレルムの結果に絞りたい場合、 WHERE句でを付けて使用してください。


.. code-block:: sql
    :caption: レルム一覧を出す
    
    SELECT NAME, ENABLED, COMMON FROM DBA_DV_REALM ORDER BY NAME;



.. code-block:: sql
    :caption: レルムにアクセスできるユーザーを出す

    SELECT REALM_NAME, GRANTEE, AUTH_RULE_SET_NAME FROM DBA_DV_REALM_AUTH ORDER BY REALM_NAME;

AUTH_RULE_SET_NAME列: 認可の際にチェックされるルール・セット名



.. code-block:: sql
    :caption: レルム内にあるオブジェクト一覧を出す

    SELECT REALM_NAME, OWNER, OBJECT_NAME, COMMON_REALM FROM DBA_DV_REALM_OBJECT ORDER BY REALM_NAME;

COMMON_REALM列: レルムが共通かローカルかを示す。

+ 共通: ``YES``
+ ローカル: ``NO``


********************************
ルール・セット
********************************

以下のビューを使います。
:``DBA_DV_RULE_SETビュー``: 作成済みのルール・セット一覧
:``DBA_DV_RULE_SET_RULE``: ルールセットに関連付けられているルール一覧

.. code-block:: sql
    :caption: 作成済みのルール・セット一覧を出す

    SELECT RULE_SET_NAME, DESCRIPTION, ENABLED, FAIL_MESSAGE FROM DBA_DV_RULE_SET;



.. code-block:: sql
    :caption: ルールセットに関連付けられているルールを確認する
    
    SELECT RULE_SET_NAME, RULE_NAME, RULE_EXPR, ENABLED FROM DBA_DV_RULE_SET_RULE;



********************************
ルール
********************************
``DBA_DV_RULEビュー`` を使用します。


.. code-block:: sql
    :caption: 定義済みのルール一覧を出す

    SELECT NAME, RULE_EXPR FROM DBA_DV_RULE OEDER BY NAME;




********************************
コマンド・ルール
********************************
``DBA_DV_COMMAND_RULEビュー`` を使用します。

.. code-block:: sql
    :caption: 作成済みのルール・セット一覧を出す
    
    SELECT COMMAND, RULE_SET_NAME FROM DBA_DV_COMMAND_RULE;

