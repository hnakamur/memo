===============================================
DockerでApache Traffic Serverのテストを実行する
===============================================

* :更新日: 2018-12-27


はじめに
========

`githubのtrafficseverのレポジトリのtestsディレクトリのREADME <https://github.com/apache/trafficserver/tree/6ac53e6487e7ccd691530a59393491d6baf64228/tests>`__ (`最新版 <https://github.com/apache/trafficserver/tree/master/tests>`_)を参考にテストを実行しようとしたのですが、試行錯誤が必要だったので手順をDockerfileにまとめました。

.. code::

   FROM ubuntu:18.04

   # Install packages to build trafficserver
   RUN ln -fs /usr/share/zoneinfo/Etc/UTC /etc/localtime \
    && sed -i 's/^# deb-src/deb-src/' /etc/apt/sources.list \
    && apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get build-dep -y trafficserver \
    && apt-get install -y git \
    && apt-get install -y python3 python3-virtualenv virtualenv python3-dev curl netcat \
    && useradd -r -m -s /bin/bash build

   USER build

   # Get the source and configure trafficserver
   RUN mkdir -p ~/dev \
    && cd ~/dev \
    && git clone --depth 1 https://github.com/apache/trafficserver \
    && cd trafficserver \
    && git log -1 \
    && autoreconf -if

   # NOTE: Add patches here if available

   # Build trafficserver
   RUN cd ~/dev/trafficserver \
    && ./configure --enable-experimental-plugins \
    && make

   USER root
   RUN cd ~build/dev/trafficserver \
    && make install \
    && echo /usr/local/lib > /etc/ld.so.conf.d/trafficserver.conf \
    && ldconfig

   USER build

   # Set up test environment
   RUN cd ~/dev/trafficserver/tests \
    && virtualenv --python=python3 env-test \
    && env-test/bin/pip install pip --upgrade \
    && env-test/bin/pip install autest==1.7.2 hyper hyper requests dnslib httpbin

   # Run tests for trafficserver
   RUN cd ~/dev/trafficserver/tests \
    && . env-test/bin/activate \
    && env-test/bin/autest -D gold_tests --ats-bin /usr/local/bin

   USER root
   CMD ["/bin/bash"]

ビルド手順
==========

`githubのtrafficserverレポジトリのREADME <https://github.com/apache/trafficserver>`_ からリンクされている `Building <https://cwiki.apache.org/confluence/display/TS/Building>`_ とその(左のツリーでの)子ページの `Ubuntu <https://cwiki.apache.org/confluence/display/TS/Ubuntu>`_ を参考にしました。

当初rootユーザでビルドしていたのですが、テストの実行時にPIDファイルの作成でPermission Deniedになりました。非rootユーザでビルドするようにしたら解消したので、上記のDockerfileでは :code:`build` というユーザを作ってビルドしています。

テストツール
============

`githubのtrafficseverのレポジトリのtestsディレクトリのREADME <https://github.com/apache/trafficserver/tree/6ac53e6487e7ccd691530a59393491d6baf64228/tests>`__ に書かれていますが
`autest <https://pypi.org/project/autest/>`_ というテストツールを使っています。

   Reusable Gold testing system, or autest for short, is a testing system targeted toward gold file, command line process testing

AuTestのAuは金の元素記号ですね。bitbucketにレポジトリ `reusable-gold-testing-system <https://bitbucket.org/dragon512/reusable-gold-testing-system.git>`_ があります。

テストツールAuTestの拡張
========================

`trafficserverのgithubレポジトリのtests/gold_tests/autest-siteディレクトリ <https://github.com/apache/trafficserver/tree/6ac53e6487e7ccd691530a59393491d6baf64228/tests/gold_tests/autest-site>`_ にAuTestのtrafficserver用の拡張が含まれています。

`Writting tests for AuTest <https://github.com/apache/trafficserver/tree/6ac53e6487e7ccd691530a59393491d6baf64228/tests#writting-tests-for-autest>`_ の項に書かれているように :code:`autest` のコマンドに :code:`--ats-bin` というオプションを追加したりしています。

テストの実行
============

trafficserverのtestsディレクトリ(:code:`/home/build/dev/trafficserver/tests`) に移動して
:code:`virtualenv` で作成したPython3の環境を有効にしてテストを実行します。

.. code::

   cd ~/dev/trafficserver/tests
   . env-test/bin/activate \
   env-test/bin/autest -D gold_tests --ats-bin /usr/local/bin

:code:`autest` の :code:`-D` オプションで gold_tests ディレクトリを対象にして実行しています。

:code:`--filter` オプションを追加すると一部のテストだけ実行することもできます。
:code:`-D` で指定している gold_tests のサブディレクトリを指定すれば良いようです。
例えば tls のテストだけ実行するときは :code:`--filter tls` のようにします。

テストの実行結果
================

テストの実行結果が出力されるとともに、 :code:`_sandbox` というディレクトリが作られてその中にテスト実行時に作成された設定ファイルやログファイルが残っています。テストが失敗した場合はテストケースに対応したディレクトリ以下に含まれるログファイルを確認します。
