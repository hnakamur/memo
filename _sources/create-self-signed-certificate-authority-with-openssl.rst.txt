============================
OpenSSLで自己認証局を作る
============================

* :作成日: 2018-12-05

毎回検索してたので自分用にまとめておきます。

`How to setup your own CA with OpenSSL <https://gist.github.com/Soarez/9688998>`_ と
`CentOS7.2 64bit OpenSSLを使用して自己認証局で署名したSSLクライアント証明書を作成 | kakiro-web カキローウェブ <https://www.kakiro-web.com/linux/ssl-client.html>`_ を参考にしました。

試した環境はUbuntu 18.04.1 LTSでOpenSSLのバージョンは1.1.0gです。

.. code:: console

   $ openssl version
   OpenSSL 1.1.0g  2 Nov 2017

自己認証局を作成
=================

管理しやすくするため、OSのディストリビューションで用意されているファイルは使わず、独立した作業ディレクトリを作るという方針にします。

カレントディレクトリに :code:`myCA` というディレクトリを作ってそこで管理することにします。以下のディレクトリ名、ファイル名、証明書のサブジェクトは適宜変更してください。

.. code:: console

   mkdir -p myCA

.. code:: console

   cd myCA

認証局の秘密鍵用のディレクトリを作って鍵を生成します。4096bitのRSAにしています。

.. code:: console

   mkdir private

.. code:: console

   openssl genrsa -out private/my-ca.key.pem 4096

.. note::

   認証局の秘密鍵の内容を確認するには以下のコマンドを実行します。

   .. code:: console

      openssl rsa -in private/my-ca.key.pem -text

上で生成した秘密鍵を使って認証局の自己証明書を作成します。

.. code:: console

   openssl req -new -sha256 -x509 -key private/my-ca.key.pem \
        -days 365 \
        -subj "/C=JP/ST=Osaka/L=Osaka City/O=Example, Inc./CN=my-ca.example.com" \
        -out my-ca.crt.pem

.. note::

   認証局の証明書の内容を確認するには以下のコマンドを実行します。

   .. code:: console

      openssl x509 -in my-ca.crt.pem -text

認証局用の設定ファイル :code:`my-ca.cnf` を作成します。
メッセージダイジェストのアルゴリズムはsha256にしています。

.. code:: console

   cat <<'EOF' > my-ca.cnf
   # we use 'ca' as the default section because we're usign the ca command
   [ ca ]
   default_ca = my_ca

   [ my_ca ]
   #  a text file containing the next serial number to use in hex. Mandatory.
   #  This file must be present and contain a valid serial number.
   serial = ./serial

   # the text database file to use. Mandatory. This file must be present though
   # initially it will be empty.
   database = ./index.txt

   # specifies the directory where new certificates will be placed. Mandatory.
   new_certs_dir = ./newcerts

   # the file containing the CA certificate. Mandatory
   certificate = ./my-ca.crt.pem

   # the file contaning the CA private key. Mandatory
   private_key = ./private/my-ca.key.pem

   # the message digest algorithm. Remember to not use MD5
   default_md = sha256

   # for how many days will the signed certificate be valid
   default_days = 365

   # a section with a set of variables corresponding to DN fields
   policy = my_policy

   [ my_policy ]
   # if the value is "match" then the field value must match the same field in the
   # CA certificate. If the value is "supplied" then it must be present.
   # Optional means it may be present. Any fields not mentioned are silently
   # deleted.
   countryName = match
   stateOrProvinceName = supplied
   organizationName = supplied
   commonName = supplied
   organizationalUnitName = optional
   commonName = supplied
   EOF

認証局の動作に必要なファイルとディレクトリを作成します。

.. code:: console

   mkdir newcerts

.. code:: console

   touch index.txt

.. code:: console

   echo 'unique_subject = yes' > index.txt.attr

.. code:: console

   echo '01' > serial

自己認証局を使ってサーバ証明書を発行
=====================================

まず上記で作成した :code:`myCA` ディレクトリに移動しておきます。

まずサーバ証明書を保存する作業ディレクトリを作成します。

.. code:: console

   mkdir ../server-certs

サーバの秘密鍵を生成します。4096bitのRSAにしています。

.. code:: console

   openssl genrsa -out ../server-certs/sv01.example.com.key.pem 4096

.. note::

   サーバの秘密鍵の内容を確認するには以下のコマンドを実行します。

   .. code:: console

      openssl rsa -in ../server-certs/sv01.example.com.key.pem -text

自己認証局に証明書の署名を依頼するためにCSR (Certificate Siging Request) を作成します。

.. code:: console

   openssl req -new -key ../server-certs/sv01.example.com.key.pem \
     -sha256 \
     -subj "/C=JP/ST=Osaka/L=Osaka City/O=Example, Inc./CN=sv01.example.com" \
     -out ../server-certs/sv01.example.com.csr.pem

.. note::

   CSRの内容を確認するには以下のコマンドを実行します。

   .. code:: console

      openssl req -in ../server-certs/sv01.example.com.csr.pem -text

自己認証局で証明書を発行し署名します。

.. code:: console

   openssl ca -batch -config my-ca.cnf \
     -days 365 \
     -notext \
     -out ../server-certs/sv01.example.com.crt.pem \
     -infiles ../server-certs/sv01.example.com.csr.pem

.. note::

   証明書の内容を確認するには以下のコマンドを実行します。

   .. code:: console

      openssl x509 -in ../server-certs/sv01.example.com.crt.pem -text

   また証明書を発行すると :code:`serial` ファイル内のシリアル番号が1増加され、:code:`index.txt` というファイルに発行したときのサブジェクト（:code:`-subj` オプションで指定した値）と発行時のシリアル番号などが記録されます。

   上記で :code:`index.txt.attr` に :code:`unique_subject = yes` と指定したので、同じサブジェクトで再度証明書を発行しようとすると :code:`index.txt` の中身をチェックしてエラーが出るようになります。
