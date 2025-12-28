---
date: "2025-11-30T14:04:54+09:00"
draft: false 
title: "NAT と NAPT と NAT-T"
---

- NAT : プライベートIPアドレスとグローバルIPアドレスを変換する技術全般
- NAPT : NAT の一種で IP アドレスに加えてポート番号も変換する技術
- NAT-T : NAT 環境で IPsec を使うための補助技術

## NAT（Network Address Translation）

NAT はプライベート IP アドレスと グローバル IP アドレスを相互変換する技術全般のこと。  
NAT には大きく分けて以下の3種類がある。
- 静的 NAT(Static NAT)
  - プライベートIPアドレスとグローバルIPアドレスを1対1で固定的にマッピング
  - 例：192.168.1.10 ↔ 203.0.113.5（常にこの対応）
- 動的 NAT (Dynamic NAT)
  - プールされた複数のグローバルIPアドレスから動的に割り当て
  - 1対1のマッピングだが、毎回同じグローバルIPとは限らない
  - 例：192.168.1.10 → 今回は203.0.113.5、次回は203.0.113.6
- NAPT (Network Address Port Translation)
  - 別名：PAT (Port Address Translation)、IP マスカレード
  - 1つのグローバルIPアドレスを複数のプライベートIPアドレスで共有

## NAPT（Network Address Port Translation）

NAPT は NAT の一種で、IP アドレスに加えて **ポート番号** を変換する技術。  
これによって 1 つのグローバル IP アドレスで複数の内部端末が同時に通信可能になる。  
IP マスカレードとも呼ばれる。

{{< mermaid >}}
sequenceDiagram
participant ClientA as 内部クライアント A<br/>192.168.1.10:50000
participant ClientB as 内部クライアント B<br/>192.168.1.11:50001
participant NAPT as NAPT ルータ
participant Server as Web サーバー<br/>203.0.113.100:80

    ClientA->>NAPT: 送信元: 192.168.1.10:50000<br/>宛先: 203.0.113.100:80
    Note over NAPT: NAPTテーブル記録<br/>192.168.1.10:50000 ⇔ 198.51.100.1:10001
    NAPT->>Server: 送信元: 198.51.100.1:10001<br/>宛先: 203.0.113.100:80

    ClientB->>NAPT: 送信元: 192.168.1.11:50001<br/>宛先: 203.0.113.100:80
    Note over NAPT: NAPTテーブル記録<br/>192.168.1.11:50001 ⇔ 198.51.100.1:10002
    NAPT->>Server: 送信元: 198.51.100.1:10002<br/>宛先: 203.0.113.100:80

    Server->>NAPT: 送信元: 203.0.113.100:80<br/>宛先: 198.51.100.1:10001
    Note over NAPT: テーブル参照<br/>198.51.100.1:10001 → 192.168.1.10:50000
    NAPT->>ClientA: 送信元: 203.0.113.100:80<br/>宛先: 192.168.1.10:50000

    Server->>NAPT: 送信元: 203.0.113.100:80<br/>宛先: 198.51.100.1:10002
    Note over NAPT: テーブル参照<br/>198.51.100.1:10002 → 192.168.1.11:50001
    NAPT->>ClientB: 送信元: 203.0.113.100:80<br/>宛先: 192.168.1.11:50001

{{< /mermaid >}}

## NAT-T（NAT Traversal）
NAT-T は ESP パケットを UDP でカプセル化して NAT 越え通信を可能にする技術。  

### 背景
通常の IPSec では NAT ルーターを経由すると問題が発生する。  
- AH の問題
  - IP ヘッダー全体の整合性をチェックするため、NAT による IPアドレス変換を「改ざん」と判断してしまう
- ESP の問題
  - ESP は IP ヘッダーの直後に暗号化されたペイロードを配置するため、TCP/UDP のポート情報がない。NAPT ルーターはポート番号でセッションを識別するため、ESP パケットを正しく処理できない（静的 NAT や動的 NAT では問題ない）

なお、NAT-T は ESP パケットのみを対象としており、AH には対応していない。  
AH は IP ヘッダーの整合性を検証するため、NAT による IP アドレス変換と根本的に互換性がないためである。

### NAT-T の仕組み
NAT-T は以下の方法でこの問題を解決します：
- UDP カプセル化：IPsec パケット（ESP）を UDP パケット（ポート 4500）で包む
- ポート番号の利用：NAT ルーターが UDP ポート番号で通信を識別できるようにする
- NAT 検出：通信開始時に経路上に NAT があるか自動検出し、必要な場合のみ NAT-T を有効化

### パケット構造の比較

**通常の ESP パケット（NAT-T なし）：**
```
+------------+------------+------------------+
| IP ヘッダー | ESP ヘッダー | 暗号化ペイロード |
+------------+------------+------------------+
```
→ ポート番号がないため、NAPT ルーターで識別不可

**NAT-T 使用時の ESP パケット：**
```
+------------+-------------+------------+------------------+
| IP ヘッダー | UDP ヘッダー | ESP ヘッダー | 暗号化ペイロード |
+------------+-------------+------------+------------------+
              (ポート4500)
```
→ UDP ヘッダーにポート番号があるため、NAPT ルーターで識別可能

### 動作フロー

{{< mermaid >}}
sequenceDiagram
    participant Client as VPN クライアント<br/>192.168.1.10
    participant NAT as NAT ルータ
    participant Server as VPN サーバー<br/>203.0.113.50

    Note over Client,Server: フェーズ1：IKE ネゴシエーション
    Client->>NAT: IKE (UDP 500)
    NAT->>Server: IKE (UDP 500, 送信元 IP/Port 変換)

    Server->>NAT: IKE 応答
    NAT->>Client: IKE 応答 (宛先 IP/Port 変換)

    Note over Client,Server: NAT 検出（ハッシュ値比較で NAT の存在を確認）

    Note over Client,Server: フェーズ2：IPsec SA 確立（UDP 4500 へ切り替え）
    Client->>NAT: ESP in UDP (UDP 4500)
    Note over NAT: NAT テーブル記録<br/>192.168.1.10:4500 ⇔ 198.51.100.1:15000
    NAT->>Server: ESP in UDP<br/>送信元: 198.51.100.1:15000<br/>宛先: 203.0.113.50:4500

    Server->>NAT: ESP in UDP 応答<br/>送信元: 203.0.113.50:4500<br/>宛先: 198.51.100.1:15000
    Note over NAT: テーブル参照<br/>198.51.100.1:15000 → 192.168.1.10:4500
    NAT->>Client: ESP in UDP (UDP 4500)

{{< /mermaid >}}

