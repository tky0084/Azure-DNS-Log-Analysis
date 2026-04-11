# Azure-DNS-Log-Analysis
Azure のDNS Private Resolverのログを取得する

##  概要
はじめに
AzureのHub & Spokeネットワーク構成において、ネットワークの通信やFirewallの許可の状況については、
仮想ネットワークフローログやAzureFirewallの診断設定ログにて分析することが可能である。
しかし、これらのログにおいて、AzureのマネージドDNSサービスの一つである、Azure DNSPrivate Resolverについては盲点があり、
このリゾルバーによる名前解決通信ログが表示されない。
これでは、Azureのネットワークトラブルシューティングにおいて、名前解決の観点で調査ができなくなることが考えられる。
本記事では、その原因と対策について記載する。

### 想定するネットワーク環境
今回は、AzureのHub & Spokeで構成されたネットワークと、疑似的にオンプレミスを模したネットワークが接続された環境を想定する。
- オンプレミスには「test.co.jp」というドメインをもつActive Directoryのサーバーを配置し、実質DNSサーバーの役割を持たせることとする。
- プライベートの名前解決は、Azureからの要求も含めて、すべてこのActive Directoryで行う。
- Azure側のネットワークとは、VPN Gatewayでの接続を実施する
### Azure側
- 1つのHubと1つのSpokeで構成する。
- Hub環境には、AzureFirewallを配置し、Spokeからのすべての通信はルートテーブルによって、このAzureFirewallをネクストホップで経由する。
- Hub環境には、Azure DNSPrivate Resolverを置き、ネットワークの中に「Inbound Endpoint」および「OutboudEndpoint」を配置する。
- Spoke環境からの名前解決は、既定のDNSサーバーをHubのInbound Endpointにむけるよう仮想ネットワークのパラメータに設定する。
- オンプレミス側のネットワークとは、VPN Gatewayでの接続を実施する。
- Log Analyticsワークスペースを1つ配置する。
- HubのAzureFirewallの診断設定ログをこのLog Analytisワークスペースに保存する
- また、Hub側の仮想ネットワークフローログを作成するため、ストレージアカウントを作成し、仮想ネットワークフローログを作成して上記Log Analytisワークスペースへ格納する。

### なぜ名前解決のログが出力されないのか
Microsoft learnに記載のある通り、Azure DNSPrivate Resolverは仮想ネットワークフローログでサポートされていない
