=========================================================
ZabbixをPostgreSQLとTimescaleDBとPipelineDBでセットアップ
=========================================================
 
* :作成日: 2019-01-04

はじめに
========

Zabbix 4.0をPostgreSQL 11とTimescaleDBとPipelineDBでセットアップするメモです。
Ubuntu 18.04 LTSでLXDコンテナ内で試しました。

LXDコンテナ作成
===============

LXDでUbuntu 18.04 LTSのコンテナを作成します。

.. code:: console

   lxc launch ubuntu:18.04 zabbix

作成したコンテナ内でシェルを実行します。

.. code:: console

   lxc exec zabbix bash

以降の処理はコンテナ内で実行します。上記の手順でLXDコンテナ内でrootユーザになっているのでsudoは不要ですが、LXDコンテナではない環境でインストールするときにも同じ手順が使えるように以下ではsudo付きで説明します。

PostgreSQLのインストール
========================

`PostgreSQL: Linux downloads (Ubuntu) <https://www.postgresql.org/download/linux/ubuntu/>`_ の手順にしたがってインストールします。
   
PostgreSQLのaptレポジトリを追加します。

.. code:: console

   echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -c -s)-pgdg main" | sudo tee /etc/apt/sources.list.d/pgdg.list

レポジトリの署名キーを追加します。

.. code:: console

   curl -L https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
   
レポジトリを追加したのでパッケージ情報をアップデートします。

.. code:: console

   sudo apt-get update

PostgreSQL 11のパッケージをインストールします。
   
.. code:: console

   sudo apt-get -y install postgresql-11
   
.. note::

   インストール時に最後に以下のようなメッセージが表示されていますが、実際は既にmainという名前のデータベースが作成され、postgresqlという名前のサービスでデータベースサーバが起動された状態になっています。

   .. code:: text
      
      Success. You can now start the database server using:
      
          /usr/lib/postgresql/11/bin/pg_ctl -D /var/lib/postgresql/11/main -l logfile start

      Ver Cluster Port Status Owner    Data directory              Log file
      11  main    5432 down   postgres /var/lib/postgresql/11/main /var/log/postgresql/postgresql-11-main.log

   上のメッセージにあるとおり、データベースのディレクトリは/var/lib/postgresql/11/mainでログファイルは/var/log/postgresql/postgresql-11-main.logです。

   また設定ファイルのディレクトリは/etc/postgresql/11/mainです。

.. note::

   以下のコマンドで管理者であるpostgresユーザでPostgreSQLに接続できます。

   .. code:: console

      sudo -iu postgres psql

   psqlのプロンプト :code:`postgres=#` で :code:`\q` を入力するとpsqlを抜けてシェルに戻ります。

   .. code:: console

      postgres=# \q

.. note::

   /etc/postgresql/11/main/pg_hba.conf を確認してみると以下の行があるので、接続用のユーザを作成して接続先に127.0.0.1を指定すればパスワードありで接続可能です。

   .. code:: text

      # IPv4 local connections:
      host    all             all             127.0.0.1/32            md5

TimescaleDBのインストール
=========================

`TimescaleDB Docs | Installing <https://docs.timescale.com/v1.1/getting-started/installation/ubuntu/installation-apt-ubuntu>`_ の手順にしたがってインストールします。

`TimescaleDBのPPAのレポジトリ <https://launchpad.net/~timescale/+archive/ubuntu/timescaledb-ppa>`_ を追加します。

.. code:: console

   sudo add-apt-repository -y ppa:timescale/timescaledb-ppa

レポジトリを追加したのでパッケージ情報をアップデートします。

.. code:: console

   sudo apt-get update

PostgreSQL 11用のTimescaleDBパッケージをインストールします。

.. code:: console

   sudo apt-get -y install timescaledb-postgresql-11

インストール時に以下のようなメッセージが表示されます。この後PipelineDBのインストール後にまとめて設定します。

.. code:: text

   TimescaleDB has been installed. You need to update your postgresql.conf file
   to load the library by adding TimescaleDB to your shared_preload_libraries.
   Find the line below and change the value as shown (uncomment if needed):
   
   shared_preload_libraries = 'timescaledb'

`Telemetry and Version Checking <https://docs.timescale.com/v1.1/using-timescaledb/telemetry>`_ で説明されているようにTimescaleDBは製品の改善のために利用の統計情報をデフォルトで送信するようになっています。

これを無効にするには以下のコマンドを実行します。PostgreSQLのサービスの再起動も必要だと思いますが、これは以下で行います。

.. code:: console

   echo timescaledb.telemetry_level=off | sudo tee -a /etc/postgresql/11/main/postgresql.conf

.. note::

   統計情報を送信していることは、以下の手順で :code:`sudo -iu postgres psql -c "CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;" zabbix` と実行したときに以下のようにメッセージが表示されて説明がありました。

   .. code:: text

      WARNING:
      WELCOME TO
       _____ _                               _     ____________
      |_   _(_)                             | |    |  _  \ ___ \
        | |  _ _ __ ___   ___  ___  ___ __ _| | ___| | | | |_/ /
        | | | |  _ ` _ \ / _ \/ __|/ __/ _` | |/ _ \ | | | ___ \
        | | | | | | | | |  __/\__ \ (_| (_| | |  __/ |/ /| |_/ /
        |_| |_|_| |_| |_|\___||___/\___\__,_|_|\___|___/ \____/
                     Running version 1.1.1
      For more information on TimescaleDB, please visit the following links:

       1. Getting started: https://docs.timescale.com/getting-started
       2. API reference documentation: https://docs.timescale.com/api
       3. How TimescaleDB is designed: https://docs.timescale.com/introduction/architecture

      Note: TimescaleDB collects anonymous reports to better understand and assist our users.
      For more information and how to disable, please see our docs https://docs.timescaledb.com/using-timescaledb/telemetry.

PipelineDBのインストール
========================

`PipelineDB Installation — PipelineDB 1.0.0 documentation <http://docs.pipelinedb.com/installation.html>`_ の手順にしたがってインストールします。

インストールスクリプトをダウンロードします。

.. code:: console

   sudo curl -LO http://download.pipelinedb.com/apt.sh

実行前に内容を確認します。

.. code:: console

   vim apt.sh

インストールスクリプトを実行します。実際にどういうコマンドが実行されたかを確認したいので :code:`bash` の :code:`-x` オプションを指定して実行します。 :code:`apt-transport-https` パッケージのインストール、レポジトリの追加、GPGキーの追加、 :code:`apt-get update` が実行されます。

.. code:: console

   sudo bash -x apt.sh

PostgreSQL 11用のPipelineDBパッケージをインストールします。

.. code:: console

   sudo apt-get -y install pipelinedb-postgresql-11

最後に以下のようなメッセージが表示されます。次項でTimescaleDBの設定と合わせて設定します。

.. code:: text

   Next, edit <data directory>/postgresql.conf and set:
   
     shared_preload_libraries = 'pipelinedb'
     max_worker_processes = 128
   
   Once your database is running, create the pipelinedb extension:
   
     psql -c 'CREATE EXTENSION pipelinedb'

TimescaleDBとPipelineDBの設定
=============================

上記でインストール時にメッセージで出力された設定を行います。

まずPostgreSQLの設定ファイル :code:`/etc/postgresql/11/main/postgresql.conf` 内で変更前の :code:`shared_preload_libraries` の設定を確認します。

.. code:: console

   grep shared_preload_libraries /etc/postgresql/11/main/postgresql.conf

実行してみると以下のようになりました。

.. code:: console

   root@zabbix:~# grep shared_preload_libraries /etc/postgresql/11/main/postgresql.conf
   #shared_preload_libraries = ''  # (change requires restart)

次にPostgreSQLの設定ファイル :code:`/etc/postgresql/11/main/postgresql.conf` 内で変更前の :code:`max_worker_processes` の設定を確認します。

.. code:: console

   grep max_worker_processes /etc/postgresql/11/main/postgresql.conf

実行してみると以下のようになりました。

.. code:: console

   root@zabbix:~# grep max_worker_processes /etc/postgresql/11/main/postgresql.conf
   #max_worker_processes = 8               # (change requires restart)
   #max_parallel_workers = 8               # maximum number of max_worker_processes that
   #max_logical_replication_workers = 4    # taken from max_worker_processes

`shared_preload_libraries <https://www.postgresql.org/docs/11/runtime-config-client.html#GUC-SHARED-PRELOAD-LIBRARIES>`_ によると複数のライブラリを指定するにはカンマ区切りにすれば良いそうです。ということで、以下のように :code:`shared_preload_libraries` と :code:`max_worker_processes` を設定します。

.. code:: console

   sudo sed -i -e "s/^#shared_preload_libraries = ''/shared_preload_libraries = 'timescaledb,pipelinedb'/;s/^#max_worker_processes = 8/max_worker_processes = 128/" /etc/postgresql/11/main/postgresql.conf

以下のコマンドで変更結果を確認します。

.. code:: console

   grep -E '(shared_preload_libraries|max_worker_processes)' /etc/postgresql/11/main/postgresql.conf

結果は以下のようになり期待通り変更できていました。

.. code:: console

   root@zabbix:~# grep -E '(shared_preload_libraries|max_worker_processes)' /etc/postgresql/11/main/postgresql.conf
   max_worker_processes = 128              # (change requires restart)
   #max_parallel_workers = 8               # maximum number of max_worker_processes that
   #max_logical_replication_workers = 4    # taken from max_worker_processes
   shared_preload_libraries = 'timescaledb,pipelinedb'     # (change requires restart)

設定変更を反映させるためPostgreSQLのサービスを再起動します。

.. code:: console

   sudo systemctl restart postgresql

Zabbixのインストール
====================

`2 Debian/Ubuntu/Raspbian [Zabbix Documentation 4.0] <https://www.zabbix.com/documentation/4.0/manual/installation/install_from_packages/debian_ubuntu>`_ の手順にしたがってインストールします。

Zabbixのaptレポジトリを追加するためのパッケージファイルをダウンロードします。

.. code:: console

   sudo curl -LO https://repo.zabbix.com/zabbix/4.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_4.0-2+bionic_all.deb

Zabbixのaptレポジトリを追加するためのパッケージファイルをインストールします。

.. code:: console

   sudo dpkg -i zabbix-release_4.0-2+bionic_all.deb

レポジトリを追加したのでパッケージ情報をアップデートします。

.. code:: console

   sudo apt-get update

PostgreSQL用のZabbixサーバ、Zabbixのフロントエンド、PHPのPostgreSQLモジュールをインストールします。

.. code:: console

   sudo apt-get -y install zabbix-server-pgsql zabbix-frontend-php php-pgsql

.. note::

   インストールの途中で以下のようなメッセージが表示されていました。

   .. code:: text

      adduser: Warning: The home directory `/var/lib/snmp' does not belong to the user you are currently creating.
      Created symlink /etc/systemd/system/multi-user.target.wants/snmpd.service → /lib/systemd/system/snmpd.service.

   /etc/passwd を確認するとzabbixユーザの他にDebian-snmpというユーザが作られたようです。

   .. code:: console

      root@zabbix:~# grep -E '(/var/lib/snmp|zabbix)' /etc/passwd
      Debian-snmp:x:112:117::/var/lib/snmp:/bin/false
      zabbix:x:113:118::/var/lib/zabbix/:/usr/sbin/nologin

   /var/lib/snmp の所有者を確認するとDebian-snmpユーザだったので上記のメッセージが出たあとに所有者が変更されたようです。

   .. code:: console

      root@zabbix:~# ls -ld /var/lib/snmp
      drwxr-xr-x 3 Debian-snmp Debian-snmp 4096 Dec 31 01:04 /var/lib/snmp

Zabbixのデータベース作成
========================

`Zabbix 4.0 LTS, Ubuntu 18.04, PostgreSQLのダウンロードページ <https://www.zabbix.com/jp/download?zabbix=4.0&os_distribution=ubuntu&os_version=bionic&db=PostgreSQL>`__ を参考にして以下の手順で作成します。

PostgreSQLにzabbixというユーザを作成します。パスワードのプロンプトが出ますのでzabbixユーザ用に設定したいパスワードを入力します。

.. code:: console

   sudo -iu postgres createuser -P zabbix

.. note::

   :code:`-P` オプションを忘れてパスワード無しでユーザを作成してしまった場合、以下のように実行すればパスワードを設定できます。

   .. code:: console

      sudo -iu postgres psql -c "ALTER USER zabbix WITH ENCRYPTED PASSWORD '設定したいパスワード';"

PostgreSQLにzabbixというデータベースをzabbixユーザを所有者にして作成します。

.. code:: console

   sudo -iu postgres createdb -O zabbix zabbix

zabbixデータベース内にTimescaleDBのエクステンションを作成します。

.. code:: console

   sudo -iu postgres psql -c "CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;" zabbix

zabbixデータベース内にPipelineDBのエクステンションを作成します。

.. code:: console

   sudo -iu postgres psql -c "CREATE EXTENSION IF NOT EXISTS pipelinedb" zabbix

Zabbix用のデータベーススキーマを作成するSQLをzabbixユーザで実行します。パスワードプロンプトが出ますので上記で設定したパスワードを入力してください。

.. code:: console

   zcat /usr/share/doc/zabbix-server-pgsql/create.sql.gz | psql -h localhost -U zabbix zabbix

上記で作成したスキーマに含まれるhistoryテーブルに対してTimescaleDBのhypertableを作成します。パスワードプロンプトが出ますので上記で設定したパスワードを入力してください。

.. code:: console

   psql -h localhost -U zabbix -c "SELECT create_hypertable('history', 'clock', chunk_time_interval => 2592000)" zabbix

.. note::

   historyテーブルの定義は以下のようになっており、clockカラムの型はtimestamp with time zoneやtimestampではなくintegerになっています。

   .. code:: console

      root@zabbix:~# sudo -iu postgres psql -c '\d history' zabbix
                       Table "public.history"
       Column |     Type      | Collation | Nullable | Default
      --------+---------------+-----------+----------+---------
       itemid | bigint        |           | not null |
       clock  | integer       |           | not null | 0
       value  | numeric(16,4) |           | not null | 0.0000
       ns     | integer       |           | not null | 0
      Indexes:
          "history_1" btree (itemid, clock)
          "history_clock_idx" btree (clock DESC)
      Triggers:
          ts_insert_blocker BEFORE INSERT ON history FOR EACH ROW EXECUTE PROCEDURE _timescaledb_internal.insert_blocker()
      Number of child tables: 1 (Use \d+ to list them.)

   このため `Time column as integer type fails with default config ("PGRES_FATAL_ERROR:ERROR value 2592000000000 is out of range") · Issue #150 · timescale/timescaledb <https://github.com/timescale/timescaledb/issues/150>`_ にあるように上記のcreate_hypertableには :code:`chunk_time_interval => 2592000` を指定する必要があります。

Zabbixサーバの設定
===================

`Zabbix 4.0 LTS, Ubuntu 18.04, PostgreSQLのダウンロードページ <https://www.zabbix.com/jp/download?zabbix=4.0&os_distribution=ubuntu&os_version=bionic&db=PostgreSQL>`__ を参考にして以下の手順で作成します。

以下のコマンドを実行し :code:`/etc/zabbix/zabbix_server.conf` を編集して :code:`DBPassword` の値を設定します。

.. code:: console

   sudo vim /etc/zabbix/zabbix_server.conf

:code:`DBPassword` の設定内容は以下のようにします。コメントの部分は元からあるので :code:`DBPassword=上記で設定したパスワード` の行を追加します。

.. code:: text

   ### Option: DBPassword
   #       Database password.
   #       Comment this line if no password is used.
   #
   # Mandatory: no
   # Default:
   # DBPassword=

   DBPassword=上記で設定したパスワード

以下のコマンドで現在のZabbixサーバのサービスの起動状態を確認します。停止状態になっているはずです。

.. code:: console

   sudo systemctl status zabbix-server

以下のコマンドで現在のZabbixサーバのサービスを起動します。

.. code:: console

   sudo systemctl status zabbix-server

再び以下のコマンドで現在のZabbixサーバのサービスの起動状態を確認します。今度は起動状態になっているはずです。

.. code:: console

   sudo systemctl status zabbix-server

Zabbixフロントエンドの設定
===========================

`Zabbix 4.0 LTS, Ubuntu 18.04, PostgreSQLのダウンロードページ <https://www.zabbix.com/jp/download?zabbix=4.0&os_distribution=ubuntu&os_version=bionic&db=PostgreSQL>`__ からリンクされている `3 Installation from sources [Zabbix Documentation 4.0] <https://www.zabbix.com/documentation/4.0/manual/installation/install#installing_frontend>`_　を参考に以下のように設定します。

:code:`/etc/zabbix/apache.conf` を編集して :code:`<IfModule mod_php7.c>` 内の date.timezone の値を設定します。ここでは :code:`Asia/Tokyo` にしています。

.. code:: console

   sudo vim /etc/zabbix/apache.conf

変更前は以下のようになっていました。

.. code:: text

       <IfModule mod_php7.c>
           php_value max_execution_time 300
           php_value memory_limit 128M
           php_value post_max_size 16M
           php_value upload_max_filesize 2M
           php_value max_input_time 300
           php_value max_input_vars 10000
           php_value always_populate_raw_post_data -1
           # php_value date.timezone Europe/Riga
       </IfModule>

:code:`date.timezone` の行を以下のように書き換えました。

.. code:: text

       <IfModule mod_php7.c>
           php_value max_execution_time 300
           php_value memory_limit 128M
           php_value post_max_size 16M
           php_value upload_max_filesize 2M
           php_value max_input_time 300
           php_value max_input_vars 10000
           php_value always_populate_raw_post_data -1
           php_value date.timezone Asia/Tokyo
       </IfModule>

.. note::

   :code:`/etc/zabbix/apache.conf` は以下のようにapache2から参照されるようになっていました。

   .. code:: console

      root@zabbix:~# ls -l /etc/apache2/conf-available/zabbix.conf /etc/apache2/conf-enabled/zabbix.conf
      lrwxrwxrwx 1 root root 23 Dec 31 22:41 /etc/apache2/conf-available/zabbix.conf -> /etc/zabbix/apache.conf
      lrwxrwxrwx 1 root root 29 Dec 31 22:41 /etc/apache2/conf-enabled/zabbix.conf -> ../conf-available/zabbix.conf

   またapache2のサイト設定ファイルは以下のようになっていました。

   .. code:: console

      root@zabbix:~# ls -l /etc/apache2/sites-available/
      total 12
      -rw-r--r-- 1 root root 1332 Oct 10 18:59 000-default.conf
      -rw-r--r-- 1 root root 6338 Oct 10 18:59 default-ssl.conf
      root@zabbix:~# ls -l /etc/apache2/sites-enabled/
      total 0
      lrwxrwxrwx 1 root root 35 Dec 31 22:41 000-default.conf -> ../sites-available/000-default.conf

`3 Installation from sources [Zabbix Documentation 4.0] <https://www.zabbix.com/documentation/4.0/manual/installation/install#installing_zabbix_web_interface>`_ を参考に、以下のようにしてZabbixフロントエンドのPHPファイルをapache2のドキュメントルート配下にコピーしました。

.. code:: console

   sudo cp -pr /usr/share/zabbix /var/www/html/

.. note::

   以下のように :code:`/var/www/html/zabbix/index.php` というファイルが出来ていればOKです。

   .. code:: console

         root@zabbix:~# ls /var/www/html/zabbix/index.php
         /var/www/html/zabbix/index.php

:code:`/` を :code:`/zabbix` にリダイレクトするように設定します。

.. code:: console

   sudo vim /etc/apache2/mods-enabled/alias.conf

変更前の設定内容は以下のとおりでした。

.. code:: text

   <IfModule alias_module>
           # Aliases: Add here as many aliases as you need (with no limit). The format is
           # Alias fakename realname
           #
           # Note that if you include a trailing / on fakename then the server will
           # require it to be present in the URL.  So "/icons" isn't aliased in this
           # example, only "/icons/".  If the fakename is slash-terminated, then the
           # realname must also be slash terminated, and if the fakename omits the
           # trailing slash, the realname must also omit it.
           #
           # We include the /icons/ alias for FancyIndexed directory listings.  If
           # you do not use FancyIndexing, you may comment this out.

           Alias /icons/ "/usr/share/apache2/icons/"

           <Directory "/usr/share/apache2/icons">
                   Options FollowSymlinks
                   AllowOverride None
                   Require all granted
           </Directory>

   </IfModule>

   # vim: syntax=apache ts=4 sw=4 sts=4 sr noet

以下のように :code:`RedirectMatch permanent "^/$" "/zabbix"` の行を追加します。

.. code:: text

   <IfModule alias_module>
           # Aliases: Add here as many aliases as you need (with no limit). The format is
           # Alias fakename realname
           #
           # Note that if you include a trailing / on fakename then the server will
           # require it to be present in the URL.  So "/icons" isn't aliased in this
           # example, only "/icons/".  If the fakename is slash-terminated, then the
           # realname must also be slash terminated, and if the fakename omits the
           # trailing slash, the realname must also omit it.
           #
           # We include the /icons/ alias for FancyIndexed directory listings.  If
           # you do not use FancyIndexing, you may comment this out.

           Alias /icons/ "/usr/share/apache2/icons/"

           <Directory "/usr/share/apache2/icons">
                   Options FollowSymlinks
                   AllowOverride None
                   Require all granted
           </Directory>

           RedirectMatch permanent "^/$" "/zabbix"

   </IfModule>

   # vim: syntax=apache ts=4 sw=4 sts=4 sr noet


apache2のサービスを再起動します。

.. code:: console

   systemctl restart apache2

ウェブブラウザで :code:`http://コンテナのIPアドレス/zabbix` を開くとZabbixフロントエンドのセットアップ画面になります。

.. note::

   LXDコンテナのホストと別のマシンからアクセスする場合は、LXDホストにnginxを入れてリバースプロキシするか `第535回　LXD 3.0のネットワーク設定：Ubuntu Weekly Recipe｜gihyo.jp … 技術評論社 <https://gihyo.jp/admin/serial/01/ubuntu-recipe/0535?page=4>`_ の「特定のアドレス・ポートを公開する」の手順でコンテナのポートを公開してください。

   例えばzabbixというコンテナのIPアドレスが10.70.121.121で、コンテナのポート80をLXDをホストの8080で公開するには以下のようにします。

   .. code:: console

      lxc config device add zabbix http proxy listen=tcp:0.0.0.0:8080 connect=tcp:10.70.121.121:80 bind=host

   作成したプロキシの情報確認はLXDホストで以下のコマンドを実行します。

   .. code:: console

      lxc config device show zabbix

   実行例は以下のようになります。

   .. code:: console

      LXDホスト$ lxc config device show zabbix
      http:
        bind: host
        connect: tcp:10.70.121.121:80
        listen: tcp:0.0.0.0:80
        type: proxy

   上記で作成したプロキシを削除するには以下のようにします。

   .. code:: console

      lxc config device rm zabbix http

セットアップ画面では以下のように操作します。

1. タイトルがCheck of pre-requisitesのページでは一番右の列が全てOKになっていることを確認し、[Next step]ボタンを押します。

2. タイトルがConfigure DB connectionのページではDatabase TypeがPostgreSQLになっていることを確認し、Passwordの欄に上記で設定したパスワードを入力して[Next step]ボタンを押します。

3. タイトルがZabbix server detailsのページはそのまま[Next step]ボタンを押します。

4. Pre-installation summaryのページで設定内容を確認し、問題なければ[Next step]ボタンを押します。

5. :code:`Congratulations! You have successfully installed Zabbix frontend.` と表示されれば成功です。[Finish]ボタンを押すとZabbixのログイン画面が表示されます。

.. note::

   上記の :code:`Congratulations! You have successfully installed Zabbix frontend.` のメッセージの下に :code:`Configuration file "/usr/share/zabbix/conf/zabbix.conf.php" created.` と表示されていました。

   確認してみると :code:`/usr/share/zabbix/conf/zabbix.conf.php` と :code:`/var/www/html/zabbix/conf/zabbix.conf.php` はともに :code:`/etc/zabbix/web/zabbix.conf.php` へのシンボリックリンクになっていました。

   .. code:: console

      root@zabbix:~# ls -l /usr/share/zabbix/conf/zabbix.conf.php
      lrwxrwxrwx 1 root root 31 Dec 20 09:50 /usr/share/zabbix/conf/zabbix.conf.php -> /etc/zabbix/web/zabbix.conf.php
      root@zabbix:~# ls -l /var/www/html/zabbix/conf/zabbix.conf.php
      lrwxrwxrwx 1 root root 31 Dec 20 09:50 /var/www/html/zabbix/conf/zabbix.conf.php -> /etc/zabbix/web/zabbix.conf.php

   /etc/zabbix/web/zabbix.conf.php の内容は以下のようになっていました。

   .. code:: php

      <?php
      // Zabbix GUI configuration file.
      global $DB;

      $DB['TYPE']     = 'POSTGRESQL';
      $DB['SERVER']   = '127.0.0.1';
      $DB['PORT']     = '0';
      $DB['DATABASE'] = 'zabbix';
      $DB['USER']     = 'zabbix';
      $DB['PASSWORD'] = '上記で設定したパスワード';

      // Schema name. Used for IBM DB2 and PostgreSQL.
      $DB['SCHEMA'] = '';

      $ZBX_SERVER      = 'localhost';
      $ZBX_SERVER_PORT = '10051';
      $ZBX_SERVER_NAME = '';

      $IMAGE_FORMAT_DEFAULT = IMAGE_FORMAT_PNG;

Zabbixフロントエンドへのログイン
================================

ウェブブラウザで :code:`http://コンテナのIPアドレス/zabbix` を開くとZabbixのログイン画面が表示されます。初期設定ではユーザ名 :code:`Admin` 、パスワードは :code:`zabbix` でログイン可能です。ログイン後ページ右上の人のアイコンをクリックしてパスワードを変更してください。

Zabbixエージェントのインストール
=================================

以下のコマンドを実行してZabbixエージェントをインストールします。

.. code:: console

   sudo apt-get -y install zabbix-agent

インストール後以下の手順でZabbixエージェントからのデータが来ていることを確認します。

1. Zabbixのフロントエンドで[Monitoring]/[Latest data]メニューを選択。
2. フィルタのフォームのHostsの[Select]ボタンを押しポップアップが表示されたらZabbix serverを選んで[Select]を押す。
3. ポップアップが閉じたら[Apply]を押す。
4. フィルタのフォームの下に最新データ一覧が表示され、Last checkの時刻がほぼ現在日時になっていればOK。
