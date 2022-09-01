# FLEDGE Key/Value service trust model

Authors:

- Philip Lee [pjl@google.com](mailto:pjl@google.com)
- Peiwen Hu [peiwenhu@google.com](mailto:peiwenhu@google.com)

## Context

We are currently in an early requirements gathering and design phase and aim to test the FLEDGE key/value service either in 2H 2022 or 1H 2023. Use of this service is not currently part of the critical path for third-party cookie deprecation, however we welcome input on the design, privacy properties and timing from the ecosystem on both setting up their own server as well as leveraging one described here.

現在、私たちは初期の要件収集および設計段階にあり、2022 年後半または 2023 年前半に FLEDGE のキー/バリューサービスをテストすることを目標としています。このサービスの利用は、現時点ではサードパーティークッキー廃止のクリティカルパスの一部ではありませんが、設計、プライバシー特性、タイミングについて、エコシステムが独自のサーバーを設置する場合と、ここで説明したサーバーを活用する場合の両方から意見をお待ちしています。

## Introduction

[FLEDGE is a proposal](https://github.com/WICG/turtledove/blob/main/FLEDGE.md) for an API to serve remarketing use cases without third-party cookies. [FLEDGE services](https://github.com/privacysandbox/fledge-docs/blob/main/trusted_services_overview.md) add real-time signals into ad selection for both buyers and sellers.

[FLEDGE](https://github.com/WICG/turtledove/blob/main/FLEDGE.md) は、サードパーティの Cookie を使わずにリマーケティングのユースケースを提供する API の提案である。[FLEDGE サービス](https://github.com/privacysandbox/fledge-docs/blob/main/trusted_services_overview.md)は、買い手と売り手の双方にとって、広告の選択にリアルタイムシグナルを追加するものです。

Overall, this explainer proposes using similar techniques to the [Aggregation Service proposal](https://github.com/WICG/conversion-measurement-api/blob/main/AGGREGATION_SERVICE_TEE.md) for the Attribution Reporting API, while some specific areas may benefit from improvements in the future for better performance.

全体として、この説明者は、Attribution Reporting API について、[Aggregation Service proposal](https://github.com/WICG/conversion-measurement-api/blob/main/AGGREGATION_SERVICE_TEE.md) と同様の技術を使うことを提案していますが、いくつかの特定の領域は、より良いパフォーマンスのために将来的に改善することが有益であるかもしれません。

## Proposed design principles

The key/value service for the FLEDGE API intends to meet a set of privacy and security goals through technical measures. This includes:

FLEDGE API のキー/バリューサービスは、技術的な手段によって一連のプライバシーとセキュリティの目標を満たすことを意図しています。これには以下が含まれます。

1. Uphold the Privacy Sandbox design goals for privacy protections in FLEDGE.
2. Prevent inappropriate access to the lookup keys (or other intermediate data) and prevent logging/persistence of user data through technical enforcement.
3. Allow adtechs to retain control over the realtime data they serve.
4. Provide open and transparent implementations for any infrastructure outside of the client.

---

1. FLEDGE におけるプライバシー保護のための Privacy Sandbox の設計目標を支持すること。
2. アドテクが提供するリアルタイムデータの制御を保持できるようにする。
3. ルックアップキー(または他の中間データ)への不適切なアクセスを防止し、技術的な強制力によりユーザーデータの記録/存続を防止する。
4. クライアント外のあらゆるインフラに対して、オープンで透明性の高い実装を提供すること。

## Key terms

Before reading this explainer, it will be helpful to familiarize yourself with key terms and concepts. These have been ordered non-alphabetically to build knowledge based on previous terms. All of these terms will be reintroduced and further described throughout this proposal.

この説明書を読む前に、重要な用語と概念に慣れておくと便利です。これらの用語は、以前の用語を基に知識を深めるために、アルファベット順ではありません。これらの用語はすべて、この提案の中で再度紹介され、さらに説明されます。

- Adtech: a company that provides services to deliver ads. This term is used to describe the buyers and sellers who will use the key/value service, as described in the [FLEDGE explainer](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#summary).
- Client software: shorthand for the implementation of FLEDGE in a browser (such as a [Chrome browser](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#31-fetching-real-time)) or on a device (such as an [Android device](https://developer.android.com/design-for-safety/privacy-sandbox/fledge#ad-selection-ad-tech-platform-managed-trusted-server)).
- Attestation: a mechanism to authenticate software identity, usually with [cryptographic hashes](https://en.wikipedia.org/wiki/Cryptographic_hash_function) or signatures. In this proposal, attestation matches the code running in the adtech-operated key/value service with the open source code.
- Trusted execution environment (TEE): a dedicated, closed execution context that is isolated through hardware memory protection and cryptographic protection of storage. The TEE's contents are protected from observation and tampering by unauthorized parties, including the root user.
- Key management service (KMS): a centralized component tasked with provision of decryption keys to appropriately secured FLEDGE service instances. Provision of public encryption keys to end user devices and key rotation also fall under key management.

---

- Adtech: 広告を配信するサービスを提供する会社である。[FLEDGE 説明書](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#summary)に記載されている、キー/バリューサービスを利用する買い手と売り手のことを指します。
- Client software: ブラウザ([Chrome ブラウザ](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#31-fetching-real-time)など)やデバイス([Android デバイス](https://developer.android.com/design-for-safety/privacy-sandbox/fledge#ad-selection-ad-tech-platform-managed-trusted-server)など)で FLEDGE を実装するための略記。
- Attestation: ソフトウェアの同一性を認証するメカニズムで、通常は [暗号ハッシュ](https://en.wikipedia.org/wiki/Cryptographic_hash_function) や署名を使用します。この提案では、認証は、アドテクノロジーが運営するキー/バリューサービスで実行されるコードとオープンソースコードを照合します。
- Trusted execution environment (TEE): ハードウェアメモリ保護とストレージの暗号化保護によって隔離された、専用の閉じた実行コンテキストです。TEE のコンテンツは、ルートユーザーを含む権限のない第三者による観察や改ざんから保護されています。
- Key management service (KMS): 適切に保護された FLEDGE サービス・インスタンスに復号化鍵を提供することを任務とする集中管理コンポーネントである。エンドユーザー・デバイスへの公開暗号鍵の提供および鍵のローテーションも、鍵管理の対象となる。

## Key/value service workflow

Adtechs use the key/value service to supply realtime information to the FLEDGE ad auction. This information could be used, for example, to add budgeting data about each ad. The proposed workflow is as follows:

Adtech は、キー/バリューサービスを使用して、FLEDGE の広告オークションにリアルタイムで情報を供給します。この情報は、例えば、各広告に関するバジェットデータを追加するために使用することができます。提案するワークフローは次のとおりです。

1. Adtechs deploy and operate the key/value service on a cloud provider with the necessary TEE capabilities.
2. Adtechs load key/value data into the service, and retain the ability to push changes to this data at any time.
3. While running a FLEDGE auction, the client device sends the lookup request to the key/value service that was specified by the buyer or seller. The data in-transit is encrypted to make sure only the job is able to see the cleartext.
4. To decrypt the requests, the adtech-operated key/value service uses the private keys, which it has previously obtained from the KMS by attesting that it is running an approved version of the code in the TEE. The KMS will not release the private keys to anything or anyone else. Other mechanisms to secure data in-transit may also be explored in the future.
5. The service looks up the matching data for the keys and returns it also in encrypted form. Refer to the [FLEDGE Key/Value Server APIs Explainer](https://github.com/WICG/turtledove/blob/main/FLEDGE_Key_Value_Server_API.md).

---

1. アドテクノロジーは、必要な TEE 機能を持つクラウドプロバイダー上にキー／バリューサービスを展開し、運用します。
2. アドテクノロジーは、キー/バリューデータをサービスにロードし、このデータに対していつでも変更を加えることができます。
3. アドテクノロジーが運営するキー／バリュー・サービスは、リクエストを解読するために、TEE で承認されたバージョンのコードを実行していることを証明することで、KMS から事前に入手した秘密鍵を使用します。KMS は秘密鍵を何にも、誰にも公開しない。転送中のデータを保護する他のメカニズムも、将来的に検討されるかもしれない。
4. FLEDGE オークションの実行中、クライアントデバイスは買い手または売り手が指定したキー/バリューサービスにルックアップリクエストを送信します。転送中のデータは暗号化され、ジョブのみが平文を見ることができるようになります。
5. このサービスは、キーに対応するデータを検索し、暗号化された形で返します。[FLEDGE Key/Value Server APIs Explainer](https://github.com/WICG/turtledove/blob/main/FLEDGE_Key_Value_Server_API.md)を参照してください。

## Privacy and security considerations

### Overall flow

The FLEDGE explainer outlines the following security/privacy goals for the key/value service:

FLEDGE の説明では、Key/Value サービスにおけるセキュリティ/プライバシーの目標について、以下のようにまとめています。

The browser needs to trust that the return value for each key will be based only on that key and the hostname, and that the server does no event-level logging and has no other side effects based on these requests.

ブラウザは、各キーの返り値がそのキーとホスト名のみに基づくこと、およびサーバーがイベントレベルのロギングを行わず、これらのリクエストに基づく他の副作用がないことを信頼する必要があります。

To meet these requirements, the proposed design for the key/value service relies on a number of factors:

これらの要件を満たすために、キー/バリューサービスの設計案は、いくつかの要素に依存しています。

- Attestation ensures that an adtech-operated key/value service runs an approved codebase.
- Data in use:
  - Lookup of key/value data happens within a secure and isolated Trusted Execution Environment. This prevents any party from learning the keys being requested.
- Data in transit:
  - Client devices will pre-load and periodically refresh the public encryption key signed by a KMS that the client devices recognize.
  - The KMS only grants the private key to successfully attested servers.
- Data at rest:
  - The key/value service will never do per-request lookups to persistent storage for the key/value data. Techniques such as pre-loading the dataset should be used.
- Auditability:
  - Open source implementations of the key/value service ensure that these systems are publicly accessible and can be inspected by a broad set of stakeholders.

---

- 認証は、アドテクノロジーが運営するキー／バリューサービスが、承認されたコードベースを実行していることを保証します。
- Data in use:
  - Key/Value データの検索は、安全かつ隔離された Trusted Execution Environment 内で行われる。これにより、いかなる当事者も要求された鍵を知ることができない。
- Data in transit:
  - クライアント端末は、クライアント端末が認識する KMS によって署名された公開暗号鍵を事前にロードし、定期的にリフレッシュします。
  - KMS は、認証に成功したサーバーにのみ秘密鍵を付与する。
    Data at rest:
  - key/value サービスは、key/value データの永続的なストレージへのリクエストごとのルックアップを決して行いません。データセットのプリロードなどのテクニックを使用する必要があります。
    Auditability:
  - 鍵/値サービスのオープンソース実装により、これらのシステムは一般に公開され、幅広いステークホルダーが検査できるようになります。

![Overview of the FLEDGE key/value service workflow.](images/fledge_key_value_server.png)

### Threats

Ideally, the key/value service would never get access to any information that could be used to identify a specific client or user. However, in practice, request timestamps and other necessary metadata included in lookup requests make this idealized system impractical.

理想的には、key/value サービスは、特定のクライアントやユーザーを識別する ために使用できるいかなる情報にもアクセスすることはないだろう。しかし、実際には、リクエストのタイムスタンプや、ルックアップリクエストに含まれる他の必要なメタデータが、この理想的なシステムを非現実的なものにしています。

The main threats that we're concerned with are:

私たちが関心を寄せる主な脅威は、以下の通りです。

1. That the timestamp of the request could be used to correlate the lookup keys with the request of the top-level page and thus identify a user. This is mitigated by limiting the logs that are written.
2. During the FLEDGE auction, a client asks a key/value service about an assortment of keys that were previously stored on the client - perhaps including keys that were stored in multiple different contexts. That set of keys, therefore, risks leaking some information about the client's activity. The privacy goal of key/value service design is to give the client confidence that the set of keys cannot be used for tracking or profiling purposes. This threat is mitigated by the TEE protections to allow only pre-approved code.
3. Note that the values being stored in the key/value service are loaded by adtechs and expected to be publicly visible to clients. These are therefore not confidential.
4. The user's IP address could be used for fingerprinting. This is mitigated by the logging requirements and TEE protections to allow only pre-approved code. 2. We also have the option to route the traffic to these instances through [Gnatcatcher](https://github.com/bslassey/ip-blindness) to mask the user's IP address.

---

1. リクエストのタイムスタンプが、ルックアップキーをトップページのリクエストと関連付けるために使われ、その結果ユーザを特定することができること。これは、書き込まれるログを制限することで緩和される。
2. FLEDGE オークションの間、クライアントは key/value サービスに、そのクライアントに以前保管されていた様々な鍵(おそらく複数の異なるコンテキストで保管されていた鍵も含む)について問い合わせる。そのため、その鍵の集合は、クライアントの活動に関する何らかの情報を漏洩する危険性がある。鍵/価値サービスの設計におけるプライバシーの目標は、鍵のセットが追跡やプロファイリング の目的で使用されないという確信をクライアントに与えることである。この脅威は、事前に承認されたコードのみを許可する TEE 保護機能によって軽減される。
3. Key/Value サービスに格納される値は、アドテクノロジーによって読み込まれ、クライアントから一般に見えることが期待されていることに留意してください。したがって、これらは機密情報ではありません。
4. ユーザーの IP アドレスがフィンガープリントに使われる可能性がある。これは、事前に承認されたコードのみを許可するためのロギング要件と TEE 保護によって緩和されます。2.また、これらのインスタンスへのトラフィックを[Gnatcatcher](https://github.com/bslassey/ip-blindness)を介してルーティングし、ユーザーの IP アドレスをマスクするオプションもあります。

### Side effects

There are a number of ways that visible side effects of request handling are possible in general with servers. Here's how we plan to mitigate those effects:

リクエスト処理による目に見える副作用は、一般的にサーバーで可能な方法がいくつかあります。ここでは、それらの影響をどのように軽減していくかを計画しています。

1. Monitoring metrics - Only noised aggregate metrics will be available for monitoring and alerting. These will be aggregated to at least k size.
2. For example, counting the approximate number of failed requests in the past n minutes is likely to be fine but doing so at millisecond granularity is not.
3. Logging - No event-level logs will be written.
4. Event-level logging that normally happens from any shared libraries we might use will be disabled.
5. Outbound RPCs - These servers will make a small set of outbound RPCs that they initiate.
6. They'll do so to each other for load balancing and sharding of data. Requests will only be made to other key/value service that are part of this system and that have the same protections in place. Attestation checks will be chained together.
7. There are no outbound RPCs to other systems.
8. Inbound RPC responses - As in the [API explainer](https://github.com/WICG/turtledove/blob/main/FLEDGE_Key_Value_Server_API.md), there are two sets of APIs.
9. For the client device to read key/value data. The client will provide its own secure channel and user data is allowed to go back to the browser.
10. For adtech server operators to mutate key/value data. These are private APIs that are only available to the server operator. The key/value service will acknowledge success or failure but not send other responses.

---

1. モニタリング・メトリクス - ノイズされたアグリゲート・メトリクスのみがモニタリングとアラートに利用可能です。これらは少なくとも k 個のサイズに集約される。
2. 例えば、過去 n 分間に失敗したリクエストのおおよその数を数えることは問題ないと思われるが、ミリ秒の粒度でそれを行うことは問題である。
3. ログ記録 - イベントレベルのログは書き込まれません。
4. 通常、私たちが使用するかもしれない共有ライブラリから発生するイベントレベルのログは無効になります。
5. アウトバウンド RPC - これらのサーバーは、彼らが開始するアウトバウンド RPC の小さなセットを作成します。
6. データの負荷分散とシャーディングのために、お互いにそうすることになります。リクエストは、このシステムの一部であり、同じ保護が施されている他のキー/バリューサービスに対してのみ行われる。認証のチェックは連鎖的に行われる。
7. 他システムへのアウトバウンド RPC はありません。
8. Inbound RPC responses - [API 解説](https://github.com/WICG/turtledove/blob/main/FLEDGE_Key_Value_Server_API.md)にあるように、API には 2 つのセットがあります。
9. アドテクサーバー運営者がキー/バリューデータを変異させるためのものです。これらは、サーバー運営者だけが利用できるプライベート API です。key/value サービスは、成功または失敗を認めますが、その他のレスポンスは送信しません。
10. クライアントデバイスがキー/バリューデータを読み取るために。クライアントは独自のセキュアなチャネルを提供し、ユーザーデータはブラウザに戻ることができます。

### Trusted execution environment

A trusted execution environment (TEE) is a combination of hardware and software mechanisms that allows for code to execute in isolation, not observable by any other process regardless of the credentials used. The code running in a TEE will be open sourced and audited to ensure proper operation. Only TEE instances running an approved version of the key/value service code will be able to decrypt lookup requests.

信頼された実行環境(TEE)は、使用される認証情報に関係なく、他のプロセスから観察できないように、コードを分離して実行できるハードウェアとソフトウェアの仕組みの組み合わせである。TEE で実行されるコードは、オープンソース化され、適切な動作を保証するために監査されます。キー/バリューサービスコードの承認されたバージョンを実行している TEE インスタンスのみが、ルックアップリクエストを解読することができる。

The code running within a TEE performs the following tasks:

TEE 内で実行されるコードは、以下のタスクを実行します。

- Look up the values for the keys being requested.
  - 要求されているキーの値を調べる。
- Handle error reporting, logging, crashes and stack traces, access to external storage and network requests in a way that aids usability and troubleshooting while protecting raw and intermediate data at all times.
  - エラーレポート、ログ、クラッシュ、スタックトレース、外部ストレージへのアクセス、ネットワークリクエストを、生データと中間データを常に保護しながら、ユーザビリティとトラブルシューティングを支援する方法で処理します。

Adtechs will operate their own TEE-based key/value service deployment on a cloud provider with the necessary TEE capabilities. Adtechs will also control the data that they load into the key/value service to be served. In addition to TLS, requests are encrypted by the client and decryption inside only the TEE ensures that the adtech will not see which keys are being requested.

アドテクノロジーは、必要な TEE 機能を備えたクラウドプロバイダー上で、独自の TEE ベースのキー／バリューサービスの展開を運用します。また、アドテクノロジーは、キー／バリューサービスにロードするデータを制御してサービスを提供します。TLS に加え、リクエストはクライアントによって暗号化され、TEE 内部のみで復号化されるため、どの鍵がリクエストされたかはアドテックにはわからないようになっています。

### Attestation and cryptographic key management

The code running within the TEE is the only place in the system where the list of items to look up will be decrypted. The code will be open sourced so it can be audited by security researchers, privacy advocates, and adtechs.

TEE 内で動作するコードは、システム内で唯一、調べる項目のリストが復号化される場所です。このコードは、セキュリティ研究者、プライバシー擁護者、アドテク企業などが監査できるよう、オープンソース化される予定です。

The server releaser will periodically release binary images of the key/value service code for TEE deployment. A cryptographic hash of the build product (the image to be deployed on the TEE) is obtained as part of the build process. The build is reproducible so that anyone can build binaries from source and verify they are identical to the images released by Google. The way in which the list of authorized images is maintained has not yet been determined, please see the [Aggregation Service explainer](https://github.com/WICG/conversion-measurement-api/blob/main/AGGREGATION_SERVICE_TEE.md) for some ideas.

サーバリリーサーは、キー/バリューサービスコードのバイナリイメージを TEE 展開のために定期的にリリースする。ビルドプロセスの一部として、ビルドプロダクト(TEE にデプロイされるイメージ)の暗号化ハッシュが取得されます。ビルドは再現可能であるため、誰でもソースからバイナリをビルドし、Google が公開するイメージと同一であることを検証することができる。許可されたイメージのリストをどのように管理するかはまだ決まっていません。いくつかのアイデアは[Aggregation Service explainer](https://github.com/WICG/conversion-measurement-api/blob/main/AGGREGATION_SERVICE_TEE.md)を参照してください。

When a new approved version of the key/value service binary is released, it will be added to the list of authorized images. If an image is found in retrospect to contain a critical security or functional flaw, it can be removed from the list. Images older than a certain age will also be periodically retired from the list.

Key/Value サービス・バイナリの新しい承認済みバージョンがリリースされると、承認済みイメージのリストに追加されます。もし、あるイメージが重大なセキュリティ上の欠陥や機能的な欠陥を含んでいることが後から判明した場合、そのイメージはリストから削除される可能性があります。また、一定期間以上経過したイメージは、定期的にリストから抹消されます。

Public encryption keys are necessary for the client software to encrypt requests. Decryption keys are required for the key/value service to process the requests. The decryption keys are released only to TEE images whose cryptographic hash matches one of the images on the authorized images list.

公開暗号化キーは、クライアントソフトウェアがリクエストを暗号化するために必要です。復号化キーは、Key/Value サービスがリクエストを処理するために必要である。復号化キーは、暗号化ハッシュが許可されたイメージリストのいずれかに一致する TEE イメージにのみ解放される。

The encryption is bi-directional. Responses back to the client software are also encrypted.

暗号化は双方向に行われます。クライアントソフトウェアに返すレスポンスも暗号化されます。

## Initial experiment plans

The initial implementation strategy for the key/value service is as follows:

Key/Value サービスの初期導入方針は以下の通りである。

- A subsequent update of the [API explainer](https://github.com/WICG/turtledove/blob/main/FLEDGE_Key_Value_Server_API.md) would incorporate new parameters to the Read API to facilitate secure communication, [Github issue here](https://github.com/WICG/turtledove/issues/294).
  - その後の[API 解説](https://github.com/WICG/turtledove/blob/main/FLEDGE_Key_Value_Server_API.md)の更新で、Read API に新しいパラメータが組み込まれ、安全な通信ができるようになります[Github issue はこちら](https://github.com/WICG/turtledove/issues/294)。
- The TEE based key/value service implementation would be deployed on cloud service(s) which support needed security features. We envision the key/value service being capable of deployment with multiple cloud providers.
  - TEE ベースの鍵/値サービスの実装は、必要なセキュリティ機能をサポートするクラウドサービス上に展開される。この鍵/価値サービスは、複数のクラウドプロバイダーへの展開が可能であることを想定しています。

## Support for user-defined functions (UDFs)

Given ecosystem feedback, we plan to support user-defined functions (UDFs), to be loaded and executed inside the key/value service. This will allow more flexibility in how this server is used and provide a server-side FLEDGE platform for code execution. The loading mechanism has yet to be determined, and will be included in a future update to this explainer.

エコシステムのフィードバックを受けて、私たちはユーザー定義関数(UDF)をサポートする予定です。UDF はキー/バリューサービス内でロードして実行することができます。これにより、このサーバーの使用方法をより柔軟に変更でき、コード実行用のサーバーサイド FLEDGE プラットフォームが提供されます。ロードの仕組みはまだ決定していませんが、この説明書の将来の更新に含まれる予定です。

By default we will provide a reference UDF that will do a lookup for the key and simply return the value. Replacing this with a custom UDF is not needed if only basic lookup functionality is required. (We expect the performance impact of the reference UDF to be negligible.) The data store that holds the adtech-loaded data will be exposed to the UDF via new read-only APIs.

デフォルトでは、キーに対するルックアップを行い、単に値を返す参照 UDF を提供する。基本的なルックアップ機能のみが必要な場合は、これをカスタム UDF に置き換える必要はありません。(アドテックにロードされたデータを保持するデータストアは、新しい読み取り専用の API を通じて UDF に公開されます。

This diagram shows how the interaction will work:

この図は、インタラクションがどのように機能するかを示しています。

![UDF server-integration diagram.](images/fledge_kv_server_custom_logic.png)

The diagram shows the request handler inside the key/value service being able to invoke a UDF which can use a new API to read data from the data store.

この図は、key/value サービス内のリクエストハンドラが、データストアからデータを読み込むために新しい API を使用できる UDF を呼び出すことができることを示している。

## Design principles

The following principles are necessary in order to preserve the trust model:

信頼モデルを維持するためには、以下の原則が必要である。

- Sandbox - the custom code will be executed inside a sandbox that limits what it is allowed to do. We're currently looking at the [Open Source V8 engine](https://v8.dev/) inside [Sandbox2](https://developers.google.com/code-sandboxing/sandbox2), which has support for both JavaScript and Web Assembly (WASM). Other suggestions are welcome!
- No network, disk access, timers, or logging - this will be enforced using the sandbox, above. This preserves the key/value service's principle of no side-effects and avoids leaking user data. Coarse timers may be allowed but fine-grained timers are disallowed to help prevent covert channels (e.g. SPECTRE).
- Individual request handling - Per the [FLEDGE explainer](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#31-fetching-real-time-data-from-a-trusted-server), a request to the key/value service may be for multiple keys. The UDF will be called separately for each individual key, rather than once for all of the keys. This ensures that each key is processed independently and prevents a group of keys from being used as a cross-site profile for a user.
- Data store APIs - The key/value service will expose an API to the UDF to read data from the data store. There will be no write APIs to the data store.
- Side-effect free - The UDF can read data from the Data store APIs but cannot write data to any location apart from returning it to the FLEDGE client. No state is shared between UDF executions.
- Limited request metadata access - Each request to the key/value service contains the keys to look up as well as some amount of request metadata. This includes the user IP address, request timestamp, and [experiment id](https://github.com/WICG/turtledove/issues/191). We expect to allow the UDF to have access to some of this metadata and will be updating this explainer with details of that once we work through how this will fit into the privacy model.
- No Open Source requirement - UDFs are provided by adtechs and do not need to be disclosed or shared publicly.

---

- サンドボックス - カスタムコードは、それが許可されているものを制限するサンドボックスの内部で実行されます。現在、[Sandbox2](https://developers.google.com/code-sandboxing/sandbox2) 内の [Open Source V8 engine](https://v8.dev/) を検討しています。これは JavaScript と Web Assembly (WASM) の両方をサポートしています。その他の提案も歓迎します。
- ネットワーク、ディスクアクセス、タイマー、ロギングを行わない - これは上記のサンドボックスを使って強制されます。これは、Key/Value サービスの副作用のない原則を維持し、ユーザーデータの漏えいを避けるためです。粗いタイマーは許可されるかもしれませんが、細かいタイマーは秘密のチャンネル(例:SPECTRE)を防ぐために許可されません。
- 個別のリクエスト処理 - [FLEDGE 説明書](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#31-fetching-real-time-data-from-a-trusted-server)によると、 key/value サービスに対するリクエストは複数のキーに対するものである可能性がある。UDF は、すべてのキーに対して一度ではなく、個々のキーに対して別々に呼び出される。これにより、各キーは独立して処理され、キーのグループがユーザーのクロスサイトプロファイルとして使用されるのを防ぐことができます。
- データストア API - Key/Value サービスは、データストアからデータを読み取るための API を UDF に公開する。データストアへの書き込み API は存在しない。
- 副作用なし - UDF はデータストア API からデータを読み込むことができますが、FLEDGE クライアントにデータを返す以外に、データを任意の場所に書き込むことはできません。UDF の実行間で状態が共有されることはありません。
- リクエストのメタデータへのアクセス制限 - key/value サービスへの各リクエストは、検索するキーと、ある程度のリクエストメタデータを含んでいます。これには、ユーザー IP アドレス、リクエストタイムスタンプ、[experiment id](https://github.com/WICG/turtledove/issues/191)が含まれる。我々は、UDF がこのメタデータのいくつかにアクセスすることを許可すると予期しており、これがプライバシーモデルにどのように適合するかを検討した後、それに関する詳細でこの説明書を更新する予定である。
- オープンソースの必要なし - UDF はアドテクが提供するものであり、公開・共有する必要はない。

### API

We'll update the [API explainer](https://github.com/WICG/turtledove/blob/main/FLEDGE_Key_Value_Server_API.md) with the APIs that we plan to provide for UDFs.

UDF で提供予定の API については、[API 説明書](https://github.com/WICG/turtledove/blob/main/FLEDGE_Key_Value_Server_API.md)を更新していく予定です。

## Open questions

Explainers are the first step in the standardization process followed by The Privacy Sandbox proposals. The key/value service is not finalized. We anticipate community feedback which will lead to improved designs and proposed implementations. There are many ways to provide feedback on this proposal and participate in ongoing discussions, which includes commenting on the issues below, [opening new issues in this repository](https://github.com/WICG/turtledove/issues), or attending a [WICG meeting](https://github.com/WICG/turtledove/issues/88). We intend to incorporate and iterate based on feedback.

Explainers は、The Privacy Sandbox の提案に続く標準化プロセスの最初のステップとなるものです。Key/Value サービスは最終的なものではありません。我々はコミュニティからのフィードバックを期待しており、それが設計の改善や実装の提案につながるだろう。この提案に対するフィードバックを提供し、進行中の議論に参加するには、以下の課題へのコメント、[このリポジトリで新しい課題を開く](https://github.com/WICG/turtledove/issues)、[WICG ミーティング](https://github.com/WICG/turtledove/issues/88)に参加するなど、多くの方法がある。私たちは、フィードバックを取り入れ、反復するつもりです。

- How will the open source project be managed and what will be the process for accepting contributions?
- What feature set will this service grow to include?
- What will the system architecture and design look like for sharding of large amounts of data to be stored?
- What will the release schedule be and how will rollback work?
- How will monitoring and alerting work?
- How will debugging be supported?
