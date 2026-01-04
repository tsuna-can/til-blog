---
date: '2025-12-19T23:37:52+09:00'
draft: false 
title: 'Kerberos Authentication'
---

## Kerberos 認証とは
Kerberos 認証はチケットベースでの SSO を実現するプロトコル。  
Windows Server Active Directory がユーザー認証で利用しているプロトコルとして有名。

## Kerberos 認証の用語
- Client
  - Client はサービスを利用したいユーザーやその PC
- KDC（Key Distribution Center）
  - KDC は認証システムの中核で AS と TGS の 2 つの機能をもつ
  - すべての認証情報を管理し、チケットを発行する
- AS（Authentication Server)
  - AS はユーザーの初回認証を行うサービス
  - ユーザー名とパスワードを確認し、成功すると TGT を発行する
- TGS（Ticket Granting Server）
  - TGS はサービスへのアクセスに必要なチケットを発行する
  - ユーザーが TGT を提示すると、要求されたサービス用のチケットを生成する
- TGT（Ticket Granting Ticket）
  - TGT はユーザーが認証済みであることを証明するチケット
  - これを使って複数のサービスチケットを取得することができる
- セッションキー
  - セッションキーは一時的な通信用の暗号鍵
  - AS からは TGS との通信用、TGS からは Service Server との通信用のセッションキーが発行される
  - Client 側でパスワードから生成した鍵で復号することで、サーバーの正当性も確認できる
- Service Server
  - Service Server はユーザーが利用したいリソースやサービスを提供するサーバー
- ST（Service Ticket）
  - Service Ticket は特定のサービスにアクセスするためのチケット
  - サービスごとに個別に発行される
- Principal
  - Principal は認証システムで識別される個々のエンティティのこと
  - ユーザーの場合、`username@REALM` のように表す
- Realm
  - Realm は認証管理の境界を定義する論理的な管理ドメイン
  - 同じ Realm に属するユーザーとサービスが認証を共有する
  - 例： `EXAMPLE.COM` など

## Kerberos 認証のフロー
Client が何と通信するかで大きく 3 段階に分けられる。

### 初回認証（AS との通信）
1. Client がユーザー ID を AS に送信（AS-REQ）
1. AS はクライアントを確認し、TGT とセッションキーを発行（AS-REP）
1. クライアント側でパスワードを入力して、セッションキーを復号する（これによって AS の正当性を Client側で検証可能）

### サービスチケットの取得（TGS との通信）
1. TGT、セッションキーで暗号化した認証情報、アクセスしたいサービス名を TGS に送信（TGS-REQ）
1. TGS は TGT を検証し、ST と新しいセッションキー（Service Server 用）を発行（TGS-REP）
1. ST は Service Server の秘密鍵で暗号化されている

### サービスへのアクセス（Service Server との通信）
1. Client が ST とセッションキーで暗号化した Authenticator（タイムスタンプなどを含む認証情報）を Service Server に送信（AP-REQ）
1. Service Server は自身の秘密鍵で ST を復号し、内容を検証
1. 検証が成功すれば、認証成功のレスポンスを返す（AP-REP）

{{< mermaid >}}
sequenceDiagram
    participant Client
    box KDC
        participant AS
        participant TGS
    end
    participant Service Server

    Client->>AS: ユーザー ID 送信
    AS->>Client: TGT とセッションキーの発行
    Note over Client: パスワード入力でセッションキーを復号

    Client->>TGS: TGT + セッションキーで暗号化した認証情報 + サービス名を送信
    TGS->>Client: ST（Service Server の鍵で暗号化）とセッションキー発行

    Client->>Service Server: ST + セッションキーで暗号化した Authenticator 送信
    Note over Service Server: チケット検証・認証
    Service Server->>Client: 認証成功レスポンス
{{< /mermaid >}}

また、2つ目以降のサービスにアクセスする際は、TGT が有効期限内であれば AS との通信は省略できる。  
その場合は、既存の TGT を使って TGS から新しい ST を取得する。  
これにより、パスワード入力は最初の 1 回だけで済む（SSO の実現）。

## Kerberos 認証のメリット・デメリット
- メリット
  - パスワードがネットワーク上を流れない（クライアント側でのみ使用され、鍵の生成に利用される）
  - KDC が中央集権的に認証情報を一元管理するので、ユーザーやサービスの追加・削除が容易
  - クライアントだけでなく、サーバー側も正当性を証明するフローがある
- デメリット
  - クライアント・KDC・Service Server 間で時刻同期が必要
  - KDC が単一障害点となる
  - チケットが盗まれると有効期限内は悪用される可能性がある（Pass-the-ticket 攻撃）

