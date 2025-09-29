:meta:
   :description: Oracle Databaseの主要なセキュリティ機能を、実際に手を動かしながら学べるチュートリアルサイトです。
   :keywords: Oracle Database, security, TDE, VPD, SQL Firewall, Database Vault, Data Redaction, チュートリアル
   :og:title: Oracle DB Security Tutorial
   :og:description: Oracle Databaseの主要なセキュリティ機能を、実際に手を動かしながら学べるチュートリアルサイトです。
   :og:url: https://koi141.github.io/demo-dbsec/
   :og:site_name: Oracle DB Security Tutorial


############################################
Oracle DB Security Tutorial
############################################

.. raw:: html

   <p align="left">
      <a href="https://github.com/koi141/demo-dbsec">
         <img src="https://img.shields.io/github/last-commit/koi141/demo-dbsec" alt="Last Commit">
      </a>
      <a href="https://github.com/koi141/demo-dbsec">
         <img src="https://img.shields.io/github/commit-activity/w/koi141/demo-dbsec" alt="Commit Activity">
      </a>
   </p>

Oracle Databaseのセキュリティ機能を簡単に試してみるチュートリアルを紹介するサイトです。


.. note::

   | このサイトで紹介する手順では、実行結果の一部を見やすくするために整形や省略を行っています。
   | そのため、実際の結果とは若干異なる場合がありますのでご了承ください。


.. grid:: 1

   .. grid-item-card::

      .. toctree::
         :maxdepth: 1
         :caption: 環境を準備する
         :glob:

         /setupEnv/*

.. grid:: 2
   :gutter: 2

   .. grid-item-card::
      :padding: 1

      .. toctree::
         :maxdepth: 1
         :caption: 透過的データ暗号化 (TDE)
         :glob:
         
         /tde/*

   .. grid-item-card::
      :padding: 1
      
      .. toctree::
         :maxdepth: 1
         :caption: ネイティブ・ネットワーク暗号化
         :glob:
         
         /nne/*

   .. grid-item-card::
      :padding: 1

      .. toctree::
         :maxdepth: 1
         :caption: Data Redaction
         :glob:
         
         /redact/*

   .. grid-item-card::
      :padding: 1

      .. toctree::
         :maxdepth: 1
         :caption: Virtual Private Database
         :glob:
         
         /vpd/*

   .. grid-item-card::
      :padding: 1

      .. toctree::
         :maxdepth: 1
         :caption: SQL Firewall
         :glob:
         
         /sqlfirewall/*

   .. grid-item-card::
      :padding: 1

      .. toctree::
         :maxdepth: 1
         :caption: Database Vault
         :glob:
         
         /dbv/*

   .. grid-item-card::
      :padding: 1

      .. toctree::
         :maxdepth: 1
         :caption: Oracle Label Security
         :glob:
         
         /ols/*
   
   .. grid-item-card::
      :padding: 1

      .. toctree::
         :maxdepth: 1
         :caption: Oracle Data Safe（準備中）

         /datasafe/1_setup.rst         

   .. grid-item-card::
         :padding: 1

         .. toctree::
            :maxdepth: 1
            :caption: グローバル認証（準備中）

            /globalAuthn/*         

   .. grid-item-card::
         :padding: 1

         .. toctree::
            :maxdepth: 1
            :caption: 統合監査（準備中）
            :glob:

            /unifiedAudit/*  