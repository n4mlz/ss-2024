---
marp: true
paginate: true
theme: gaia
backgroundColor: #fff
backgroundImage: url('https://marp.app/assets/hero-background.jpg')
---

<!-- _class: lead -->
# OCI Runtime Spec に準拠した
# 非特権コンテナランタイムの開発

---

# 動機

- コンテナランタイム自作は簡単である
    - 50行程度で実装するチュートリアルもある
- しかし, 非特権で動作するコンテナランタイムを自作する例はまだ少ない
- 出来るだけシンプルで理解しやすいコードで, 非特権に動作するコンテナランタイムを自作する

➔ 自身の非特権コンテナへの理解と, コンテナランタイムの自作に関する参考資料としての活用を目指す

---

# 成果物 "tiny-runc"

- リポジトリ: https://github.com/n4mlz/tiny-runc
- OCI Runtime Spec に準拠した非特権コンテナランタイム
    - OCI Runtime Spec: コンテナランタイムの標準仕様
    - Docker や Podman が依存する低レベルコンテナランタイムもこれに準拠
- 既存の低レベルコンテナのデファクトスタンダードである runc を参考に Go 言語で実装

---

# デモ

- ArchLinux 上で tiny-runc により,  Ubuntu 22.04 のコンテナを起動するデモ
- ホスト名, ユーザ名, プロセスの PID 等がコンテナ内とホストで異なることを確認

<!-- <h1></h1>
<video src="tiny-runc-demo.mp4" width="80%" height="80%" controls muted>
</video> -->

---

<video src="demo.mp4" width="100%" height="100%" controls muted>
</video>

---

# コンテナに求められること

- <span style="font-size: 35px">5 つの原則: OCI Runtime Spec の定める [Standard Containers](https://github.com/opencontainers/runtime-spec/blob/main/principles.md)</span>
    <span style="font-size: 30px">
    1. Standard operations
    標準化されたコンテナの基本的な操作が行えること
    2. Content-agnostic
    コンテナの中身に関係なく, 同じ操作が同じ影響を与えること
    3. Infrastructure-agnostic
    インフラに関わらず同じように操作を行えること
    4. Designed for automation
    自動化を前提に設計されていること
    5. Industrial-grade delivery
    商用環境での利用に耐えうる信頼性を持つこと

---

# コンテナを実現する技術

tiny-runc ではこう解決しました

- Linux Namespace
    - PID, Network, Mount, User, UTS, IPC, ...
    - 名前空間を分離することで, プロセス間のリソースを隔離
    - `unshare` システムコールを使う
    - runc でも同様に名前空間を分離

---

# 非特権で困ること

- `unshare` システムコールを発行できない
    - `CAP_SYS_ADMIN` ケーパビリティが必要
- 非特権環境では, 特権環境での機能が使えない
    - 例: ホスト名の設定, ファイルシステムのマウント
    - こちらもケーパビリティの不足が原因

---

<!-- # 解決した方法: User Namespace の活用

- uid/gid の分離
- 唯一 root 権限を持たずに作成できる namespace
- uid/gid のマッピングを行うことで, コンテナ内で root 権限を持つことができる
    - 例: ホストの uid 1000 をコンテナ内の uid 0 にマッピングする
- 実際には `shadow-utils` パッケージの `newuidmap`, `newgidmap` バイナリを使用

--- -->

## 解決: 複数プロセスに分け, 段階的に権限を得る

- User Namespace を分離, 子プロセスを生成
- 親プロセスから uid/gid を 0 にマッピング
    - 名前空間内に閉じたいくつかのケーパビリティを得る
    - ここで `CAP_SYS_ADMIN` を得る
- 他の名前空間を分離
- それぞれの分離した名前空間を使用した設定
    - ホスト名, ファイルシステム, ネットワーク, ...
- 目的のプロセスを起動

---

# tiny-runc の実際の処理の流れ

<img src="figure.svg" width="95%" height="95%">

---

# 課題・今後の展望
- OCI Runtime Spec に完全には準拠していない
- 分離しきれていないリソース
    - ネットワークの設定
        - ネットワーク名前空間の分離
        - ブリッジデバイスの作成
        - ルーティングの設定
    - Cgroup の設定
        - リソース制限の設定

---

# まとめ

- OCI Runtime Spec に準拠した非特権コンテナランタイムを開発した
- コンテナランタイムの自作に関する参考資料としての活用を目指し、シンプルで理解しやすいコードで実装した
- 非特権に動作し、Standard Containers の 5 つの原則をある程度実現するランタイムを開発できた