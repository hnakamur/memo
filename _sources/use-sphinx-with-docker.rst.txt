======================
DockerでSphinxを使う
======================

* :更新日: 2018-12-01

はじめに
===========================

`Sphinx <http://www.sphinx-doc.org/en/master/>`_ をDockerで使うためのDockerイメージを作成し
`hnakamur/sphinx - Docker Hub <https://hub.docker.com/r/hnakamur/sphinx/>`_ に公開しています。

このDockerイメージは
`ブロック図生成ツール blockdiag <http://blockdiag.com/ja/index.html>`__ と
`UML作成ツール PlantUML <http://plantuml.com/>`__ も使えるように対応するSphinx拡張を組み込んであります。

以下ではこのDockerイメージを使ってSphinxを使う手順を説明します。

Docker ID作成
===========================

WindowsとmacOSの場合はDocker for Windows, Docker for MacをダウンロードするのにDocker IDというユーザIDの登録が必要になります。
Linuxでも自作のDockerイメージを `Docker Hub <https://hub.docker.com/>`_ で公開するにはDocker IDが必要になります。

`Docker ID accounts <https://docs.docker.com/docker-id/>`_　のRegister for a Docker IDの手順で登録してください。

Dockerのセットアップ
======================

Windows
---------

`Install Docker for Windows <https://docs.docker.com/docker-for-windows/install/>`_ の手順に従ってインストールします。

macOS
-------

`Install Docker for Mac <https://docs.docker.com/docker-for-mac/install/>`_ の手順に従ってインストールします。

Linux
-------

ディストリビューションに応じて下記のインストール手順でインストールします。

* `Get Docker CE for CentOS <https://docs.docker.com/install/linux/docker-ce/centos/>`_
* `Get Docker CE for Debian <https://docs.docker.com/install/linux/docker-ce/debian/>`_
* `Get Docker CE for Fedora <https://docs.docker.com/install/linux/docker-ce/fedora/>`_
* `Get Docker CE for Ubuntu <https://docs.docker.com/install/linux/docker-ce/ubuntu/>`_ 

Dockerの共有フォルダのセットアップ
====================================

Windowsでは共有フォルダのセットアップが必要です。

1. タスクトレイのDockerアイコンを右クリックしてポップアップメニューを開き[Settings]を選択。
2. 左のリストで[Shared Drives]を選択。
3. 右のリストで共有フォルダを使いたいドライブを選択して[Shared]の列のチェックボックスを選択し[Apply]ボタンを押す。

`F-Secure <https://www.f-secure.com/ja_JP/welcome>`_ をお使いの場合は
`Allowing file sharing with F-Secure firewall turned on - F-Secure Community <https://community.f-secure.com/t5/Business/Allowing-file-sharing-with-F/ta-p/77051>`_ の手順でファイアウォール設定を変更する必要があります。


Sphinxドキュメントプロジェクトの作成
======================================

Sphinxドキュメントプロジェクトの作成には
`sphinx-quickstart <http://www.sphinx-doc.org/ja/stable/invocation.html>`_
を使います。

:code:`hnakamur/sphinx` のDockerイメージにはこれをラップしたスクリプト :code:`my-sphinx-quickstart` が含まれています。
blockdiagやPlantUMLのSphinx拡張を組み込み、テーマは `Sphinx Themes <http://sphinx-themes.org/>`_ で紹介されていた `solar-theme <https://pypi.org/project/solar-theme/>`_ を使うようになっています。

Windows
----------

ドキュメントプロジェクトを作成し、コマンドプロンプトを開いてそこに移動し、以下のコマンドを実行します。

.. code-block:: console

   docker run --rm -it -v "%cd%:/documents" hnakamur/sphinx my-sphinx-quickstart

プロジェクト名、著者名、リリース番号を聞かれますので入力します。

あるいは以下のようにオプションで指定することも可能です（コマンドが長くて折り返して表示されていると思いますが1行で入力してください。以下全てのコマンドも同様です）。

.. code-block:: console

   docker run --rm -it -v "%cd%:/documents" hnakamur/sphinx my-sphinx-quickstart -p "YourProjectName" -a "John Doe <john.doe@example.com>" -r 1.0


macOS
------

ドキュメントプロジェクトを作成し、ターミナルを開いてそこに移動し、以下のコマンドを実行します。

.. code-block:: console

   docker run --rm -it -v "$PWD:/documents" hnakamur/sphinx my-sphinx-quickstart

プロジェクト名、著者名、リリース番号を聞かれますので入力します。

あるいは以下のようにオプションで指定することも可能です。

.. code-block:: console

   docker run --rm -it -v "$PWD:/documents" hnakamur/sphinx my-sphinx-quickstart -p "YourProjectName" -a "John Doe <john.doe@example.com>" -r 1.0

Linux
-------

ドキュメントプロジェクトを作成し、ターミナルを開いてそこに移動し、以下のコマンドを実行します。

.. code-block:: console

   docker run --rm -it -v "$PWD:/documents" -e USE_GOSU=1 -e LOCAL_UID=$(id -u $USER) -e LOCAL_GID=$(id -g $USER) hnakamur/sphinx my-sphinx-quickstart

プロジェクト名、著者名、リリース番号を聞かれますので入力します。

あるいは以下のようにオプションで指定することも可能です。

.. code-block:: console

   docker run --rm -it -v "$PWD:/documents" -e USE_GOSU=1 -e LOCAL_UID=$(id -u $USER) -e LOCAL_GID=$(id -g $USER) hnakamur/sphinx my-sphinx-quickstart -p "YourProjectName" -a "John Doe <john.doe@example.com>" -r 1.0

ドキュメントのビルド
======================================

ドキュメントを書いて保存したら以下の手順でビルドします。

Windows
----------

ビルドには以下のコマンドを実行します。

.. code-block:: console

   docker run --rm -it -v "%cd%:/documents" hnakamur/sphinx make html

あるいは以下のコマンドでもビルドできます。

.. code-block:: console

   docker run --rm -it -v "%cd%:/documents" hnakamur/sphinx sphinx-build -b html source build

macOS
----------

ビルドには以下のコマンドを実行します。

.. code-block:: console

   docker run --rm -it -v "$PWD:/documents" hnakamur/sphinx make html

あるいは以下のコマンドでもビルドできます。

.. code-block:: console

   docker run --rm -it -v "$PWD:/documents" hnakamur/sphinx sphinx-build -b html source build


Linux
----------

ビルドには以下のコマンドを実行します。

.. code-block:: console

   docker run --rm -it -v "$PWD:/documents" -e USE_GOSU=1 -e LOCAL_UID=$(id -u $USER) -e LOCAL_GID=$(id -g $USER) hnakamur/sphinx make html

あるいは以下のコマンドでもビルドできます。

.. code-block:: console

   docker run --rm -it -v "$PWD:/documents" -e USE_GOSU=1 -e LOCAL_UID=$(id -u $USER) -e LOCAL_GID=$(id -g $USER) hnakamur/sphinx sphinx-build -b html source build


ドキュメントを編集しつつ自動ビルド
======================================

`sphinx-autobuild <https://pypi.org/project/sphinx-autobuild/>`_ を使うと、ドキュメントを編集し保存したら自動でビルドを実行してブラウザをリロードできます。

Windows
----------

コマンドプロンプトを開いてドキュメントプロジェクトのディレクトリに移動し以下のコマンド実行します。

.. code-block:: console

   docker run --rm -it -v "%cd%:/documents" -p 8000:8000 hnakamur/sphinx sphinx-autobuild source build/html -H 0.0.0.0

ブラウザで http://127.0.0.1:8000 を開くとビルドされたドキュメントを確認できます。ポートを変えたい場合は :code:`-p` オプションのコロンより前の番号を変えてください。

本来ならこれだけで良いのですが、Docker for Windowsの制約でWindows上のファイルを保存しても変更通知がコンテナに伝わらず自動ビルドが走らないという問題があります。

回避策として
`hnakamur/docker-windows-volume-watcher <https://github.com/hnakamur/docker-windows-volume-watcher>`_
を使います。
`hnakamur/docker-windows-volume-watcherのReleasesページ <https://github.com/hnakamur/docker-windows-volume-watcher>`_
から :code:`docker-windows-volume-watcher` をダウンロードしてください。

もう1つコマンドプロンプトを開き以下のコマンドを実行してください。

.. code-block:: console

   docker-windows-volume-watcher.exe -ignoredir .git;build

これでドキュメントのファイルを保存すると自動ビルドが走りブラウザがリロードされます。

macOS
----------

ターミナルを開いてドキュメントプロジェクトのディレクトリに移動し以下のコマンド実行します。

.. code-block:: console

   docker run --rm -it -v "$PWD:/documents" -p 8000:8000 hnakamur/sphinx sphinx-autobuild source build/html -H 0.0.0.0

ブラウザで http://127.0.0.1:8000 を開くとビルドされたドキュメントを確認できます。ポートを変えたい場合は :code:`-p` オプションのコロンより前の番号を変えてください。

Linux
----------

ターミナルを開いてドキュメントプロジェクトのディレクトリに移動し以下のコマンド実行します。

.. code-block:: console

   docker run --rm -it -v "$PWD:/documents" -p 8000:8000 -e USE_GOSU=1 -e LOCAL_UID=$(id -u $USER) -e LOCAL_GID=$(id -g $USER) hnakamur/sphinx sphinx-autobuild source build/html -H 0.0.0.0

ブラウザで http://127.0.0.1:8000 を開くとビルドされたドキュメントを確認できます。ポートを変えたい場合は :code:`-p` オプションのコロンより前の番号を変えてください。

ビルドしたドキュメントをGitHub Pagesで公開
=============================================

以下のコマンドを実行するとビルドしたHTMLをGitHub Pagesで公開できます。

.. code-block:: console

   git subtree push --prefix build/html/ origin gh-pages

なお、これを行うためにはbuild以下のファイルもmasterブランチでコミットしておく必要がありました。

また、GitHub Pagesを使うための設定として、一度 :code:`gh-pages` ブランチにプッシュした後でGitHubのプロジェクトの[Settings]のGitHub PagesセクションでSourceの下のドロップダウンで[gh-pages branch]を選んで[Save]ボタンを押す必要があります。
