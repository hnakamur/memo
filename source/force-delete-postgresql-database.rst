=======================================
PostgreSQLのデータベースを強制削除する
=======================================

* :作成日: 2019-01-04
* :試した環境: Ubuntu 18.04 LTS, postgresql-11 11.1-1.pgdg18.04+1

:code:`sudo -iu postgres dropdb データベース名` で削除しようとしたら以下のように他のユーザによってアクセス中のため削除できないというエラーメッセージが表示されるケースがありました。

.. code:: console

   $ sudo -iu postgres dropdb zabbix
   dropdb: database removal failed: ERROR:  database "zabbix" is being accessed by other users
   DETAIL:  There are 4 other sessions using the database.

`postgresql - Force drop db while others may be connected - Database Administrators Stack Exchange <https://dba.stackexchange.com/questions/11893/force-drop-db-while-others-may-be-connected>`_ の手順で削除出来ました。

postgresユーザでPostgreSQLに接続します。

.. code:: console

   sudo -iu postgres psql

以下のSQLを実行します。

.. code:: text

   -- Making sure the database exists
   SELECT * from pg_database where datname = 'zabbix';

   -- Disallow new connections
   UPDATE pg_database SET datallowconn = 'false' WHERE datname = 'zabbix';
   ALTER DATABASE zabbix CONNECTION LIMIT 1;

   -- Terminate existing connections
   SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = 'zabbix';

   -- Drop database
   DROP DATABASE zabbix;

PostgreSQLのプロンプトで :code:`\q` を入力しpsqlを終了します。
