# Azure-DNS-Log-Analysis
Azure のDNS Private Resolverのログを取得する

## 概要

- Azure の Hub & Spoke ネットワーク構成において、ネットワーク通信や Firewall の許可状況については、仮想ネットワークフローログや Azure Firewall の診断設定ログで分析できる。  
- しかし、これらのログでは、Azure のマネージド DNS サービスの 1 つである Azure DNS Private Resolver に盲点があり、仮想ネットワークフローログでは DNS Private Resolver はサポートされていないため、仮想ネットワークフローログでは名前解決のログが正しく表示されない。  
- そのため、Azure のネットワークトラブルシューティングにおいて、名前解決の観点での調査が困難になる可能性がある。
- 本記事では、その原因と対策について記載する。

## 想定するネットワーク環境
### 全体構成
今回は、Azure の Hub & Spoke で構成されたネットワークと、疑似的なオンプレミス環境を模したネットワークが接続された構成を想定する。

### Azure側
- 1 つの Hub と 1 つの Spoke で構成する。
- Hub 環境には Azure Firewall を配置し、Spoke からのすべての通信はルートテーブルによって Azure Firewall をネクストホップとして経由させる。
- Hub 環境には Azure DNS Private Resolver を配置し、ネットワーク内に `Inbound Endpoint` および `Outbound Endpoint` を構成する。
- Spoke 環境の名前解決は、既定の DNS サーバーを Hub の Inbound Endpoint に向けるよう、仮想ネットワークの設定で構成する。
- Hub 側では、Spoke 側からの名前解決要求を Inbound Endpoint で受信し、オンプレミス側の AD サーバーへフォワード（転送）する DNS 転送ルールセットを定義する。
- オンプレミス側ネットワークとは VPN Gateway で接続する。
- Log Analytics ワークスペースを 1 つ配置する。
- Hub 側の仮想ネットワークフローログを作成するため、ストレージアカウントを作成し、仮想ネットワークフローログを Log Analytics ワークスペースへ格納する。
- ただし、仮想ネットワークフローログでは DNS Private Resolver はサポートされていないため、DNS セキュリティポリシーの診断設定ログを Log Analytics ワークスペースへ格納する。
### オンプレミス側
- オンプレミスの仮想マシン 2 台には、「test.co.jp」というドメインを持つ Active Directory サーバーを配置し、DNS サーバーとして利用する。
- プライベート名前解決は、Azure からの要求も含め、すべてこの Active Directory で実施する。
- Azure 側ネットワークとは VPN Gateway で接続する。

![alt text](/img/architecture.png)

## 環境構築

## Log Analytics ワークスペースから DNS クエリログを確認する

Spoke 環境の仮想マシンに接続し、PowerShell から `Resolve-DnsName` コマンドで作成したドメインの名前解決を実行する。  <br>
ドメインに 2 台の AD サーバーが紐づいていることが確認できる。

![alt text](</img/スクリーンショット 2026-05-06 001410.png>)

<br>

作成した Log Analytics ワークスペースの画面で、左ペインから「ログ」を選択し、以下のクエリを実行する。

```kusto
DNSQueryLogs
| where QueryName == "test.co.jp"
| project TimeGenerated, OperationName, QueryType, SourceIpAddress, DestinationIpAddress
```

![alt text](</img/スクリーンショット 2026-05-06 010750.png>)

<br>

> **ログの結果について**  
> `SourceIpAddress`（Spoke 環境の VM）から `OperationName`（ドメイン）への名前解決要求が送信され、その要求が `DestinationIpAddress`（AD サーバー 2 台）へ転送されていることが確認できる。  
>
> これは、Hub 構築時に作成した DNS 転送ルールセットにおいて、すべてのドメインの名前解決を AD サーバー 2 台へ転送するよう定義し、そのルールセットを Hub ネットワークおよび DNS Private Resolver に関連付けているためである。

---

## 【検証】フォワード先からセカンダリを削除する

DNS 転送ルールセットの設定を変更し、フォワード先からセカンダリ AD サーバーを削除する。<br>
作成したルールを「編集」し、セカンダリ AD サーバーを削除して保存する。

![alt text](</img/スクリーンショット 2026-05-06 010854.png>)

### 【結果】
再度、Spoke 仮想マシンから名前解決を実行し、Log Analytics ワークスペースでログを確認する。<br>
名前解決要求の転送先は、プライマリ AD サーバーのみになっていることが確認できる。<br>
一方、クエリ結果として `test.co.jp` は、依然としてプライマリおよびセカンダリの AD サーバーに紐づいている。<br>
このことから、AD サーバーを廃止する場合などは、元の DNS サーバー（オンプレミス側）のレコードからセカンダリ情報を削除する必要がある。<br>

![alt text](</img/スクリーンショット 2026-05-06 011251.png>)

---

## 【検証】ドメインからセカンダリ AD サーバーの紐づけを解除する

今回は、オンプレミス側の AD サーバーに権威 DNS サーバーが存在している。<br>
そのため、AD 側の DNS サーバーからレコードを削除する。<br>

> **注意**  
> 実際の環境で実施する場合は、影響範囲や詳細な手順を十分確認すること。  
> 本記事では、名前解決の挙動確認を目的としている。

プライマリ AD サーバーに接続し、Server Manager の右上から「Tools」→「DNS」を開く。<br>
ドメイン内の「same as parent folder」にある、セカンダリ AD サーバーの IP アドレスが設定されたレコードを右クリックし、「Delete」を選択して削除する。

![alt text](</img/スクリーンショット 2026-05-06 021622.png>)

<br>

再度、Spoke の VM から名前解決を実行し、ログを確認する。<br>
名前解決結果として、プライマリサーバーのみが紐づき、セカンダリサーバーの紐づけが削除されたことが確認できる。

![alt text](</img/スクリーンショット 2026-05-06 021803.png>)

---

## まとめ

- 以上、仮想ネットワークフローログではサポートされていない DNS ログを確認する方法を紹介した。  
- ログ取得方法だけでなく、DNS 転送や名前解決要求の結果がどのように変化するかも確認できる。  
- DNS セキュリティポリシー自体の料金は、100 万クエリあたり 100 円未満であり、高額ではない。  
- ただし、クエリログを診断設定で保存する場合は、Log Analytics Workspace（LAW）のログ料金が増加する点に注意が必要である。<br>
https://azure.microsoft.com/ja-jp/pricing/calculator/?msockid=039fabe1d331667f3eafbcbdd279673e