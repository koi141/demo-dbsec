
監査ポリシーの一覧取得

.. code-block::

    SELECT policy_name, audit_condition, eval_name, enabled
    FROM   audit_unified_policies
    ORDER BY policy_name;


