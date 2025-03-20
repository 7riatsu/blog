---
title: "MySQL の LockWaitTimeout エラーを調査する方法"
date: 2025-03-20T12:22:34+09:00
author: "Atsuki Narita"
description: ""
tags: ["DB", "MySQL"]
draft: false
share: true
---
<!--more-->

## はじめに

MySQL を使用している環境で、LockWaitTimeout エラーが発生して困った経験はないでしょうか。
LockWaitTimeout エラーは「あるプロセスが長時間ロックを保持 → 他のプロセスがロックを取得できずにタイムアウトする」という流れで発生することが多いです。

こうした問題を根本的に解決するには「長時間ロックを保持しているトランザクションを特定する」ことが第一歩となります。
しかし、実行中のクエリであればロギングして原因調査しやすいものの、クエリが完了してしまうと追跡および調査が難しくなります。

そこで、実行時間が一定の閾値（今回は 30 秒）を超えたトランザクションを検知し、ログとして残す仕組みを用意しておけば、発生時点のトランザクション情報を確認できるため、原因特定が非常にスムーズになります。

## 調査アプローチの概要

1. 長時間実行中のトランザクションを検知してログを残すクラス（e.g. LongTransactionLogger）を用意。
2. 10 秒間隔など、一定周期で LongTransactionLogger を呼び出すジョブをバックグラウンドで実行。
3. 30 秒以上実行中のトランザクションがあれば、その情報をアプリケーションログに出力する。
4. LockWaitTimeout エラー発生時にログを確認すれば、どのトランザクションが長時間ロックを保持していたか調査が可能。

この仕組みにより、LockWaitTimeout エラーの原因を迅速に追跡・対策できるようになります。

## 実装例

下記は、Ruby on Rails 環境で ActiveRecord を利用しているケースを想定したサンプルコードです。
information_schema.innodb_trx や performance_schema テーブルを参照し、実行中のトランザクションを取得しています。

long_running_transactions_sql メソッドの箇所だけ流用すれば Rails 以外のどの言語でも同様の結果を得られます。参考にしてみてください。

```ruby
# frozen_string_literal: true

class LongTransactionLogger
  LONG_RUNNING_THRESHOLD_SECONDS = 30
  private_constant :LONG_RUNNING_THRESHOLD_SECONDS

  TrxStruct = Struct.new(
    :trx_id, :trx_mysql_thread_id, :trx_query, :trx_started, :duration,
    :sql_text, :event_name, :event_id, :lock_time, keyword_init: true
  )

  def check_and_save_log
    log_transactions(long_running_transactions)
  end

  private

  def long_running_transactions
    results = ActiveRecord::Base.connection.exec_query(long_running_transactions_sql)
    results.map do |row|
      TrxStruct.new(
        trx_id: row['trx_id'],
        trx_mysql_thread_id: row['trx_mysql_thread_id'],
        trx_query: row['trx_query'],
        trx_started: row['trx_started'],
        duration: row['duration'],
        sql_text: row['SQL_TEXT'],
        event_name: row['EVENT_NAME'],
        event_id: row['EVENT_ID'],
        lock_time: row['LOCK_TIME']
      )
    end.map(&:to_h)
  end

  def long_running_transactions_sql
    <<-SQL.squish
      SELECT
        it.trx_id, it.trx_mysql_thread_id, it.trx_query, it.trx_started,
        TIMESTAMPDIFF(SECOND, it.trx_started, NOW()) AS duration,
        esc.SQL_TEXT, esc.EVENT_NAME, esc.EVENT_ID, esc.LOCK_TIME
      FROM
        information_schema.innodb_trx it
      LEFT JOIN performance_schema.threads pt
        ON it.trx_mysql_thread_id = pt.PROCESSLIST_ID
      LEFT JOIN performance_schema.events_statements_history esc
        ON pt.THREAD_ID = esc.THREAD_ID
      WHERE
        TIMESTAMPDIFF(SECOND, it.trx_started, NOW()) > #{LONG_RUNNING_THRESHOLD_SECONDS}
      ORDER BY duration DESC
      LIMIT 100
    SQL
  end

  def log_transactions(transactions)
    if transactions.any?
      Rails.logger.info(
        event: 'long_running_transactions',
        message: 'Detected long-running transactions',
        transactions: transactions
      )
    else
      # ここはログが埋め尽くされる要因になるため不要な場合は除外してください
      Rails.logger.info(
        event: 'long_running_transactions',
        message: 'No long-running transactions detected'
      )
    end
  end
end
```

innodb_trx から実行中のトランザクションを取得しつつ、threads, events_statements_history と結合してロック時間や実行中 SQL の詳細を補足しています。
30 秒を超えるトランザクションがあれば情報をまとめてログに出力します。

sql_text カラムから長時間ロックを保持しているクエリを特定します。ロックを長時間保持しているクエリさえ特定できれば、あとはアプリケーションレイヤーで問題解決を試みるだけです。
クエリの見直しや改善、インデックス最適化、トランザクションのスコープの縮小、テーブル設計やロック粒度の最適化あたりを中心に見直すと根本解決につながる場合が多いでしょう。　

## 参照しているテーブルとカラムの解説

### information_schema.innodb_trx テーブル

- **innodb_trx** テーブルは、現在実行中（コミットやロールバックが完了していない）の InnoDB トランザクションに関する情報を提供します。
  - trx_id: トランザクション ID
  - trx_mysql_thread_id: トランザクションが実行されている MySQL スレッド ID
  - trx_query: 実行中（または直近に実行した）クエリ
  - trx_started: トランザクションが開始された日時

InnoDB で実行中のトランザクションすべてが格納されているので、まずはここから「実行中のクエリ」や「開始時間」などを取得して、どれだけ長く動いているかを確認できます。

- [『MySQL 公式ドキュメント INFORMATION_SCHEMA INNODB_TRX テーブル』](https://dev.mysql.com/doc/refman/8.0/ja/information-schema-innodb-trx-table.html)

### performance_schema.threads テーブル

- **threads** テーブルは、MySQL の各スレッドに関する情報を保持しています。
  - THREAD_ID: 内部で管理されるスレッドごとのユニークな ID
  - PROCESSLIST_ID: MySQL の従来のスレッド ID（SHOW PROCESSLIST などで表示される）

innodb_trx が持つ trx_mysql_thread_id と threads の PROCESSLIST_ID を結合することで、各トランザクションを実行している MySQL スレッドにひも付けられます。

- [『MySQL 公式ドキュメント スレッドテーブル』](https://dev.mysql.com/doc/refman/8.0/ja/performance-schema-threads-table.html)

### performance_schema.events_statements_history テーブル

- **events_statements_history** テーブルは、ある程度直近に実行されたステートメント（SQL 文）に関するイベント情報が履歴として格納されています。
  - THREAD_ID: ステートメントを実行したスレッドを示す
  - SQL_TEXT: 実行された SQL の内容
  - EVENT_NAME, EVENT_ID: 実行イベント自体を示す情報
  - LOCK_TIME: SQL 文が行ロックなどを保持していた時間

ステートメントの履歴は大規模環境だとすぐに上書きされる可能性がありますが、長時間トランザクションであればまだ履歴に残っているため、このテーブル経由で実際の SQL やロック時間をチェックできます。

- [『MySQL 公式ドキュメント events_statements_history テーブル』](https://dev.mysql.com/doc/refman/8.0/ja/performance-schema-events-statements-history-table.html)
- [『MySQL 公式ドキュメント events_statements_current テーブル](https://dev.mysql.com/doc/refman/8.0/ja/performance-schema-events-statements-current-table.html)

## ロギング時の工夫

### 1. 構造化ログを活用
  - JSON 形式などの構造化ログにしておけば、後から解析ツール（例：Datadog, Fluentd, Elasticsearch, BigQuery）でフィルタリング・検索しやすくなります。アプリケーションログにこのまま出力しても良いですが、transactions: の部分を JSON 化するなど、可能な限り機械可読性を高めると分析効率が向上します。

### 2. 必要なメタデータの付与
  - ログには「イベント種別」「エラーコード」「SQL 実行開始時刻」などのメタデータを付与すると、問題発生時のトラブルシューティングが容易になります。たとえば environment（本番・ステージング等）や host（DB サーバー名）を入れておくのも有効です。

### 3. ログサイズに注意する
  - 長時間トランザクションが非常に多い場合、ログが膨大になる恐れがあります。必要な情報だけを抽出し、それ以外はサマリーにとどめるなど、ログの肥大化を防ぐ工夫をしましょう。

## 注意点・運用上のポイント

### 1. performance_schema 有効化とオーバーヘッド
  - performance_schema が有効になっていない場合は設定変更が必要です。また、大量の SQL を捌く環境ではオーバーヘッドが増える可能性があります。必要に応じて監視粒度（performance_schema 設定）を調整してください。

### 2. 定期実行タスクの停止タイミング
  - トラブルシュートが完了したら、不要に長時間タスクを走らせないようにしましょう。フィーチャーフラグなどでオン・オフを切り替えられると便利です。

### 3. 実行頻度の考慮
  - 監視を細かくしすぎると逆にサーバー負荷が上がる可能性があります。一方、大雑把すぎると見逃しが発生するかもしれません。環境の負荷状況や許容範囲に応じて、適切な監視間隔を設定しましょう。

### 4. セキュリティポリシーの確認
  - 実行クエリがログに出力されるので、そもそも出力して OK なのか判断してください。また、必要に応じてマスキング対応をするなどの工夫があると良いかもしれません。
