# Bidding and Auction Services API

[The Privacy Sandbox][4] aims to develop technologies that enable more private advertising on the web and mobile devices. Today, real-time bidding and ad auctions are executed on servers that may not provide technical guarantees of security. Some users have concerns about how their data is handled to generate relevant ads and in how that data is shared with ad providers. FLEDGE ([Android][24], [Chrome][5]) provides ways to preserve privacy and limit third-party data sharing by serving personalized ads based on previous mobile app or web engagement.

[The Privacy Sandbox][4]は、Web やモバイル端末上で、よりプライベートな広告を実現するための技術開発を目的としています。現在、リアルタイム入札や広告オークションは、技術的に安全が保証されていないサーバで実行されています。また、ユーザーの中には、広告を生成するために自分のデータがどのように扱われるか、そのデータが広告プロバイダーとどのように共有されるかについて懸念を抱いている人もいる。FLEDGE（[Android][24]，[Chrome][5]） は、以前のモバイルアプリやウェブへの関与に基づいてパーソナライズされた広告を提供することにより、プライバシーを保護し、第三者によるデータ共有を制限する方法を提供します。

For all platforms, FLEDGE may require [real-time services][6]. In the initial [FLEDGE proposal][7], bidding and auction for remarketing ad demand is executed locally. This can demand computation requirements that may be impractical to execute on devices with limited processing power, or may be too slow to render ads due to network latency.

全てのプラットフォームにおいて、FLEDGE は「リアルタイムサービス」[6]を必要とする場合がある。当初の「FLEDGE 提案」[7]では、リマーケティング広告の入札とオークションはローカルで実行されます。このため、処理能力が限られたデバイスで実行するには現実的でない計算要件が要求されたり、ネットワーク遅延のために広告のレンダリングに時間がかかりすぎたりする可能性があります。

The Bidding and Auction Services API proposal outlines a way to allow FLEDGE computation to take place on cloud servers in a trusted execution environment, rather than running locally on a user's device. Moving computations to cloud servers can help optimize the FLEDGE auction, to free up computational cycles and network bandwidth for a device. This is not a requirement for Chrome at this point.

Bidding and Auction Services API 提案では、FLEDGE の計算をユーザーのデバイス上でローカルに実行するのではなく、信頼できる実行環境のクラウドサーバー上で行う方法を概説しています。計算をクラウドサーバーに移行することで、FLEDGE のオークションを最適化し、デバイスの計算サイクルとネットワーク帯域幅を解放することができます。これは、現時点では Chrome の要件ではありません。

This contrasts with [Google Chrome's initial FLEDGE proposal][7], which proposes bidding and auction execution to occur locally. There are other ideas, similar to FLEDGE, that propose server-side auction and bidding, such as [Microsoft Edge's PARAKEET][8] proposal. Unlike PARAKEET and Criteo's Gatekeeper designs, this proposal does not involve any communication between services that cannot be attested.

これは、入札やオークションの実行をローカルで行うことを提案している【Google Chrome の初期 FLEDGE 案】[7]と対照的です。FLEDGE と同様に、サーバーサイドでのオークションや入札を提案するものとして、Microsoft Edge の PARAKEET 提案][8]などがあります。この提案は、PARAKEET や Criteo の Gatekeeper の設計とは異なり、認証できないサービス間の通信を伴わない。

This document focuses on the API design for FLEDGE's bidding and auction services. Based on adtech feedback, changes may be incorporated in the design. This API proposal in this document does not currently cover sell-side and buy-side reporting, but it will be updated later to incorporate both. A document that details system design for bidding and auction services will also be published at a later date.

このドキュメントは、FLEDGE の入札およびオークションサービスの API 設計に焦点を当てています。アドテクノロジーのフィードバックに基づき、デザインに変更が加えられる可能性があります。このドキュメントの API 案は、現在セルサイドとバイサイドのレポーティングをカバーしていませんが、後日両方を取り入れるために更新される予定です。また、入札・オークションサービスのシステム設計の詳細については、後日ドキュメントを公開する予定です。

## Background

Read the [FLEDGE services overview][6] to learn about the environment, trust model, server attestation, request-response encryption, and other details.

[FLEDGE サービス概要][6]を読んで、環境、信頼モデル、サーバ証明書、リクエスト・レスポンス暗号化などの詳細を確認する。

Each FLEDGE service is hosted in a virtual machine (VM) within a secure, hardware-based trusted execution environment (TEE). Adtech platforms operate and deploy FLEDGE services on a public cloud. Adtechs may choose the cloud platform from one of the options that are planned. As cloud customers, adtechs are the owners and only tenants of such VM instances.

各 FLEDGE サービスは、安全なハードウェアベースの信頼できる実行環境（TEE）内の仮想マシン（VM）でホストされています。Adtech プラットフォームは、パブリッククラウド上で FLEDGE サービスを運用・展開します。アドテクノロジーは、予定されているオプションの中からクラウドプラットフォームを選択することができます。クラウドの顧客であるアドテックは、そのような VM インスタンスの所有者であり、唯一のテナントでもあります。

## Bidding and Auction system architecture

The following is a high-level overview of the architecture of the Bidding and Auction system.

以下は、Bidding and Auction システムのアーキテクチャの概要です。

![Architecture diagram.](images/bidding-auction-services-architecture.png)

_In this diagram, one seller and one buyer are represented in the service workflow. In reality, a single seller auction has multiple participating buyers_.

この図では、サービスのワークフローにおいて、1 人の売り手と 1 人の買い手を表現しています。実際には、1 つの売り手オークションに複数の買い手が参加する\_。

1. The client starts the auction process.
   1. The client sends a `SelectWinningAd` request to the `SellerFrontEnd` service. This request includes the seller's auction configuration and encrypted input for each participating buyer.
1. The `SellerFrontEnd` service orchestrates `GetBid` requests to participating buyers’ `BuyerFrontEnd` services.
1. Both the seller and buyer’s front-end services fetch proprietary code.
   1. The `SellerFrontEnd` service fetches the seller’s proprietary code required to score ads.
   1. The `BuyerFrontEnd` services fetch real-time data from the buyer’s key-value service, as well as the buyer owned proprietary code to generate bids.
1. The `BuyerFrontEnd` service sends a `GenerateBids` request to the bidding service. The bidding service returns ad candidates with bids.
1. The `BuyerFrontEnd` selects the top eligible ad candidate and returns the selection to the `SellerFrontEnd` with `AdWithBid`.
1. Once `SellerFrontEnd` has received bids from all buyers, it requests real-time data from the seller’s key/value service required to score the ads for auction.
1. `SellerFrontEnd` sends a `ScoreAds` request to the `auction` service to score and select a winner.
1. `SellerFrontEnd` returns the winning ad and additional data to the client to render.

---

1. クライアントがオークションの処理を開始する。
   1. クライアントは `SelectWinningAd` リクエストを `SellerFrontEnd` サービスに送ります。このリクエストには、売り手のオークション設定と、参加している各バイヤーの暗号化された入力が含まれます。
1. `SellerFrontEnd` サービスは、参加している買い手の`BuyerFrontEnd`サービスに対して`GetBid` リクエストをオーケストレーションします。
1. 売り手と買い手の両方のフロントエンドサービスは、独自のコードを取得します。
   1. SellerFrontEnd`サービスは、広告の採点に必要な販売者の独自コードを取得します。
   1. BuyerFrontEnd` サービスは、買い手のキーバリューサービスからリアルタイムのデータを取得し、買い手が所有する独自のコードも取得して入札を生成します。
1. `BuyerFrontEnd` サービスは `GenerateBids` リクエストを入札サービスへ送信します。入札サービスは広告候補に入札を付けて返します。
1. `BuyerFrontEnd` は最上位の広告候補を選択し、その選択を `AdWithBid` と共に `SellerFrontEnd` に返します。
1. `SellerFrontEnd` は、すべての買い手から入札を受けると、オークションの広告をスコアリングするために必要な売り手のキー/バリューサービスにリアルタイムのデータを要求します。
1. `SellerFrontEnd` は `auction` サービスに `ScoreAds` リクエストを送り、スコアをつけて勝者を選びます。
1. `SellerFrontEnd` は当選した広告と追加データをクライアントに返し、レンダリングします。

### Sell-side platform (SSP) system

The following are the FLEDGE services that are to be operated by an SSP, also referred to as a Seller. Server instances are deployed so that they are co-located in a data center within a cloud region.

以下は、売り手と呼ばれる SSP が運営する FLEDGE サービスである。サーバーインスタンスは、クラウド地域内のデータセンターに併設されるように配置される。

#### SellerFrontEnd service

The `SellerFrontEnd` service orchestrates calls to other adtechs. For a single seller auction, this service sends requests to DSPs participating in the auction for bidding. This service also fetches real-time seller signals and adtech proprietary code required for the auction.

SellerFrontEnd`サービスは、他のアドテクノロジーへの呼び出しをオーケストレーションします。シングルセラーのオークションでは、このサービスはオークションに参加している DSP に入札のリクエストを送信します。また、オークションに必要なリアルタイムのセラーシグナルやアドテック独自のコードも取得します。

_Note: In this model and with the_ [_proposed APIs_][9], _a seller can have auction execution on a cloud platform only when all buyers participating in the auction also operate services for bidding on any supported cloud platform. If required for testing purposes, the client-facing API can be updated to support seller-run auctions in the cloud seamlessly, without depending on a buyer's adoption of services_.

注：このモデルおよび* [*提案された API*][9]では、*オークションに参加するすべての買い手が、サポートされている任意のクラウドプラットフォーム上で入札のためのサービスを操作する場合にのみ、売り手は、クラウド-プラットフォーム上でオークションを実行することができます。テスト目的で必要であれば、クライアント向け API を更新して、買い手のサービス導入に依存することなく、シームレスにクラウド上で売り手の実行するオークションをサポートすることができる\_。

#### Auction service

The `Auction` service only responds to requests from `SellerFrontEnd` service, with no outbound network access.

オークション」サービスは「SellerFrontEnd」サービスからのリクエストにのみ応答し、ネットワークへのアウトバウンドアクセスは行いません。

For every ad auction request, the `Auction` service executes seller owned auction code written in JavaScript or WebAssembly in a sandbox instance within a TEE. Execution of code in the sandbox within a TEE ensures that all input and output (such as logging, file access, and disk access) is disabled, and the service has no network or storage access.

広告オークションのリクエストごとに、`Auction` サービスは JavaScript または WebAssembly で書かれた売り手所有のオークションコードを TEE 内のサンドボックスインスタンスで実行します。TEE 内のサンドボックスでコードを実行すると、すべての入出力（ロギング、ファイルアクセス、ディスクアクセスなど）が無効になり、サービスにはネットワークやストレージへのアクセスがないことが保証されます。

The hosting environment protects the confidentiality of the seller's proprietary code, if the execution happens only in the cloud and proprietary code is fetched in a `SellerFrontEnd` service.

ホスティング環境では、クラウド上でのみ実行され、独自コードは `SellerFrontEnd` サービスで取得される場合、販売者の独自コードの機密性は保護されます。

#### Seller's key/value service

A key/value service is a critical dependency for the auction system. The [FLEDGE key/value service][10] receives requests from the `SellerFrontEnd` service in this architecture (or directly from the client in case bidding and auction runs locally on client's device). The service returns real-time seller data required for auction that corresponds to lookup keys available in buyers' bids (such as `ad_render_urls` or `ad_component_render_urls`).

キー/バリューサービスは、オークションシステムにとって重要な依存関係です。FLEDGE key/value service][10] は、このアーキテクチャの `SellerFrontEnd` サービスからリクエストを受け取る（または、入札とオークションがクライアントのデバイス上でローカルに実行される場合は、クライアントから直接）。このサービスは、買い手の入札で利用可能なルックアップキー（`ad_render_urls`や`ad_component_render_urls`など）に対応するオークションに必要なリアルタイムの売り手データを返します。

The seller’s key/value system may include other services running in a TEE. The details of this system are out of scope of this document.

販売者の Key/Value システムには、TEE で動作する他のサービスが含まれる場合がある。このシステムの詳細は、本書の範囲外である。

### Demand-side platform (DSP) system

This section describes FLEDGE services that will be operated by a DSP, also referred to as a buyer. Server instances are deployed such that they are co-located in a data center within a given cloud region.

このセクションでは、バイヤーとも呼ばれる DSP によって運用される FLEDGE サービスについて説明します。サーバー・インスタンスは、特定のクラウド地域内のデータセンターに併設されるように配置されます。

#### BuyerFrontEnd service

The front-end service of the system that receives requests to generate bids from a `SellerFrontEnd` service. This service fetches real-time bidding signals, buyer signals, and proprietary adtech code that is required for bidding.

SellerFrontEnd`サービスから入札の生成要求を受けるシステムのフロントエンドサービスです。このサービスは、リアルタイムの入札シグナル、買い手シグナル、および入札に必要な独自のアドテクコードを取得します。

_Note: With the proposed APIs, a `BuyerFrontEnd` service can also receive requests directly from the client, such as an Android app or web browser. This is supported during the interim testing phase so that buyers (DSPs) can roll out servers independently without depending on seller (SSP) adoption_.

注：提案された API では、`BuyerFrontEnd`サービスは、Android アプリや Web ブラウザなどのクライアントから直接リクエストを受け取ることもできます。これは、買い手（DSP）が売り手（SSP）の採用に依存せずに独立してサーバーを展開できるように、中間テスト段階でサポートされています\_。

#### Bidding service

The FLEDGE bidding service can only respond to requests from a `BuyerFrontEnd` service, and otherwise has no outbound network access. For every bidding request, the service executes buyer owned bidding code written in JavaScript and (optional) WebAssembly in a sandbox instance within a TEE. All input and output (such as logging, file access, and disk access) are disabled, and the service has no network or storage access.

FLEDGE の入札サービスは、`BuyerFrontEnd`サービスからのリクエストにのみ応答し、それ以外はアウトバウンドネットワークアクセスを持ちません。入札要求ごとに、サービスは JavaScript と（オプションで）WebAssembly で書かれたバイヤー所有の入札コードを、TEE 内のサンドボックスインスタンスで実行します。すべての入出力（ロギング、ファイルアクセス、ディスクアクセスなど）は無効化され、サービスはネットワークやストレージにアクセスすることができません。

This environment protects the confidentiality of a buyer's proprietary code, if the execution happens only in the cloud and proprietary code is fetched in a `BuyerFrontEnd` service.

この環境は、クラウド上でのみ実行され、専有コードが「BuyerFrontEnd」サービスで取得される場合、購入者の専有コードの機密性を保護します。

#### Buyer’s key/value Service

A buyer's key/value service is a critical dependency for the bidding system. The [FLEDGE key/value service][10] receives requests from the `BuyerFrontEnd` service in this architecture (or directly from the client in case bidding and auction runs locally on client's device). The service returns real-time buyer data required for bidding, corresponding to lookup keys (`bidding_signals_keys`).

バイヤーのキー/バリューサービスは、入札システムにとって重要な依存関係です。FLEDGE key/value service][10] は、このアーキテクチャの `BuyerFrontEnd` サービスからリクエストを受け取ります（入札とオークションがクライアントのデバイス上でローカルに実行される場合は、クライアントから直接受け取ります）。このサービスは、入札に必要なリアルタイムのバイヤーデータを、ルックアップキー（`bidding_signals_keys`）に対応させて返します。

The buyer’s key/value system may include other services running in a TEE. The details of this system are out of scope of this document.

購入者の Key/Value システムには、TEE で動作する他のサービスが含まれる場合がある。このシステムの詳細は、本書の範囲外である。

### Dependencies

Through techniques such as prefetching and caching, the following dependencies are in the non-critical path of ad serving.

プリフェッチやキャッシュなどの技術により、広告配信のノンクリティカルパスには以下のような依存関係があります。

- A key management system required for FLEDGE service attestation. Learn more in the [FLEDGE Services public document][6].
- A [K-anonymity service][11] ensures the ad is served to K unique users. This service is a dependency for the `BuyerFrontEnd` service in this architecture. The details are not included in this document, but will be covered in a subsequent document.

## API

This API proposal for FLEDGE services is based on the gRPC framework. [gRPC][12] is an open source, high performance RPC framework built on top of HTTP2 that is used to build scalable and fast APIs. gRPC uses [protocol buffers][13] as the [interface description language][14] and underlying message interchange format.

この FLEDGE サービスの API 提案は、gRPC フレームワーク をベースにしている。[gRPC][12] は HTTP2 上に構築されたオープンソースの高性能 RPC フレームワークで、スケーラブルで高速な API を構築するために使用されます。gRPC は [インタフェース記述言語][14] とメッセージ交換フォーマットの基本として [プロトコルバッファ][13] を使用しています。

FLEDGE services expose RPC API endpoints. In this document, the proposed API definitions use [proto3][15].

FLEDGE サービスは、RPC API エンドポイントを公開します。本書では、提案する API 定義に[proto3][15]を使用する。

All communications between FLEDGE services are RPC and are encrypted. All client-to-server communication is also encrypted. Refer to the [FLEDGE services explainer][6] for more information.

FLEDGE サービス間の通信はすべて RPC であり、暗号化されています。また、クライアントからサーバーへの通信もすべて暗号化されています。詳しくは、[FLEDGE サービス解説][6]を参照してください。

A client, such as an Android app or web browser, can call FLEDGE services using RPC or HTTPS. A proxy service hosted in the same VM instance as the FLEDGE service converts HTTPS requests to RPC. Details of this service are out of scope of this document.

Android アプリや Web ブラウザなどのクライアントは、RPC または HTTPS を使用して FLEDGE サービスを呼び出すことができます。FLEDGE サービスと同じ VM インスタンスでホストされるプロキシサービスは、HTTPS リクエストを RPC に変換します。このサービスの詳細は、このドキュメントの範囲外です。

Requests to FLEDGE services and corresponding responses are encrypted. Every request includes an encrypted payload (`request_ciphertext`) and a raw key version (`key_id`) which corresponds to the public key that is used to encrypt the request. The service that decrypts the request will have to use private keys(corresponding to the same key version) to decrypt the request.

FLEDGE サービスに対するリクエストとそれに対応するレスポンスは暗号化されます。すべてのリクエストは暗号化されたペイロード (`request_ciphertext`) と、リクエストを暗号化するために使われた公開鍵に対応する生の鍵バージョン (`key_id`) を含んでいます。リクエストを復号化するサービスは、同じ鍵バージョンに対応する秘密鍵を使用して、リクエストを復号化する必要があります。

### Public APIs

A client such as an Android app or web browser can access public APIs. Clients can send RPC or HTTPS to FLEDGE service.

Android アプリや Web ブラウザなどのクライアントが公開 API にアクセスすることができます。クライアントは FLEDGE のサービスに対して RPC や HTTPS を送信することができます。

#### Protocol buffer <-> JSON / YAML

Given gRPC APIs are defined as protocol buffers, following are some options to convert protocol buffers to JSON or YAML.

gRPC API はプロトコルバッファとして定義されているため、プロトコルバッファを JSON や YAML に変換するオプションは以下の通りです。

[Proto3][15]には、JSON シリアライザとパーサが組み込まれています。これらのライブラリは多言語に対応しており、プロトコルバッファのメッセージオブジェクトを JSON に変換したり、その逆に変換したりするのに利用できます。

[Proto3][15] has a built-in JSON serializer and parser. These libraries have multi-language support and can be used to convert protocol buffer message objects to JSON and vice-versa.

- C++:
  - [json_util.h][16]
    - `MessageToJsonString()`: Converts Protobuf message to JSON.
    - `JsonStringToMessage()`: Converts JSON to Protobuf message.
- Java:
  - [`JsonFormat.Printer`][17]: Converts Protobuf message to JSON format.
  - [`JsonFormat.Parser`][17]: Converts JSON to Protobuf message.

_Learn more about_ [_Proto3 to JSON mapping_][18].

YAML can also be used with HTTPS. The open source tool [gnostic][19] can convert protocol buffers to and from YAML Open API descriptions.

YAML は HTTPS でも使用することができます。オープンソースのツール [gnostic][19] はプロトコルバッファを YAML Open API の記述と相互変換することができます。

Also, you may refer to the [protoc-gen-openapi plugin][20] to generate Open API output corresponding to a proto definition.

また、プロトの定義に対応する Open API 出力を生成するには、[protoc-gen-openapi plugin][20] を参照するとよいでしょう。

#### SelectWinningAd

The Seller FrontEnd service (`SellerFrontEnd`) exposes an API endpoint (`SelectWinningAd`). A client such as Android device or web browser sends a `SelectWinningAd` RPC or HTTPS request to `SellerFrontEnd`. After processing the request, the `SelectWinningAdResponse` includes the winning ad for the publisher ad slot that would render on the user's device.

Seller FrontEnd サービス (`SellerFrontEnd`) は API エンドポイント (`SelectWinningAd`) を公開する。Android デバイスや Web ブラウザなどのクライアントは、`SelectWinningAd` の RPC または HTTPS リクエストを `SellerFrontEnd` に送信します。リクエストの処理後、`SelectWinningAdResponse`には、ユーザーのデバイスにレンダリングされるパブリッシャー広告スロットの当選広告が含まれます。

_Note: The following API is designed for the desired end state, where sellers and all buyers operate auction and bidding services (respectively) in the cloud. The `SelectWinningAd` API can be updated so that sellers (SSPs) can roll out services in the cloud independently. In such a scenario, `SellerFrontEndService` will not orchestrate bidding requests to buyers, but will still be able to execute auctions in the cloud. This setup is only recommended during the testing and early adoption phase_.

注: 以下の API は、売り手とすべての買い手がそれぞれオークションと入札サービスをクラウドで運用する、望ましい最終状態を想定して設計されています。SelectWinningAd` API は、売り手 (SSP) が独立してクラウドでサービスを展開できるように更新することができます。このようなシナリオでは、`SellerFrontEndService`はバイヤーへの入札要求をオーケストレーションしませんが、クラウド上でオークションを実行することは可能です。このセットアップは、テストと初期の採用段階でのみ推奨されます。

Following is the `SelectWinningAd` API definition.

以下は `SelectWinningAd` の API 定義である。

```
syntax = "proto3";

// Seller's FrontEnd service.
service SellerFrontEnd {
  // Selects a winning ad for the Publisher ad slot that would be
  // rendered on the user's device.
  rpc SelectWinningAd(SelectWinningAdRequest) returns (SelectWinningAdResponse) {}
}

// SelectWinningAdRequest is sent from user's device to the
// SellerFrontEnd Service.
message SelectWinningAdRequest {
  // Unencrypted request.
  message SelectWinningAdRawRequest {
    enum ClientType {
      UNKNOWN = 0;

      // An Android device with Google Mobile Services (GMS).
      // Note: This covers apps on Android and browsers on Android.
      ANDROID = 1;

      // An Android device without Google Mobile Services (GMS).
      ANDROID_NON_GMS = 2;

      // Any browser.
      BROWSER = 3;
    }

    // AuctionConfig required by the seller for ad auction.
    // The auction config includes contextual signals and other data required
    // for auction. This also includes the url to fetch seller's proprietary
    // auction logic and real-time signals. This config is passed from client
    // to SellerFrontEnd service in the umbrella request
    // (SelectWinningAdRequest).
    message AuctionConfig {
      // Custom seller inputs for advertising on Android.
      message CustomSellerInputsForAndroid {
        // To be updated later if any custom fields are required to support
        // Android.
      }

      // Custom seller inputs for advertising on web.
      message CustomSellerInputsForBrowser {
        // This timeout can be specified to restrict the runtime (in
        // milliseconds) of the seller's scoreAd() script for auction.
        int64 seller_timeout_ms = 1;

        // Optional. Component auction configuration can contain additional
        // auction configurations for each seller's "component auction".
        google.protobuf.Struct component_auctions = 2;

        // The Id is specified by the seller to support coordinated experiments
        // with the seller's Key/Value services.
        int64 experiment_group_id = 3;
      }

      // Optional. Custom seller inputs.
      oneof CustomSellerInputs {
        CustomSellerInputsForAndroid custom_seller_inputs_android = 1;

        CustomSellerInputsForBrowser custom_seller_inputs_browser = 2;
      }

      // Url for fetching seller owned auction code.
      // For simplicity, this is one url. However, separate endpoints to fetch
      // JS and WASM may be added later.
      string decision_logic_url = 3;

      // Url endpoint for seller Key/Value lookup.
      string scoring_signals_url = 4;

      /*..........................Contextual Signals.........................*/
      // Contextual Signals refer to seller_signals and auction_signals
      // derived contextually.

      // Seller specific signals that include information about the context
      // (e.g. Category blocks Publisher has chosen and so on). This can
      // not be fetched real-time from Key-Value Server.
      // Represents a JSON object.
      google.protobuf.Struct seller_signals = 5;

      // Information about auction (ad format, size).
      // This information is required for both bidding and auction.
      // Represents a JSON object.
      google.protobuf.Struct auction_signals = 6;

      // Signals about device.
      // Required for both bidding and auction.
      oneof DeviceSignals {
        // A JSON object constructed by Android containing contextual
        // information that SDK or app knows about and that adtech's bidding
        // and auction code can ingest.
        google.protobuf.Struct android_signals = 7;

        // A JSON object constructed by the browser, containing information that
        // the browser knows about and that adtech's bidding and auction code
        // can ingest.
        google.protobuf.Struct browser_signals = 8;
      }
    }

    // Ad request timestamp.
    // Note: The precision will be limited to preserve privacy.
    google.protobuf.Timestamp timestamp = 1;

    // Optional. Required by Android to identify an ad selection request. This
    // is going to be cached on the server side to help with reporting.
    // Note: The precision will be limited to preserve privacy.
    int64 ad_selection_request_id = 2;

    // Encrypted BuyerInput per buyer.
    // The key in the map corresponds to buyer Id that can identify a buyer
    // participating in the auction. Buyer Id can be Adtech enrollment Id with
    // Privacy Sandbox or eTLD+1.
    // The value corresponds to BuyerInput ciphertext that will be ingested by
    // the buyer for bidding. BuyerInput include information about Custom
    // Audience (a.k.a Interest Group), certain signals required for bidding
    // and url endpoints to fetch buyer code. The value is encrypted on client
    // device before that is sent server side.
    // The SellerFrontEnd service sends the BuyerInput ciphertext (value in map)
    // to a BuyerFrontEnd service corresponding to buyer Id (key in map). The
    // SellerFrontEnd doesn't decrypt any BuyerInput ciphertext.
    map<string, bytes> encrypted_input_per_buyer = 3;

    // Includes configuration data required in Remarketing ad auction.
    // Some of the data in AuctionConfig is passed to BuyerFrontEnd.
    AuctionConfig auction_config = 4;

    // Type of end user's device / client, that would help in validating the
    // integrity of an attested client.
    // Note: Not all types of clients will be attested.
    ClientType client_type = 5;

    // Field representing client attestation data will be added later.
  }

  // Encrypted SelectWinningAdRawRequest.
  bytes request_ciphertext = 1;

  // Version of the public key used for request encryption. The service
  // needs use private keys corresponding to same key_id to decrypt
  // 'request_ciphertext'.
  string key_id = 2;
}

message SelectWinningAdResponse {
  // Unencrypted response.
  message SelectWinningAdRawResponse {
    // The ad that will be rendered on the end user's device. This url is
    // validated on the client side to ensure that it actually belongs to
    // Custom Audience (a.k.a Interest Group).
    string ad_render_url = 1;

    // Score of the winning ad.
    double score = 2;

    // Custom Audience (a.k.a Interest Group) name.
    string custom_audience_name = 3;

    // Bid for the winning ad candidate, generated by a buyer participating in
    // the auction.
    double bid_price = 4;

    // The version of the binary running on the server and Guest OS of Secure
    // Virtual Machine. The client may validate that with the version
    // available in open source repo.
    string server_binary_version = 5;
  }

  // Encrypted SelectWinningAdRawResponse.
  bytes response_ciphertext = 1;
}
```

##### Ad

Following is the representation of an ad object.

以下は、広告オブジェクトの表現である。

```
syntax = "proto3";

// An ad object.
// This is downloaded to device as part of Custom Audience (a.k.a Interest
// Group) fetch and then sent to servers.
message Ad {
  // Identifies an ad creative rendering url.
  string ad_render_url = 1;

  // Arbitrary metadata of the ad provided by Buyer.
  string ad_metadata = 2;
}
```

##### BuyerInput

Encrypted `BuyerInput` data corresponding to each buyer participating in the auction, is passed in the umbrella request (`SelectWinningAdRequest`) from the client to the `SellerFrontEnd` service. The `BuyerInput` is encrypted in the client and decrypted in the `BuyerFrontEnd` service operated by the buyer.

オークションに参加する各バイヤーに対応する暗号化された `BuyerInput` データは、クライアントから `SellerFrontEnd` サービスへのアンブレラリクエスト (`SelectWinningAdRequest`) で渡されます。BuyerInput`はクライアントで暗号化され、買い手が運営する `BuyerFrontEnd` サービスで復号されます。

```
syntax = "proto3";

// A BuyerInput includes data that a buyer (DSP) requires to generate bids.
message BuyerInput {
  // CustomAudience (a.k.a InterestGroup) includes the endpoint from where
  // AdTech's proprietary code for bidding will be fetched, the set of ads
  // corresponding to this audience, user bidding signals and other optional
  // fields. Each Custom Audience has a name that is unique for a buyer.
  message CustomAudience {
    // Name or tag of Custom Audience (a.k.a Interest Group).
    string name = 1;

    // Ad creatives belonging to this CustomAudience (a.k.a InterestGroup).
    repeated Ad ads = 2;

    // Url for fetching buyer bidding code.
    string bidding_logic_url = 3;

    // Optional. The buyer may provide computationally-expensive
    // subroutines in WebAssembly (WASM) that can be fetched using this url.
    string bidding_wasm_helper_url = 4;

    // Keys to lookup from buyer Key/Value service.
    string bidding_signals_keys = 5;

    // The endpoint for buyer Key/Value service. This points to that shard of
    // KV Service that has data corresponding to 'bidding_signals_keys'.
    string bidding_signals_url = 6;

    // Optional. This field may be populated for browser but not required for
    // Android at this point.
    //
    // This field contains the various ad components (or "products") that can
    // be used to construct Ads Composed of Multiple Pieces. Each entry is an
    // object that includes both a rendering URL and arbitrary metadata that
    // can be used at bidding time.
    repeated string ad_components = 7;

    // Optional. This field may be set for browser but not required for Android
    // at this point.
    //
    // The priority is used to select which Interest Groups (a.k.a Custom
    // Audiences) participate in an auction when the number of Interest Groups
    // are limited by the buyer_group_limits in BrowserBuyerInput. If not
    // specified, the default value of 0.0 is assigned.
    //
    // These values are only used to select Interest Groups to participate in
    // an auction such that if there is an Interest Group participating in the
    // auction with priority x, all interest groups corresponding to the same
    // buyer having a priority y where y > x should be considered for generating
    // bids. In case due to buyer_group_limits if all Interest Groups with same
    // priority can not participate, then Interest Groups will be uniformly
    // randomly chosen from the set of interest groups with that priority.
    float priority = 8;

    // User bidding signals for storing additional metadata that the Buyer can
    // use during bidding.
    google.protobuf.Struct user_bidding_signals = 9;
  }

  // The Custom Audiences (a.k.a Interest Groups) corresponding to the buyer.
  repeated CustomAudience custom_audiences = 1;

  // Buyer may provide additional contextual information that could help in
  // generating bids. Not fetched real-time.
  // Represents a JSON object.
  google.protobuf.Struct buyer_signals = 2;

  // Custom buyer inputs for advertising on Android.
  message CustomBuyerInputsForAndroid {
    // To be updated later if any custom fields are required to support Android.
  }

  // Custom buyer inputs for advertising on browsers.
  message CustomBuyerInputsForBrowser {
    // This timeout can be specified to restrict the runtime (in milliseconds)
    // of the buyer's generateBid() scripts for bidding. This can also be a
    // default value if timeout is unspecified for the buyer.
    int64 buyer_timeout_ms =  1;

    // This can be specified to limit the number of Interest Groups / Custom
    // Audiences from a particular buyer that participates in the auction. This
    // can also be a default value if the limit is unspecified for the buyer.
    int64 buyer_group_limit = 2;

    // The Id is specified by the buyer to support coordinated experiments with
    // the buyer's Key/Value services.
    int64 experiment_group_id = 3;
  }

  // Optional. Custom buyer input for app or web advertising.
  oneof CustomBuyerInputs {
    CustomBuyerInputsForAndroid custom_buyer_inputs_android = 3;

    CustomBuyerInputsForBrowser custom_buyer_inputs_browser = 4;
  }
}
```

#### GetBid

The `BuyerFrontEnd` service exposes an API endpoint `GetBid`. The `SellerFrontEnd` service sends `GetBidRequest` to the `BuyerFrontEnd` service with encrypted `BuyerInput` and other data. After processing the request, `BuyerFrontEnd` returns `GetBidResponse`, which includes a bid and other data corresponding to the top eligible ad candidate. Refer to [`AdWithBid`][22] for more information.

BuyerFrontEnd`サービスは API エンドポイント`GetBid` を公開しています。SellerFrontEnd` サービスは `GetBidRequest` を `BuyerFrontEnd` サービスに送信し、暗号化された `BuyerInput` とその他のデータを送信する。BuyerFrontEnd` はリクエストを処理した後、`GetBidResponse`を返します。このレスポンスには、上位の広告候補に対応する入札額とその他のデータが含まれます。詳細は [`AdWithBid`][22] を参照してください。

The communication between the `BuyerFrontEnd` service and the `SellerFrontEnd` service is between each service’s TEE and is end-to-end encrypted.

BuyerFrontEnd`サービスと`SellerFrontEnd` サービス間の通信は、各サービスの TEE 間で行われ、エンドツーエンドで暗号化されています。

_Note: Temporarily, as adtechs test these systems, clients can call `BuyerFrontEnd` services directly using the API below_.

注：一時的に、アドテクがこれらのシステムをテストしている間、クライアントは以下の API を使用して `BuyerFrontEnd` サービスを直接呼び出すことができます\_。

```
syntax = "proto3";

// Buyer’s FrontEnd service.
service BuyerFrontEnd {
  // Returns bid for the top eligible ad candidate.
  rpc GetBid(GetBidRequest) returns (GetBidResponse) {}
}

// GetBidRequest is sent by the `SellerFrontEnd` Service to the `BuyerFrontEnd`
// service.
message GetBidRequest{
  // Unencrypted request.
  message GetBidRawRequest {
    // Whether this is a fake request from SellerFrontEnd service
    // and should be dropped.
    // Note: `SellerFrontEnd` service will send chaffs to a few other buyers
    // not participating in the auction. This is required for privacy reasons
    // to prevent seller from figuring the buyers by observing the network
    // traffic to `BuyerFrontEnd` Services, outside of TEE.
    bool is_chaff = 1;

    // Encrypted BuyerInput corresponding to the buyer.
    // This includes CustomAudiences (a.k.a InterestGroups) owned by
    // the buyer, some signals required for generating bids and url endpoints
    // from where buyer code can be fetched.
    // Encrypted on client device and passed to server.
    bytes buyer_input_ciphertext = 2;

    // Information about auction (ad format, size) derived contextually.
    // Represents a JSON object.
    // Copied from Auction Config in SellerFrontEnd service.
    google.protobuf.Struct auction_signals = 3;

    // Signals about client device.
    // Copied from Auction Config in SellerFrontEnd service.
    oneof DeviceSignals {
      // A JSON object constructed by Android containing contextual
      // information that SDK or app knows about and that adtech's bidding
      // and auction code can ingest.
      google.protobuf.Struct android_signals = 4;

      // A JSON object constructed by the browser, containing information that
      // the browser knows about and that adtech's bidding and auction code
      // can ingest.
      google.protobuf.Struct browser_signals = 5;
    }
  }

  // Encrypted GetBidRawRequest.
  bytes request_ciphertext = 1;

  // Version of the public key used for request encryption. The service
  // needs use private keys corresponding to same key_id to decrypt
  // 'request_ciphertext'.
  string key_id = 2;
}

// Response to GetBidRequest.
message GetBidResponse {
  // Unencrypted response.
  message GetBidRawResponse {
    // Includes ad_render_url and corresponding bid value pairs.
    // Represents a JSON object.
    AdWithBid bid = 1;
  }

  // Encrypted GetBidRawResponse.
  bytes response_ciphertext = 1;
}
```

##### AdWithBid

The bid for an ad candidate, includes `ad_render_url, ad_metadata, custom_audience_name` and corresponding `bid_price`. This is returned in [`GetBidResponse`][23].

広告候補の入札には、`ad_render_url, ad_metadata, custom_audience_name` とそれに対応する `bid_price` が含まれます。これは、[`GetBidResponse`][23]で返されます。

```
syntax = "proto3";

// Bid for an ad candidate.
message AdWithBid {
  // Identifies an ad creative.
  string ad_render_url = 1;

  // Metadata of the ad, this will be passed to Seller's scoring function.
  google.protobuf.Struct ad_metadata = 2;

  // Name of the Custom Audience / Interest Group this ad belongs to.
  string custom_audience_name = 3;

  // Bid price corresponding to an ad.
  double bid_price = 4;
}
```

### Internal APIs

Internal APIs refer to the interface for communication between FLEDGE services within a SSP system or DSP system.

内部 API とは、SSP システムまたは DSP システム内の FLEDGE サービス間の通信用インターフェイスを指します。

#### GenerateBids

The `Bidding` service exposes an API endpoint `GenerateBids`. The `BuyerFrontEnd` service sends `GenerateBidsRequest` to the `Bidding` service, which includes the buyer's proprietary bidding code and required input. After processing the request, the `Bidding` service returns the `GenerateBidsResponse` which includes bids that correspond to each ad (`AdWithBid`).

Bidding`サービスは API エンドポイント`GenerateBids` を公開しています。BuyerFrontEnd` サービスは `GenerateBidsRequest` を `Bidding` サービスに送信します。このリクエストには、バイヤー独自の入札コードと必要な入力が含まれています。リクエストの処理後、`Bidding` サービスは各広告に対応する入札額（`AdWithBid`）を含む `GenerateBidsResponse` を返します。

The communication between the `BuyerFrontEnd` service and `Bidding` service occurs between each service’s TEE and request-response is end-to-end encrypted.

BuyerFrontEnd」サービスと「Bidding」サービス間の通信は、各サービスの TEE 間で行われ、リクエストとレスポンスはエンドツーエンドで暗号化されています。

```
syntax = "proto3";

// Bidding service operated by buyer.
service Bidding {
  // Generate bids for ads in Custom Audiences (a.k.a InterestGroups) and
  // filters ads.
  rpc GenerateBids(GenerateBidsRequest) returns (GenerateBidsResponse) {}
}

// Generate bids for all Custom Audiences (a.k.a InterestGroups) corresponding
// to the Buyer.
message GenerateBidsRequest {
  // Unencrypted request.
  message GenerateBidsRawRequest {
    // Custom Audience (a.k.a Interest Group) for bidding.
    message CustomAudienceForBidding {
      // Unique string that identifies the Custom Audience (a.k.a Interest
      // Group) for a buyer.
      string name = 1;

      // Ad creative render urls belonging to the Custom Audience (a.k.a
      // Interest Group).
      repeated Ad ads = 2;

      // User bidding signals for storing additional metadata that the buyer
      // can use during bidding.
      google.protobuf.Struct user_bidding_signals = 3;

      // Optional. This field may be populated for browser but not required
      // for Android at this point.
      //
      // This field contains the various ad components (or "products") that
      // can be used to construct Ads Composed of Multiple Pieces. Each entry
      // is an object that includes both a rendering URL and arbitrary
      // metadata that can be used at bidding time.
      repeated string ad_components = 4;

      // Optional. This field may be set for browser but not required for
      // Android at this point.
      //
      // The priority is used to select which Interest Groups (a.k.a Custom
      // Audiences)
      // participate in an auction when the number of Interest Groups are
      // limited by the buyer_group_limits in BrowserBuyerInput. If not
      //  specified, the default value of 0.0 is assigned.
      //
      // These values are only used to select Interest Groups to participate
      // in an auction such that if there is an Interest Group participating
      // in the auction with priority x, all interest groups corresponding to
      // the same buyer having a priority y where y > x should be considered
      // for generating bids. In case due to buyer_group_limits if all
      // Interest Groups with same priority can not participate, then Interest
      // Groups will be uniformly randomly chosen from the set of interest
      // groups with that priority.
      float priority = 5;
    }

    // Buyer logic per Custom Audience (a.k.a Interest Group).
    message BuyerCodePerAudience {
      // Note: The code runtime engine can accept JavaScript, WASM code or
      // both.
      //
      // Buyer owned JavaScript for bidding. The JavaScript may include
      // embedded WASM binary code, no file access would be allowed.
      bytes js_code = 1;

      // Optional. Buyer owned WASM code for bidding.
      bytes wasm_code = 2;

      // Custom Audience (a.k.a Interest Group) is an input to bidding code.
      CustomAudienceForBidding custom_audience_for_bidding = 3;

      /*...Real Time signals fetched from buyer’s Key/Value service...*/
      // Key-value pairs corresponding to keys in bidding_signals_keys.
      // Represents a JSON object.
      google.protobuf.Struct bidding_signals = 4;
    }

    // Buyer logic per Custom Audience (a.k.a Interest Group).
    repeated BuyerCodePerAudience buyer_code_per_audiences = 1;

    /********************* Common inputs for bidding ***********************/
    // Information about auction (ad format, size) derived contextually.
    // Represents a JSON object. Copied from Auction Config in SellerFrontEnd
    // service.
    google.protobuf.Struct auction_signals = 2;

    // Optional. Buyer may provide additional contextual information that
    // could help in generating bids. Not fetched real-time.
    // Represents a JSON object.
    //
    // Note: This is passed in encrypted BuyerInput, i.e.
    // buyer_input_ciphertext field in GetBidRequest. The BuyerInput is
    // encrypted in the client and decrypted in `BuyerFrontEnd` Service.
    // This data is copied from BuyerInput.
    google.protobuf.Struct buyer_signals = 3;

    // Signals about client device.
    // Copied from Auction Config in SellerFrontEnd service.
    oneof DeviceSignals {
      // A JSON object constructed by Android containing contextual
      // information that SDK or app knows about and that adtech's bidding
      // and auction code can ingest.
      google.protobuf.Struct android_signals = 4;

      // A JSON object constructed by the browser, containing information that
      // the browser knows about and that adtech's bidding and auction code
      // can ingest.
      google.protobuf.Struct browser_signals = 5;
    }

    /************************ Custom bidding parameters ***********************/
    // Custom parameters for buyer code execution.

    // Custom bidding params for advertising on Android.
    message CustomBiddingParamsForAndroid {
      // To be updated later if any custom fields are required to support Android.
    }

    // Custom bidding params for advertising on web.
    message CustomBiddingParamsForBrowser {
      // This timeout can be specified to restrict the runtime (in milliseconds)
      // of the buyer's generateBid() scripts for bidding. This can also be a
      // default value if timeout is unspecified for the buyer.
      int64 buyer_timeout_ms =  1;

      // This can be specified to limit the number of Interest Groups / Custom
      // Audiences from a particular buyer that participates in the auction. This
      // can also be a default value if the limit is unspecified for the buyer.
      int64 buyer_group_limit = 2;
    }

    // Optional. Custom parameters for bidding.
    oneof CustomBiddingParams {
      CustomBiddingParamsForAndroid custom_bidding_params_android = 6;

      CustomBiddingParamsForBrowser custom_bidding_params_browser = 7;
    }
  }

  // Encrypted GenerateBidsRawRequest.
  bytes request_ciphertext = 1;

  // Version of the public key used for request encryption. The service
  // needs use private keys corresponding to same key_id to decrypt
  // 'request_ciphertext'.
  string key_id = 2;
}

// Encrypted response to GenerateBidsRequest with bid prices corresponding
// to all eligible Ad creatives.
message GenerateBidsResponse {
  // Unencrypted response.
  message GenerateBidsRawResponse {
    // Bids corresponding to ads.
    repeated AdWithBid bids = 1;
  }

  // Encrypted GenerateBidsRawResponse.
  bytes response_ciphertext = 1;
}
```

#### ScoreAds

The `Auction` service exposes an API endpoint `ScoreAds`. The `SellerFrontEnd` service sends a `ScoreAdsRequest` to the `Auction` service with the Seller's proprietary auction code and inputs required by the code. The inputs include bids from each buyer and other required signals. After processing the request, the `Auction` service returns the `ScoreAdsResponse` that includes scores corresponding to each ad.

Auction`サービスは API エンドポイント`ScoreAds` を公開しています。SellerFrontEnd` サービスは `ScoreAdsRequest` を `Auction` サービスに送信し、売り手独自のオークションコードとそのコードに必要な入力を送信します。入力には、各バイヤーの入札やその他の必要な信号が含まれます。リクエストを処理した後、`Auction` サービスは各広告に対応するスコアを含む `ScoreAdsResponse` を返します。

The communication between the `SellerFrontEnd` service and `Auction` service occurs within each service’s TEE and request-response is end-to-end encrypted.

SellerFrontEnd`サービスと`Auction` サービス間の通信は、各サービスの TEE 内で行われ、リクエストとレスポンスはエンドツーエンドで暗号化されています。

```
syntax = "proto3";

// Auction service operated by the seller.
service Auction {
  // Scores all top ad candidates returned by each buyer participating
  // in the auction.
  rpc ScoreAds(ScoreAdsRequest) returns (ScoreAdsResponse) {}
}

// Scores top ad candidates of each buyer.
message ScoreAdsRequest {
  // Unencrypted request.
  message ScoreAdsRawRequest {
    // Note: The code runtime engine can accept JavaScript, WASM code or
    // both.
    // Seller owned JavaScript for auction. The JavaScript may include
    // embedded WASM.
    bytes js_code = 1;

    // Optional; seller owned WASM code for auction.
    // Note: The WASM should be initialized in the JavaScript and local file
    // access is prohibited with the Sandbox where adtech proprietary code would
    // execute.
    bytes wasm_code = 2;

    /**************** Inputs to JavaScript auction code module ****************/

    // Ad with bid.
    // This includes an ad object comprising ad render url and ad metadata,
    // bid corresponding to the ad, the name of Custom Audience (a.k.a Interest
    // Group) the ad belongs to.
    // The ad object and bid will be converted to a JSON objects before passing
    // as inputs to the scoring function. The Custom Audience name is not
    // required as an input for scoring but returned in response back to client
    // to validate that a selected ad actually belongs to the same Custom
    // Audience.
    // Note: Every ad is scored in a different process in an isolated Sandbox
    // within the TEE.
    repeated AdWithBid ad_bids = 3;

    /*....................... Contextual Signals .........................*/
    // Contextual Signals refer to seller_signals and auction_signals
    // derived contextually.

    // Seller specific signals that include information about the context
    // (e.g. Category blocks Publisher has chosen and so on). This can
    // not be fetched real-time from Key-Value Server.
    // Represents a JSON object.
    // Note: This is passed by client in AuctionConfig in SelectWinningAdRequest
    // to SellerFrontEnd service. This data is copied from AuctionConfig.
    google.protobuf.Struct seller_signals = 4;

    // Information about auction (ad format, size). This information
    // is available both to the seller and all buyers participating in
    // auction.
    // Represents a JSON object.
    // Note: This is passed by client in AuctionConfig in SelectWinningAdRequest
    // to SellerFrontEnd service. This data is copied from AuctionConfig.
    google.protobuf.Struct auction_signals = 5;

    /*....................... Real time signals .........................*/
    // Real-time signals fetched from seller Key Value Service.
    // Represents a JSON object.
    // Note: The keys used to look up scoring signals are ad_render_urls and
    // ad_component_render_urls that are part of the bids returned by buyers
    // participating in the auction.
    google.protobuf.Struct scoring_signals = 6;

    // Signals about client device.
    // Copied from Auction Config in SellerFrontEnd service.
    oneof DeviceSignals {
      // A JSON object constructed by Android containing contextual
      // information that SDK or app knows about and that adtech's bidding
      // and auction code can ingest.
      google.protobuf.Struct android_signals = 7;

      // A JSON object constructed by the browser, containing information that
      // the browser knows about and that adtech's bidding and auction code
      // can ingest.
      google.protobuf.Struct browser_signals = 8;
    }

    /************************ Custom auction parameters ***********************/
    // Custom parameters for seller code execution.

    // Custom auction params for advertising on Android.
    message CustomAuctionParamsForAndroid {
      // To be updated later if any custom fields are required to support
      // Android.
    }

    // Custom auction params for advertising on web.
    message CustomAuctionParamsForBrowser {
      // This timeout can be specified to restrict the runtime (in
      // milliseconds) of the seller's scoreAd() script for auction.
      int64 seller_timeout_ms = 1;

      // Optional. Component auction configuration can contain additional
      // auction configurations for each seller's "component auction".
      google.protobuf.Struct component_auctions = 2;
    }

    // Optional. Custom parameters for auction.
    oneof CustomAuctionParams {
      CustomAuctionParamsForAndroid custom_auction_params_android = 9;

      CustomAuctionParamsForBrowser custom_auction_params_browser = 10;
    }
  }

  // Encrypted ScoreAdsRawRequest.
  bytes request_ciphertext = 1;

  // Version of the public key used for request encryption. The service
  // needs use private keys corresponding to same key_id to decrypt
  // 'request_ciphertext'.
  bytes key_id = 2;
}

// Encrypted response that includes auction results returned by adtech's
// auction code.
message ScoreAdsResponse {
  // Identifies a scored ad belonging to a Custom Audience (a.k.a. Interest Group).
  message AdScore {
    // Ad creative render url.
    string ad_render_url = 1;

    // Score of the ad determined during the auction. Any value that is zero or
    // negative indicates that the ad cannot win the auction. The winner of the
    // auction would be the ad that was given the highest score.
    double score = 2;

    // Name of Custom Audience (a.k.a Interest Group) the ad belongs to.
    string custom_audience_name = 3;

    // Bid price corresponding to the winning ad.
    double bid_price = 4;

    /***************** Only relevant to Component Auctions *******************/
    // Additional fields for Component Auctions.

    // Optional. Arbitrary metadata to pass to top level seller.
    string  ad_metadata = 5;

    // Optional for Android, required for Web in case of component auctions.
    // If the bid being scored is from a component auction and this value is not
    // true, the bid is ignored. If not present, this value is considered false.
    // This field must be present and true both when the component seller scores
    // a bid, and when that bid is being scored by the top-level auction.
    bool allow_component_auction = 6;

    // Optional for Android, required for Web in case of component auctions.
    // Modified bid value to provide to the top-level seller script. If
    // present, this will be passed to the top-level seller's scoring function
    // instead of the original bid, if the ad wins the component auction and
    // top-level auction respectively.
    double modified_bid = 7;
  }

  // Unencrypted response.
  message ScoreAdsRawResponse {
    // Scores of ads participating in the auction.
    repeated AdScore ad_scores = 1;
  }

  // Encrypted ScoreAdsRawResponse.
  bytes response_ciphertext = 1;
}
```

[4]: https://privacysandbox.com
[5]: https://developer.chrome.com/docs/privacy-sandbox/fledge/
[6]: https://github.com/privacysandbox/fledge-docs/blob/main/trusted_services_overview.md
[7]: https://github.com/WICG/turtledove/blob/main/FLEDGE.md
[8]: https://github.com/microsoft/PARAKEET
[9]: #selectwinningad
[10]: https://github.com/WICG/turtledove/blob/main/FLEDGE_Key_Value_Server_API.md
[11]: https://github.com/WICG/turtledove/blob/main/FLEDGE_k_anonymity_server.md
[12]: https://grpc.io
[13]: https://developers.google.com/protocol-buffers
[14]: https://en.wikipedia.org/wiki/Interface_description_language
[15]: https://developers.google.com/protocol-buffers/docs/proto3
[16]: https://developers.google.com/protocol-buffers/docs/reference/cpp/google.protobuf.util.json_util
[17]: https://developers.google.com/protocol-buffers/docs/reference/java/com/google/protobuf/util/JsonFormat
[18]: https://developers.google.com/protocol-buffers/docs/proto3#json
[19]: https://github.com/google/gnostic
[20]: https://github.com/google/gnostic/tree/main/cmd/protoc-gen-openapi
[21]: #scoreads
[22]: #adwithbid
[23]: #getbid
[24]: https://developer.android.com/design-for-safety/privacy-sandbox/fledge
