##########################################
3. Data Redactionの注意点
##########################################

| Data Redactionは、データの一部を隠すために使用される機能であるため、データへのアクセスを制御する機能ではないことを理解しておく必要があります。
| Data Redactionを適用しても、特定の条件下で推測が可能になる場合があるため、慎重な運用が求められます。


*****************************************
WHERE句での推測例
*****************************************

以下は、リダクション対象の ``salary`` 列に対してwhere句を用いた例です。


.. code-block:: sql
    :caption: HRユーザーがクエリを実行した場合:

    SQL> show user
    USER is "HR"
    SQL> set pages 500
    SQL> select first_name, salary, commission_pct from employees where salary > 10000;

    FIRST_NAME               SALARY COMMISSION_PCT
    -------------------- ---------- --------------
    Steven                    24000
    Neena                     17000
    Lex                       17000
    Nancy                     12008
    Den                       11000
    John                      14000             .4
    Karen                     13500             .3
    Alberto                   12000             .3
    Gerald                    11000             .3
    Eleni                     10500             .2
    Clara                     10500            .25
    Lisa                      11500            .25
    Ellen                     11000             .3
    Michael                   13000
    Shelley                   12008

    15 rows selected.


.. code-block:: sql
    :caption: SALES_APPユーザーが同じクエリを実行した場合:

    SQL> show user
    USER is "SALES_APP"
    SQL> set pages 500
    SQL> select first_name, salary, commission_pct from hr.employees where salary > 10000;

    FIRST_NAME               SALARY COMMISSION_PCT
    -------------------- ---------- --------------
    Steven                        0
    Neena                         0
    Lex                           0
    Nancy                         0
    Den                           0
    John                          0              0
    Karen                         0              0
    Alberto                       0              0
    Gerald                        0              0
    Eleni                         0              0
    Clara                         0              0
    Lisa                          0              0
    Ellen                         0              0
    Michael                       0
    Shelley                       0

    15 rows selected.

| SALES_APPユーザーではsalary列の値が全て0にリダクションされていますが、結果がHRユーザーのものと同じことが分かります。。
| したがって、特定のWHERE句（例えば、BETWEEN句）を用いることで、リダクションされたデータが元の値を推測可能な場合があります。


*****************************************
副問い合わせで使用した場合
*****************************************

副問い合わせを含むSQL文においても注意が必要です。以下の例では、 ``salary`` 列の平均を求める副問い合わせを使用しています。

.. code-block:: sql
    :caption: HRユーザーの場合

    select first_name, salary from employees where salary > (select avg(salary) from employees);

    SQL> select first_name, salary from employees where salary > (select avg(salary) from employees);

    FIRST_NAME               SALARY
    -------------------- ----------
    Steven                    24000
    Neena                     17000
    Lex                       17000
    Alexander                  9000
    Nancy                     12008
    Daniel                     9000
    ...
    Jack                       8400
    Kimberely                  7000
    Michael                   13000
    Susan                      6500
    Hermann                   10000
    Shelley                   12008
    William                    8300

    51 rows selected.


.. code-block:: sql
    :caption: SALES_APPユーザーの場合

    SELECT employee_id, first_name, last_name, salary FROM hr.employees WHERE salary > (SELECT AVG(salary) FROM hr.employees);

    select first_name, salary from hr.employees where salary > (select avg(salary) from hr.employees);


    SQL> select first_name, salary from hr.employees where salary > (select avg(salary) from hr.employees);

    FIRST_NAME               SALARY
    -------------------- ----------
    Steven                        0
    Neena                         0
    Lex                           0
    Alexander                     0
    Nancy                         0
    Daniel                        0
    ...
    Jack                          0
    Kimberely                     0
    Michael                       0
    Susan                         0
    Hermann                       0
    Shelley                       0
    William                       0

    51 rows selected.

| SALES_APPユーザーの結果では、salary列の値が全て0にリダクションされているにもかかわらず、副問い合わせの結果がHRユーザーと同じになっています。
| これは、リダクション後の値が副問い合わせに反映されないためであり、注意が必要です。


あくまでData Redactionはデータを隠すための機能であり、アクセス制御機能としてではないことに注意してください。

以上で、 Data Redactionのデモは終了です。