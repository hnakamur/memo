.. Hello documentation master file, created by
   sphinx-quickstart on Thu Nov 15 16:01:32 2018.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Windows 10にSphinxをインストール
================================

* :更新日: 2018-11-19
  
.. toctree::
   :maxdepth: 2
   :caption: Contents:

Chocolateyのインストール
------------------------

`Installing Chocolatey <https://chocolatey.org/docs/installation#installing-chocolatey>`_ の手順に従ってChocolateyをインストールします。
管理者権限でPowerShellを起動して以下のコマンドを実行します（コマンドが長くて折り返して表示されているかもしれませんが、一行のコマンドです）。

.. code-block:: powershell

    Set-ExecutionPolicy Bypass -Scope Process -Force; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))

PowerShellの実行ポリシーをUndefinedに変更します。

.. code-block:: powershell

    Set-ExecutionPolicy -ExecutionPolicy Undefined

.. note:: PowerShellの実行ポリシーの確認

    変更後の状態は以下のコマンドで確認できます。

    .. code-block:: powershell

        Set-ExecutionPolicy -List

    実行ポリシー確認の実行例です。

    .. code-block:: console

        PS C:\windows\system32> Get-ExecutionPolicy -List

                Scope ExecutionPolicy
                ----- ---------------
        MachinePolicy       Undefined
           UserPolicy       Undefined
              Process          Bypass
          CurrentUser       Undefined
         LocalMachine       Undefined

Python, make, PlantUMLのインストール
------------------------------------

以下の3つのコマンドを実行してpython3, make, PlantUMLをインストールします（依存関係でjre8やGraphVizもインストールされます）。
途中何回か確認されますので :code:`y` を押して進んでください。

.. code-block:: powershell

    choco install python3
    choco install make
    choco install plantuml

またgitとVisual Studio Codeもインストールしておきます。

.. code-block:: powershell

    choco install git
    choco install vscode

これでPATHの設定が変更されるので、powershellをいったん閉じて、再度管理者権限でPowerShellを起動します。

その後 :code:`Python -V` と実行してバージョンを表示し、Pythonが使えることを確認します。以下のように表示されればOKです。

.. code-block:: console

    PS C:\WINDOWS\system32> Python -V
    Python 3.7.1

pipが利用できることも確認します。

.. code-block:: console

    PS C:\WINDOWS\system32> pip --version
    pip 10.0.1 from c:\python37\lib\site-packages\pip (python 3.7)

次にpipを最新版にアップグレードします。

.. code-block:: console

    c:\python37\python.exe -m pip install --upgrade pip

.. note:: pipのアップグレード

    Linuxなどでは :code:`pip install --upgrade pip` で良いのですが、以下のエラーが出ました。

    .. code-block:: console

        PS C:\WINDOWS\system32> pip install --upgrade pip
        ERROR: To modify pip, please run the following command:
        c:\python37\python.exe -m pip install --upgrade pip
        You are using pip version 10.0.1, however version 18.1 is available.
        You should consider upgrading via the 'python -m pip install --upgrade pip' command.

    上のメッセージの通り以下のように実行すればOKでした。

Sphinxなどのパッケージをpipでインストール
-----------------------------------------

以下のように7つのパッケージをpipでインストールします。

.. code-block:: console

    pip install sphinx
    pip install sphinx-autobuild
    pip install sphinxcontrib-blockdiag
    pip install sphinxcontrib-seqdiag
    pip install sphinxcontrib-actdiag
    pip install sphinxcontrib-nwdiag
    pip install sphinxcontrib-plantuml

Noto Sans CJK jpフォントのダウンロード
--------------------------------------

blockdiagで図を生成する際にNoto Sans CJK jpフォントを使用するようにします。
通常は `Google Noto Fonts <https://www.google.com/get/noto/>`_ からダウンロードしてインストールするのですが、ここで配布されているのはOpenType形式でSphinxでは使えないようです。

そこで有志の方がTrueTypeに変換して公開されているファイルを利用します。
ブラウザで https://github.com/minoryorg/Noto-Sans-CJK-JP/blob/master/fonts/NotoSansCJKjp-Regular.ttf
を開き、[Download]ボタンをクリックしてダウンロードします。
:code:`C:\fonts` というフォルダを作成してそこに移動します。

Windowsで利用するわけではないので、Windowsのフォントへのインストールは不要です。
もしNoto Sans CJK jpフォントをWindowsで利用する場合は公式サイトからダウンロードしてWindowsのフォントインストールの手順を踏んでください。

自動ビルドの実行
----------------

powershellまたはコマンドプロントをSphinxのドキュメントのベースディレクトリで開き、以下のコマンドを実行します。

.. code-block:: console

    sphinx-autobuild.exe source build\html

ブラウザで http://127.0.0.1:8000 を開くとビルドされたHTMLを確認できます。
Sphinxで記述している文書のソースファイルを編集・保存したら、自動でビルドとブラウザのリロードが実行され、表示内容が更新されます。これは便利！
