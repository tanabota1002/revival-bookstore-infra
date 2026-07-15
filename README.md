# revival-bookstore-infra

RevivalBookStoreNeo用のネットワーク層CloudFormationテンプレート（[network.yaml](./network.yaml)）の説明。

## トップレベルのタグ

CloudFormationテンプレートは、必ずこの5つのタグ（キー）の組み合わせでできている。

| タグ | 役割 |
|---|---|
| `AWSTemplateFormatVersion` | テンプレート形式のバージョン。実質固定値`"2010-09-09"`を書くだけの決まり文句。 |
| `Description` | このテンプレートが何を作るかの説明文。AWSコンソールのスタック一覧に表示される。 |
| `Parameters` | デプロイ時に外から値を渡せる「変数」の定義。同じテンプレートを`dev`用・`prd`用など使い回すために使う。 |
| `Resources` | **実際に作られるAWSリソースの本体。ここが一番重要。** 1つ1つが「論理ID: リソース定義」の形になっている。 |

> `Outputs`（他スタックへの値の受け渡し）は、ECS/RDSも含めて全部このテンプレート1つにまとめる方針のため、今回は使っていない。スタックを分割する必要が出てきたら追加する。

---

## Parameters

| パラメータ | 説明 |
|---|---|
| `ProjectName` | リソース名の先頭に付ける識別子（デフォルト`revival-bookstore`）。タグ名や名前の重複を避けるための接頭辞。 |
| `Environment` | `dev`か`prd`を選ぶ。同じテンプレートで開発環境と本番環境を分けて作るためのスイッチ。 |

---

## Resources（リソースの種類ごとの説明）

`Resources`の中の各項目は `Type: AWS::サービス名::リソース種別` を持つ。このテンプレートで使っているリソース種別は以下の9種類。

| Type（タグ） | このテンプレートでの論理ID | 何を作るか |
|---|---|---|
| `AWS::EC2::VPC` | `VPC` | ネットワークの箱そのもの。`10.0.0.0/16`という約6万5千個のIPアドレス範囲を持つ、自分専用の仮想ネットワーク。 |
| `AWS::EC2::Subnet` | `PublicSubnet1/2`<br>`PrivateSubnet1/2` | VPCを分割した区画。`AvailabilityZone`でどの物理データセンターに置くか、`MapPublicIpOnLaunch`でパブリックIPを自動付与するかを指定する。パブリックかプライベートかはこのリソース自体には書かれておらず、後述のルートテーブルとの紐付けで決まる。 |
| `AWS::EC2::InternetGateway` | `InternetGateway` | VPCとインターネットをつなぐ「玄関口」の部品そのもの。単体では何にも繋がっていない。 |
| `AWS::EC2::VPCGatewayAttachment` | `AttachGateway` | 上のInternetGatewayを実際にVPCへ取り付ける作業。これをしないとIGWは機能しない。 |
| `AWS::EC2::EIP` | `NatEip` | 固定のパブリックIPアドレス1個を確保する。NATゲートウェイに割り当てるために必要。 |
| `AWS::EC2::NatGateway` | `NatGateway` | プライベートサブネットから外部への「一方通行の出口」。パブリックサブネットに置き、EIPを紐付けて使う。 |
| `AWS::EC2::RouteTable` | `PublicRouteTable`<br>`PrivateRouteTable` | 「0.0.0.0/0（どこでも）宛の通信をどこに送るか」を決める経路表そのもの（まだ中身は空）。 |
| `AWS::EC2::Route` | `PublicRoute`（→IGW）<br>`PrivateRoute`（→NAT） | ルートテーブルの中身（実際の経路）を1本追加する。ここでパブリック用はIGWへ、プライベート用はNATゲートウェイへと分岐している。 |
| `AWS::EC2::SubnetRouteTableAssociation` | `Public/PrivateSubnet[1/2]RouteAssoc` | 「どのサブネットにどのルートテーブルを適用するか」の紐付け。この紐付けによって、サブネットが実質的に「パブリック」「プライベート」になる。 |
| `AWS::EC2::SecurityGroup` | `AlbSG` / `EcsSG` / `RdsSG` | インスタンス単位のファイアウォール。`SecurityGroupIngress`で「誰からの通信を許可するか」を定義する。`CidrIp`はIPアドレス指定、`SourceSecurityGroupId`は「そのSGが付いているものからだけ許可」という指定。 |

### セキュリティグループの許可関係（一本道になっている）

```
インターネット
    │ (80番, 誰でもOK)
    ▼
  AlbSG
    │ (8080番, AlbSGが付いているものだけ)
    ▼
  EcsSG
    │ (5432番, EcsSGが付いているものだけ)
    ▼
  RdsSG
```

ECSは「ALBからしか話しかけられない」、RDSは「ECSからしか話しかけられない」という多層防御になっている。

このあとECS/ALB/RDSのリソースを追加する場合も、同じテンプレート内なら`!Ref VPC`や`!Ref PrivateSubnet1`のように直接参照すればよく、`Outputs`は不要。
