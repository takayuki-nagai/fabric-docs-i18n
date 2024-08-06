# Performance considerations

Hyperledger Fabricのコンポーネント、構成、ワークフローの決定は、ネットワークの全体的なパフォーマンスに影響します。
チャネル数、チェーンコードの実装、トランザクションポリシーなど、これらの変数の管理は、
複数の参加組織がそれぞれのハードウェアやネットワークインフラをネットワーク環境に提供することで、複雑になる可能性があります。

このトピックでは、Hyperledger Fabricネットワークのパフォーマンスを最適化するために役立つ考慮事項について説明します。

## Hardware considerations

各参加組織は、ピアおよびオーダリングサービスノードをホスティングするためのハードウェアを提供することができます。
オーダリングサービスノードは、単一の組織が提供することも、複数の組織が提供することもできます。
どの組織もネットワーク全体のパフォーマンスに影響を与える可能性があるため（トランザクションのエンドースメントポリシーなどの要因による）、
各参加組織は、自らがデプロイするネットワークサービスとアプリケーションのために十分なリソースを提供しなければなりません。

### Persistent storage

Hyperledger Fabricネットワークは大量のディスクI/Oを実行するため、可能な限り高速なディスクストレージを使用する必要があります。
ネットワーク接続ストレージを使用する場合は、利用可能な最速のIOPSを選択するようにしてください。

### Network connectivity

Hyperledger Fabricネットワークは複数のノードに高度に分散され、
一般的に複数のクラスタ、異なるクラウド環境、さらには法的・地理的境界を越えてホストされます。
そのため、ノード間の高速ネットワーク接続は不可欠であり、すべてのノードと組織間で最低1Gbpsを確保する必要があります。

### CPU and memory

ハードウェアに関する最後の考慮事項は、ピアとオーダリングサービスノード、およびピアノードが使用するCouchDBステートデータベースに割り当てるCPUとメモリの量です。
これらの割り当ては、ネットワークパフォーマンス、さらにはノードの安定性に影響を与えるため、CPU、メモリ、ディスク容量、およびディスクとネットワークI/Oを継続的に監視し、
割り当てられたリソースの制限内にあることを確認する必要があります。
最大使用率に近づくまで、ノードのリソースを増やすのを待ってはいけません。最大作業負荷の一般的なガイドラインは、閾値の70%～80%程度の使用率です。

ネットワーク性能は、一般的にピアノードに割り当てられたCPUでスケールするので、各ピア（および使用されている場合はCouchDB）に最大CPU容量を提供することをお勧めします。
ordererノードについては、一般的なガイドラインは、2 GBのメモリと1 CPUです。

ステートデータの量が増加すると、特にステートデータベースとしてCouchDBを使用する場合、データベースストレージのパフォーマンスが低下する可能性があります。
したがって、ネットワークコンポーネントを継続的に監視し、閾値を超えた場合は調整を行うとともに、時間の経過とともに環境にコンピュートリソースを追加する計画を立てる必要があります。

## Peer considerations

ネットワーク全体のパフォーマンスに関するピアの考慮事項には、アクティブチャネル数とノード構成プロパティが含まれます。ピアの**core.yaml**ファイルとordererの**orderer.yaml**ファイルにおける、
デフォルトのピアおよびOrdererの構成を以下に示します。

### Number of peers

各組織がデプロイするピア数が多ければ多いほど、各組織のピア間でトランザクションのエンドース要求を負荷分散できるため、ネットワークの性能は向上するはずです。
しかし、ゲートウェイピアは現在のブロックの高さに基づいてトランザクションのエンドースを行うピアを選択するため、ピア間でのエンドースの分散状況は容易に予測できません。
ラウンドロビンなど、事前に定義されたポリシーに基づいて、トランザクションを提出または評価するゲートウェイピアを複数のゲートウェイピアの中から選択することで、
単一のゲートウェイピアによって行われる作業を予測可能な形で減らすことができます。
このゲートウェイピアの選択は、ロードバランサー、クライアントアプリケーションにてコード化されたもの、またはゲートウェイ選択時に使用されるgrpcライブラリによって実現できます。
(注：これらのシナリオによる性能向上の正式なベンチマークは **されていません** )。

### Number of channels per peer

ピアに十分なCPUがあり、1つのチャネルにしか参加していない場合、ピアが内部でどのようにシリアライズとロックを行っていたとしても、ピアのCPUは65～70%以上になることはありません。
ピアが複数のチャネルに参加している場合、ピアはブロック処理の並列性を高めるため、利用可能なリソースを消費します。

一般に、最大負荷で動作している各チャネルに対し、利用可能なCPUコアが少なくとも1つあることを確認してください。
一般的なガイドラインとして、各チャネルの負荷が高度に競合していない場合（つまり、全チャネルがリソースを同時に使用しない場合）、チャネルの数がCPUコアの数を超えても問題ありません。

### Total query limit

ピアは、ピアの速度を低下させる可能性のある予想よりも大きな結果セットを避けるために、
レコードの範囲またはJSON（リッチ）クエリが返すレコードの総数を制限します。この制限は、ピアの **core.yaml** ファイルで設定します:

```yaml

ledger:
  state:
    # Limit on the number of records to return per query
    totalQueryLimit: 100000
```

サンプルの**core.yaml**ファイルとテスト用のdockerイメージで設定されているように、Hyperledger Fabricのクエリ制限の合計はデフォルトで100000です。
必要に応じてこのデフォルト値を増やすこともできますが、より多くのクエリスキャンを必要としない代替設計も検討する必要があります。

### Concurrency limits

各ピアノードは、クライアントからの過剰な同時リクエストを受けないよう、制限を設けています:

```yaml
peer:
# Limits is used to configure some internal resource limits.
    limits:
        # Concurrency limits the number of concurrently running requests to a service on each peer.
        # Currently this option is only applied to endorser service and deliver service.
        # When the property is missing or the value is 0, the concurrency limit is disabled for the service.
        concurrency:
            # endorserService limits concurrent requests to endorser service that handles chaincode deployment, query and invocation,
            # including both user chaincodes and system chaincodes.
            endorserService: 2500
            # deliverService limits concurrent event listeners registered to deliver service for blocks and transaction events.
            deliverService: 2500
            # gatewayService limits concurrent requests to gateway service that handles the submission and evaluation of transactions.
            gatewayService: 500
```

Hyperledger Fabric v2.4で初めてリリースされたPeer Gateway Serviceは、デフォルト500の`gatewayService`制限を導入しました。
しかし、このデフォルト値はネットワークのTPSを制限する可能性があるため、より多くの同時リクエストを許可するためにこの値を増やす必要があるかもしれません。

### CouchDB cache setting

CouchDBを使用しており、（クエリを介してではなく）繰り返し読み込まれるキーの数が多い場合は、データベースのルックアップを回避するためにピアのCouchDBキャッシュを増やすことを選択できます:

```yaml

state:
    couchDBConfig:
       # CacheSize denotes the maximum mega bytes (MB) to be allocated for the in-memory state
       # cache. Note that CacheSize needs to be a multiple of 32 MB. If it is not a multiple
       # of 32 MB, the peer would round the size to the next multiple of 32 MB.
       # To disable the cache, 0 MB needs to be assigned to the cacheSize.
       cacheSize: 64
```

## Orderer considerations

オーダリングサービスは、Raftコンセンサスを使用してブロックを生成します。
コンセンサスに参加するordererの数やブロック生成のパラメータなどの要因は、以下に説明するように、ネットワーク全体のパフォーマンスに影響を与えます。

### Number of orderers

すべてのordererがRaftのコンセンサスに貢献するため、オーダリングサービスノードの数はパフォーマンスに影響します。
5つのオーダリングサービスノードを使用することで、2つのordererのクラッシュに対する障害耐性を提供することができ
（過半数を超える3つのordererが利用可能な状態を維持する必要がある）、良い出発点となります。
オーダリングサービスノードを追加すると、ネットワークのパフォーマンスが低下する可能性があります。
オーダリングサービスノードがネットワークのボトルネックになる場合は、各チャネルに固有のオーダリングサービスノードを配置することができます。

### SendBufferSize

Hyperledger Fabric v2.5以前では、各ordererノードのデフォルトのSendBufferSizeは10に設定されており、ボトルネックの原因となっていました。
Fabric v2.5では、このデフォルトが100に変更され、スループットが向上しました。
SendBufferSizeパラメータは**orderer.yaml**で設定します:

```yaml
General:
    Cluster:
        # SendBufferSize is the maximum number of messages in the egress buffer.
        # Consensus messages are dropped if the buffer is full, and transaction
        # messages are waiting for space to be freed.
        SendBufferSize: 100
```

### Block cutting parameters in channel configuration

チャネルのコンフィギュレーション設定は、オーダリングサービスのパフォーマンスにも影響します。
チャネルのブロックサイズとタイムアウトパラメータを増加させると、スループットが向上しますが、レイテンシも増加します。
以下に説明するように、ブロックあたりのトランザクションを増やし、ブロック生成間隔を長くするようにオーダリングサービスを構成して、パフォーマンス結果をテストすることができます。

ordererのバッチサイズに関する3つのパラメータ（Max Message Count、Absolute Max Bytes、Preferred Max Bytes）により指定されたブロックサイズと最大トランザクション数に基づいてブロック生成が実行されます。
これらのパラメータは、チャネルコンフィギュレーションを作成または更新するときに設定されます。
チャネル作成の開始点として**configtxgen**と**configtx.yaml**を使用する場合、**configtx.yaml**の以下のセクションが適用されます:

```yaml
Orderer: &OrdererDefaults
    # Batch Timeout: The amount of time to wait before creating a batch.
    BatchTimeout: 2s

    # Batch Size: Controls the number of messages batched into a block.
    # The orderer views messages opaquely, but typically, messages may
    # be considered to be Fabric transactions. The 'batch' is the group
    # of messages in the 'data' field of the block. Blocks will be a few KB 
    # larger than the batch size, when signatures, hashes, and other metadata
    # is applied.
    BatchSize:

        # Max Message Count: The maximum number of messages to permit in a
        # batch. No block will contain more than this number of messages.
        MaxMessageCount: 500

        # Absolute Max Bytes: The absolute maximum number of bytes allowed for
        # the serialized messages in a batch. The maximum block size is this value
        # plus the size of the associated metadata (usually a few KB depending
        # upon the size of the signing identities). Any transaction larger than
        # this value will be rejected by ordering. It is recommended not to exceed 
        # 49 MB, given the default grpc max message size of 100 MB configured on 
        # orderer and peer nodes (and allowing for message expansion during communication).
        AbsoluteMaxBytes: 10 MB

        # Preferred Max Bytes: The preferred maximum number of bytes allowed
        # for the serialized messages in a batch. Roughly, this field may be considered
        # the best effort maximum size of a batch. A batch will fill with messages
        # until this size is reached (or the max message count, or batch timeout is
        # exceeded). If adding a new message to the batch would cause the batch to
        # exceed the preferred max bytes, then the current batch is closed and written
        # to a block, and a new batch containing the new message is created. If a
        # message larger than the preferred max bytes is received, then its batch
        # will contain only that message. Because messages may be larger than
        # preferred max bytes (up to AbsoluteMaxBytes), some batches may exceed
        # the preferred max bytes, but will always contain exactly one transaction.
        PreferredMaxBytes: 2 MB
```

#### Batch timeout

BatchTimeout値には、最初のトランザクションが到着してからブロックを生成するまでに待つ時間を秒単位で設定します。
この値を低く設定しすぎると、希望するサイズまでバッチが充填されなくなる危険性があります。
一方、この値を高く設定しすぎると、ordererがブロックを待つことになり、全体的なパフォーマンスとレイテンシーが低下する可能性があります。
一般的なガイドラインは、BatchTimeoutの値を、最低でも（最大メッセージ数÷1秒あたりの最大トランザクション数）に設定することです。

#### Max message count

Max message count値には、1つのブロックに含めるトランザクションの最大数を設定します。

#### Absolute max bytes

Absolute max bytes値には、オーダリングサービスが生成する最大のブロックサイズをバイト単位で設定します。
Absolute max bytesの値より大きなトランザクションは生成されません。
この設定は、Preferred max bytes値の2～10倍にすることが一般的です。
注：推奨される最大サイズは、デフォルトのgrpcサイズ制限である100MBに基づく上限値である49MBです。

#### Preferred max bytes

Preferred max bytes値には、理想的なブロックサイズをバイト単位で設定します。
最小のトランザクションサイズ（エンドースを含まないもの）は約1KBです。必要なエンドース1件につき1KBを追加すると、典型的なトランザクション・サイズは約3～4KBになります。
したがって、Preferred max bytesの値は、Max message countに平均トランザクションサイズの予想値を乗じた値に近い値を設定することが推奨されます。
トランザクション実行時、可能な限り、ブロックはこのサイズを超えません。ブロックがAbsolute max bytes値を超えるようなトランザクションが到着した場合、ブロックは生成され、
トランザクションは新しいブロックに含まれます。新しいブロックは単一のトランザクションのみで生成され、これもAbsolute max bytes値を超えることはありません。

## Application considerations

アプリケーションアーキテクチャを設計する際、さまざまな選択がアプリケーションとネットワークの両方の全体的なパフォーマンスに影響を与える可能性があります。
これらのアプリケーションの考慮事項については、以下で説明します。

### Avoid CouchDB for high-throughput applications

CouchDBのパフォーマンスは組み込みのLevelDBよりも明らかに遅く、場合によっては2倍遅くなります。
[Fabric state database documentation](./deploypeer/peerplan.html#state-database) に記載されている通り、CouchDBステートデータベースが提供する唯一の追加機能は、JSON（リッチ）クエリです。
また、ステートデータの整合性を確保するために、CouchDBデータへの直接アクセス（たとえば、Fauxton UI 経由）を許可しないでください。

ステートデータベースとしてのCouchDB には、全体的なネットワークパフォーマンスを低下させ、追加のハードウェアリソース（およびコスト）を必要とするという別の制約もあります。
さらに、JSONクエリは検証時に再実行されないため、Fabricにはファントムリードに対する組み込みの保護機能がありません。
一方、範囲クエリではそのような保護機能が提供されます
（JSONクエリを使用する場合、チェーンコードの実行とブロックの検証の間に状態にレコードが追加されるなどのファントムリードを許容するようにアプリケーションを設計する必要があります）。
したがって、高スループットアプリケーションでは、CouchDBにJSONクエリを使用するのではなく、追加のキーに基づく範囲クエリを使用することを検討する必要があります。

あるいは、[Off-chain sample](https://github.com/hyperledger/Fabric-samples/tree/main/off_chain_data) に示されているように、
クエリをサポートするためにオフチェーン ストアの使用を検討してください。
クエリにオフチェーン ストアを使用すると、データをより細かく制御でき、クエリトランザクションはHyperledger Fabricピアとネットワークのパフォーマンスに影響を与えません。
また、ユーザによるクエリのニーズにより適合したSQLデータベースや分析サービスなど、オフチェーンストレージに適したデータストアを使用することもできます。

CouchDBのパフォーマンスは、ステートデータの量が増加するにつれてLevelDBよりも低下するため、アプリケーションの存続期間中、CouchDBインスタンスに十分なリソースを提供する必要があります。

### General chaincode query considerations

返されるデータの量が無制限になる可能性があるチェーンコードクエリや、返されるデータの量が制限されていてもトランザクションがタイムアウトするほど大量のデータを返すチェーンコードクエリを記述することは避けてください。
トランザクションを長時間実行することは Fabric のベストプラクティスに反するため、返されるデータの量を制限するようにクエリを最適化してください。
JSONクエリを使用する場合は、インデックスが付けられていることを確認してください。JSONクエリのガイダンスについては、
[Good practice for queries](./couchdb_as_state_database.html#good-practices-for-queries) を参照してください。

### Use the Peer Gateway Service

ピアゲートウェイサービス（v2.4の新機能）と新しいFabric-Gateway client SDK は、従来のGo、Java、およびNode SDKを大幅に強化しています。
ピアゲートウェイサービスでは、スループットが大幅に向上し、クライアントアプリケーションの機能が増え、複雑さが軽減されます。
例えばこのサービスは、チェーンコードエンドースメントポリシーだけでなく、
トランザクションシミュレーションに含まれるステートベースのエンドースメントポリシーを満たすために必要なエンドースメントを自動的に収集（試行）します（これは従来のSDKでは不可能です）。

ピアゲートウェイサービスでは、クライアントがトランザクションを送信するために維持する必要があるネットワーク接続の数も削減されます。
従来のSDKを使用すると、クライアントは組織全体の複数のピアノードとordererノードに接続する必要がある場合があります。
これに対し、ピアゲートウェイサービスを使用すると、アプリケーションは単一の信頼できるピアに接続でき、そのピアは他のピアノードとordererノードからエンドースメントを収集し、
クライアントアプリケーションに代わってトランザクションを送信します。
必要に応じて、アプリケーションは複数の信頼できるピアをターゲットにでき、高い同時実行性と冗長性を実現できます。

アプリケーション開発環境を従来のSDKから新しいピアゲートウェイ サービスに変更すると、クライアントのCPUとメモリのリソース要件も削減されますが、ピアリソース要件は若干増加します。

ピアゲートウェイサービスの詳細については、[Sample gateway application](https://github.com/hyperledger/Fabric-samples/blob/main/full-stack-asset-transfer-guide/docs/ApplicationDev/01-FabricGateway.md) を参照してください。

### Payload size

トランザクションとして送信されるデータの量と、トランザクション内のキーに書き込まれるデータの量は、アプリケーションのパフォーマンスに影響します。
ペイロードサイズには、データだけが含まれるのではないことに注意してください。Fabricに必要なデータ構造に加えて、クライアントとエンドーシングピアの署名も含まれます。

言うまでもなく、大きなペイロードサイズは、どのブロックチェーンソリューションでもアンチパターンです。
大きなデータをオフチェーンで保存し、データのハッシュをオンチェーンで保存することを検討してください。

### Chaincode language

GoチェーンコードはHyperledger Fabric環境で最高のパフォーマンスを示し、次にNodeチェーンコードが続きます。Javaチェーンコードのパフォーマンスは3つの中で最も低く、高スループットのアプリケーションには推奨されません。

### Node chaincode

Nodeは、コードを実行するために単一のスレッドのみを使用する非同期ランタイム実装です。
Nodeはガベージ コレクションなどのアクティビティのためにバックグラウンド スレッドを実行しますが、Nodeチェーンコードに2つ以上のvCPU（通常、実行可能な同時スレッドの数に相当）
を割り当ててもパフォーマンスが向上しない可能性があります（使用可能なリソースに制限があるKubernetes使用時など）。
Nodeチェーンコードのパフォーマンスを監視して、vCPUがどれだけ使用されているかを確認することは価値があります。
Nodeチェーンコードは自己制限型であり、無制限ではありませんが、Nodeチェーンコードプロセスにリソース制限を割り当てることもできます。

Node v12より前では、Nodeプロセスのメモリ使用量はデフォルトで1.5 GBに制限されていました。
Nodeチェーンコードを実行するためにメモリを増やすには、Node実行ファイルにパラメータを渡す必要がありました。
Node v12より前のバージョンではチェーンコードプロセスを実行するべきではなく、Hyperledger Fabric v2.5ではNode v16以降の使用が必須です。
Nodeチェーンコードを起動する際は様々なパラメータを指定できますが、Node v12以降ではデフォルト値を上書きする必要があるケースはほとんどありません。

### Go chaincode

Golangランタイム実装は並行処理に最適です。利用可能なすべてのCPUを使用できるため、チェーンコードプロセスに割り当てられたリソースによってのみ制限されます。チューニングは必要ありません。

### Chaincode processes and channels

Hyperledger Fabric は、チェーンコードIDとバージョンが一致する場合、チャネル間でチェーンコードプロセスを再利用します。
例えば、同じチェーンコードIDとバージョンがデプロイされた2つのチャネルにピアが参加している場合、ピアは両方のチャネルにおいて1つのチェーンコードプロセスのみと対話します。
これにより、特に自己制限的なNodeチェーンコードの場合、チェーンコードプロセスに予想以上の負荷がかかる可能性があります。
この場合、Goチェーンコードを使用してください。あるいは、異なるIDまたはバージョン番号を使用して各チャネルにNodeチェーンコードをデプロイすることで、
チャネルごとに異なるチェーンコードプロセスを確保することができます。

### Endorsement policies

トランザクションが有効なものとしてコミットされるためには、チェーンコードのエンドースメントポリシーとステートベースのエンドースメントポリシーを満たすために必要な署名が含まれている必要があります。
ピアゲートウェイサービス（Fabric v2.4以降）は、ポリシーを満たすのに十分なピアにのみリクエストを送信します（優先されるピアが利用できない場合は他のピアを試します）。
エンドースメントポリシーは、トランザクションが有効としてコミットされるために必要なピアの数（およびその署名）を規定するため、パフォーマンスに影響します。

### Private Data Collections (PDCs) vs World State

一般的に、プライベートデータコレクション(PDC)を使用するかどうかは、パフォーマンスの観点ではなく、アプリケーションアーキテクチャの観点により決まります。
ただし、たとえば、ワールドステートを使用するのではなくPDCを使用してアセットを保存すると、TPSが約半分になることに注意してください。

### Single or multiple channel architecture

アプリケーションを設計するときは、単一のチャネルを利用するか、複数のチャネルを利用できるかを検討してください。
与えられたシナリオでデータサイロが問題にならない場合は、アプリケーションで複数のチャネルからなるアーキテクチャを使用することでパフォーマンスを向上させることができます
（複数のチャネルを使用すると、より多くのピアリソースを使用できるため）。

## CouchDB considerations

前述のように、CouchDBは高スループットのアプリケーションには推奨されておらず、以下に説明するすべての制限を考慮する必要があります。

### Resources

ステートデータベースのサイズが大きくなるにつれて、CouchDBが消費するリソースも増えるため、CouchDBインスタンスが使用するリソースを必ず監視してください。

### CouchDB cache

外部の CouchDBステートデータベースを使用する場合、トランザクションのエンドースおよび検証フェーズでの参照遅延がパフォーマンスのボトルネックになることが判明しています。
Fabric v2.xでは、ピアキャッシュによって、これらのコストのかかる検索の多くが高速なローカルキャッシュ参照に置き換えられます。

キャッシュによってJSONクエリのパフォーマンスが向上することはありません。

キャッシュの構成については、上記の **CouchDB Cache setting** の詳細 (**Peer Considerations**内) を参照してください。

### Indexes

CouchDB JSON クエリを利用するチェーンコードには、CouchDBインデックスを含めてください。
クエリをテストしてインデックスが利用されていることを確認し、インデックスを使用できないクエリは避けてください。
例えば、クエリ演算子 $or、$in、$regex を使用すると、完全なデータ スキャンが実行されます。
詳細については、[Hyperledger Fabric Good Practices For Queries](./couchdb_as_state_database.html#good-practices-for-queries) を参照してください。

複雑なクエリはインデックスを使用しても時間がかかるため、クエリを最適化してください。
クエリの結果が制限されたデータセットになるようにしてください。Hyperledger Fabricによって、返される結果の総数が制限される場合もあります。

ピアとCouchDBのログをチェックすることで、クエリにかかる時間や、クエリが「インデックスを作成すべき」というwarningログエントリからインデックスを使用できなかったクエリがあるかどうかを確認できます。

クエリに時間がかかる場合は、CouchDBで使用できるCPU/メモリを増やすか、より高速なストレージを使用すると、パフォーマンスが向上する可能性があります。

CouchDBログで、"The number of documents examined is high in proportion to the number of results returned. Consider adding a more specific index to improve this."などの警告がないか確認してください。
これらのメッセージは、スキャンされているドキュメントの数が多いため、インデックスを改善できる余地があることを示しています。

### Bulk update

Hyperledger Fabric は、CouchDB のパフォーマンスを向上させるために、CouchDBへの一括更新(Bulk update)呼び出しを使用します。
一括更新はブロックレベルで実行されるため、ブロックに多くのトランザクションを含めるとスループットが向上します。
ただし、より多くのトランザクションを含めるためにブロックが生成されるまでの時間を長くすると、レイテンシにも影響します。

## HSM

FabricネットワークでHSMを使用すると、パフォーマンスに影響します。影響を定量化することはできませんが、HSMのパフォーマンスやネットワーク接続などの変数は、ネットワーク全体のパフォーマンスに影響します。

ピア、orderer、クライアントがHSMを使用するように構成した場合、ordererによるブロックの作成、ピアによるエンドースメントとブロックイベント、各クライアントによるリクエストなど、署名を必要とするものはすべてHSMを利用します。

なお、HSMは各ノードによって実行される署名の検証には関与しないことに注意してください。

## Other performance considerations

単一のトランザクションを定期的に送信すると、ブロック生成パラメータと相関するレイテンシが発生します。
たとえば、BatchTimeoutを2秒に設定して 100バイトのトランザクションを送信すると、トランザクションの送信からコミットまでの時間は2秒強になります。
ただし、これはFabricネットワークパフォーマンスの正しいベンチマークではありません。実際のベンチマークは、複数のトランザクションを同時に送信することで取得する必要があります。
