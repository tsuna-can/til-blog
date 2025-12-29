---
date: '2025-12-28T13:43:07+09:00'
draft: false
title: 'SAML Authentication'
---

## SAML 認証とは
SAML（Security Assertion Markup Language）は、XML ベースの認証情報を安全に交換するための規格。  
SSO の実装の 1 つ。  

## SAML 認証の構成要素
IdP、SP、Principal の3つの構成要素が存在する。

### IdP（Identity Provider）
ユーザーの認証を担当し、認証情報を提供する側のシステム。  
認証が成功すると、ユーザー情報を含む SAML アサーション（XML ベースのトークン）を SP へ連携する。  
Okta や Auth0 がこれにあたる。

### SP（Service Provider）
SP（Service Provider）は、ユーザーに実際のサービスやリソースを提供する側のシステム。  
IdP から受け取った SAML Response に含まれる署名や有効期間を検証し、問題なければサービスを提供する。

### Principal
認証の対象となるエンティティ。  
通常はエンドユーザーであるが、アプリケーションやシステムプロセスがこれに当たる場合もある。

## SAML メッセージ
SAML メッセージとは SAML 参加者間でやり取りされる XML 形式のデータのこと。  
以下の他にも多くのメッセージがある。

### SAML AuthnRequest
SAML AuthnRequest は SP から IdP に送信される認証リクエスト。  

### SAML Response
Response は IdP から SP に返される認証結果。  
認証成功時には必ず Assertion が含まれる。  
Assertion は認証情報や属性情報を SP に伝えるための署名付きの XML 形式のデータのこと。

## SAML 認証の流れ
SAML では最初に SP と IdP のどちらにアクセスするかで 2 つの開始パターンがある。

### SP initiated
SP（Service Provider）からアクセスを開始するパターン。  
ユーザーが直接 SP にアクセスし、SP が認証が必要と判断して IdP へリダイレクトする。

{{< mermaid >}}
sequenceDiagram
    actor User as ユーザー
    participant Browser as ブラウザ
    participant SP as SP
    participant IdP as IdP

    User->>Browser: 1. サービスにアクセス
    Browser->>SP: 2. リクエスト送信
    SP->>SP: 3. 未認証を検知
    SP->>Browser: 4. SAML AuthnRequest を生成<br/>IdP へリダイレクト指示
    Browser->>IdP: 5. SAML AuthnRequest を送信
    IdP->>Browser: 6. ログイン画面を表示
    Browser->>User: 7. ログイン画面表示
    User->>Browser: 8. 認証情報を入力
    Browser->>IdP: 9. 認証情報を送信
    IdP->>IdP: 10. 認証処理
    IdP->>Browser: 11. SAML Response を生成<br/>SP へリダイレクト指示
    Browser->>SP: 12. SAML Response を送信
    SP->>SP: 13. SAML Response を検証
    SP->>Browser: 14. アクセス許可
    Browser->>User: 15. サービス画面表示
{{< /mermaid >}}

### IdP initiated
IdP（Identity Provider）からアクセスを開始するパターン。  
ユーザーが IdP ポータルにログインし、そこから目的の SP を選択してアクセスする。

{{< mermaid >}}
sequenceDiagram
    actor User as ユーザー
    participant Browser as ブラウザ
    participant IdP as IdP
    participant SP as SP

    User->>Browser: 1. IdP ポータルにアクセス
    Browser->>IdP: 2. リクエスト送信
    IdP->>Browser: 3. ログイン画面を表示
    Browser->>User: 4. ログイン画面表示
    User->>Browser: 5. 認証情報を入力
    Browser->>IdP: 6. 認証情報を送信
    IdP->>IdP: 7. 認証処理
    IdP->>Browser: 8. 利用可能なサービス一覧を表示
    Browser->>User: 9. サービス一覧表示
    User->>Browser: 10. 目的の SP を選択
    Browser->>IdP: 11. SP 選択を送信
    IdP->>Browser: 12. SAML Response を生成<br/>SP へリダイレクト指示
    Browser->>SP: 13. SAML Response を送信
    SP->>SP: 14. SAML Response を検証
    SP->>Browser: 15. アクセス許可
    Browser->>User: 16. サービス画面表示
{{< /mermaid >}}

## SAML 認証のメリット・デメリット
- メリット
  - 2000年代初頭から採用されており、エンタープライズ標準として成熟している
  - XML 署名と暗号化が標準仕様に含まれている
- デメリット
  - IdP が停止すると連携している全システムにアクセスできなくなる
  - ブラウザリダイレクト前提の設計でモバイルや SPA との相性が悪い
 
