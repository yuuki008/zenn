---
title: "Amazon Selling-Partner APIを叩くまで by JS"
emoji: "🐷"
type: "tech"
topics:
  - "aws"
  - "javascript"
  - "amazon"
  - "spapi"
  - "sellingpartnerapi"
published: true
published_at: "2021-09-04 22:32"
---

### はじめに

業務にて Amazon の Selling-Partner-API について調査していたのですが、認証周りが複雑なのとネット上の情報が少なく、`javascript`で API を叩けるまで、恥ずかしながらものすごく時間がかかってしまいました。そのため、この記事では`javascript` で認証を乗り越え、実際に API を叩くところまで紹介していきます。
**`login with amazon(LWA)` という自身のサイトから Amazon のログイン画面にリダイレクトさせる認可サービスがありますが、今回はコードのみで認証を解いていくので使用しません**。
※別で記事にする予定です！

[https://github.com/yuuki008/auth_sp-api](https://github.com/yuuki008/auth_sp-api)このリポジトリにて、すぐ API を叩けるので、とりあえず叩いてみたい方は、どーぞ！

### Selling-Partner API とは

[公式ドキュメント](https://qiita.com/captainUmaru/items/a39af04124713557cad2)抜粋

> 出品パートナー API は RESET ベースの API で、これにより出品者は出品、注文、支払い、レポートなどのデータにプログラムからアクセスすることができます。出品パートナー API を使用するアプリケーションは、販売効率を高め、必要な労働量を削減し、顧客への応答時間を短縮し、出品者がビジネスを成長させるのに役立ちます。出品パートナー API は、Amazon マーケットプレイス Web サービス（Amazon MWS）の機能に基づいて構築されており、開発者や出品者パートナーの使いやすさとセキュリティを向上させる機能を提供します。

**商品の注文情報、配送情報、決済情報 etc...を取得できるようになるようです**

### あらかじめ

[公式ドキュメント](https://qiita.com/captainUmaru/items/a39af04124713557cad2)の**出品パートナー API アプリケーションの登録**のセクションが完了していることが前提として進めていきます！

**必要なもの**

1. AWS のアカウント（accesskey, secretkey）
2. AWS の IAM ロールの ARN
3. セラーセントラルで登録したアプリ（client_id, client_secret, refresh_token)

# 実装

![https://storage.googleapis.com/zenn-user-upload/723d5c9cbb77193634ed7e79.png](https://storage.googleapis.com/zenn-user-upload/723d5c9cbb77193634ed7e79.png)

上記の写真だとシンプルでわかりやすいのですが、いくつか難関があります。

1. access_token を取得
2. AWS STS(security token service)
3. AWS Signature(AWS 著名）
4. API を叩く

### 1. access_token を取得

```jsx
const getAccessToken = async () => {
  const response = await fetch("https://api.amazon.com/auth/o2/token", {
    method: "post",
    headers: {
      "Content-Type": "application/json",
      "Content-Length": 524,
    },
    body: JSON.stringify({
      grant_type: "refresh_token",
      client_id: YOUR_APP_CLIENT_ID,
      client_secret: YOUR_APP_CLIENT_SECRET,
      refresh_token: YOUR_APP_REFRESH_TOKEN,
    }),
  }).then((res) => res.json());
  return response;
};
```

client_id, client_secret, refresh_token のパラメーターにそれぞれ任意の値を入れてください!
**成功レスポンス**

```
{
  access_token: 'Atza|IwEBI...',
  refresh_token: 'Atzr|IwEBI...',
  token_type: 'bearer',
  expires_in: 3600
}
```

### 2. AWS STS(security token service)

[こちら](https://www.fenet.jp/aws/column/tool/1765/#:~:text=AWS%20STS%E3%81%A8%E3%81%AFAWS,%E3%81%99%E3%82%8B%E3%81%93%E3%81%A8%E3%81%8C%E3%81%A7%E3%81%8D%E3%81%BE%E3%81%99%E3%80%82)から抜粋

> AWS STS とは AWS Security Token Service の略で AWS リソースへアクセスするための一時的なセキュリティ認証情報を提供するためのサービスです。
> 一時的なセキュリティキーを作成することで、信頼するユーザーへ AWS リソースへのアクセスを許可することができます。

```jsx
const AWS = require("aws-sdk");
const SESConfig = {
  credentials: new AWS.Credentials(YOUR_AWS_ACCESS_KEY, YOUR_AWS_SECRET_KEY),
  region: "us-west-2",
};
const STS = new AWS.STS();

const createTemporaryAWSCredentials = async () => {
  STS.config.update(SESConfig);
  const response = await STS.assumeRole({
    RoleArn: YOUR_ROLE_ARN,
    RoleSessionName: YOUR_ROLE_ARN_NAME,
  }).promise();
  return response.Credentials;
};
```

- YOUR_AWS_ACCESS_KEY
- YOUR_AWS_SECRET_KEY
- YOUR_ROLE_ARN
- YOUR_ROLE_ARN_NAME (arn のスラッシュから右）

上記は、任意の値を入れてください
あと aws-sdk を使うので、インストールを、、、

```
$ npm install aws-sdk
```

**aws の id,secret で値を更新し、STS の assumeRole 関数に作成した Role の値を渡すと、以下のような値が返ってきます！**

**レスポンス**

```
{
  AccessKeyId: 'AS...',
  SecretAccessKey: '/aJKD...',
  SessionToken: 'FwoGZXIv...',
  Expiration: 2021-09-03T14:13:44.000Z
}
```

### 3. AWS Signature(AWS 著名）

[こちら](https://qiita.com/sivertigo/items/e5d877c36ea034412437)から抜粋

> AWS の API を利用するときに、クライアント側から I AM ユーザであることを示す為に送付する署名のことです。HTTP リクエストの Authorization ヘッダーの値に載せて送ります。それを受け取った AWS 側では、同じロジックに従って署名を作り、送られてきた署名と一致するかを確かめることで認証を行います。

以下のコードで Authorization header を作成します。複雑な処理なのでコピペでいいかと、、、

```jsx
const qs = require("qs");
const crypto = require("crypto-js");
const fetch = require("node-fetch");

const getAuthorizationHeader = (access_token, role_credentials, req_params) => {
  req_params.query = sortQuery(req_params.query);
  let iso_date = createUTCISODate();

  let encoded_query_string = constructEncodedQueryString(req_params.query);
  let canonical_request = constructCanonicalRequestForAPI(
    access_token,
    req_params,
    encoded_query_string,
    iso_date
  );
  let string_to_sign = constructStringToSign(
    "us-west-2",
    "execute-api",
    canonical_request,
    iso_date
  );
  let signature = constructSignature(
    "us-west-2",
    "execute-api",
    string_to_sign,
    role_credentials.secret,
    iso_date
  );

  return {
    method: req_params.method,
    url: constructURL(req_params, encoded_query_string),
    body: req_params.body ? JSON.stringify(req_params.body) : null,
    headers: {
      Authorization:
        "AWS4-HMAC-SHA256 Credential=" +
        role_credentials.id +
        "/" +
        iso_date.short +
        "/" +
        "us-west-2" +
        "/execute-api/aws4_request, SignedHeaders=host;x-amz-access-token;x-amz-date, Signature=" +
        signature,
      "Content-Type": "application/json; charset=utf-8",
      host: "sellingpartnerapi-fe.amazon.com",
      "x-amz-access-token": access_token,
      "x-amz-security-token": role_credentials.security_token,
      "x-amz-date": iso_date.full,
    },
  };
};

const constructURL = (req_params, encoded_query_string) => {
  let url = "https://sellingpartnerapi-fe.amazon.com" + req_params.api_path;
  if (encoded_query_string !== "") {
    url += "?" + encoded_query_string;
  }
  return url;
};

const sortQuery = (query) => {
  if (query && Object.keys(query).length) {
    return Object.keys(query)
      .sort()
      .reduce((r, k) => ((r[k] = query[k]), r), {});
  }
  return;
};

const createUTCISODate = () => {
  let iso_date = new Date().toISOString().replace(/[:\-]|\.\d{3}/g, "");
  return {
    short: iso_date.substr(0, 8),
    full: iso_date,
  };
};

const constructEncodedQueryString = (query) => {
  if (query) {
    query = qs.stringify(query, { arrayFormat: "comma" });
    let encoded_query_obj = {};
    let query_params = query.split("&");
    query_params.map((query_param) => {
      let param_key_value = query_param.split("=");
      encoded_query_obj[param_key_value[0]] = param_key_value[1];
    });
    encoded_query_obj = sortQuery(encoded_query_obj);
    let encoded_query_arr = [];
    for (let key in encoded_query_obj) {
      encoded_query_arr.push(key + "=" + encoded_query_obj[key]);
    }
    if (encoded_query_arr.length) {
      return encoded_query_arr.join("&");
    }
  }
  return "";
};

const constructCanonicalRequestForAPI = (
  access_token,
  params,
  encoded_query_string,
  iso_date
) => {
  let canonical = [];
  canonical.push(params.method);
  canonical.push(params.api_path);
  canonical.push(encoded_query_string);
  canonical.push("host:sellingpartnerapi-fe.amazon.com");
  canonical.push("x-amz-access-token:" + access_token);
  canonical.push("x-amz-date:" + iso_date.full);
  canonical.push("");
  canonical.push("host;x-amz-access-token;x-amz-date");
  canonical.push(crypto.SHA256(params.body ? JSON.stringify(params.body) : ""));
  return canonical.join("\n");
};

const constructStringToSign = (
  region,
  action_type,
  canonical_request,
  iso_date
) => {
  let string_to_sign = [];
  string_to_sign.push("AWS4-HMAC-SHA256");
  string_to_sign.push(iso_date.full);
  string_to_sign.push(
    iso_date.short + "/" + region + "/" + action_type + "/aws4_request"
  );
  string_to_sign.push(crypto.SHA256(canonical_request));
  return string_to_sign.join("\n");
};

const constructSignature = (
  region,
  action_type,
  string_to_sign,
  secret,
  iso_date
) => {
  let signature = crypto.HmacSHA256(iso_date.short, "AWS4" + secret);
  signature = crypto.HmacSHA256(region, signature);
  signature = crypto.HmacSHA256(action_type, signature);
  signature = crypto.HmacSHA256("aws4_request", signature);
  return crypto.HmacSHA256(string_to_sign, signature).toString(crypto.enc.Hex);
};
```

**AWS Signature(著名)に必要な値**

1. API を叩く時のクエリパラメーターと body 要素
2. AWS（account_key, secret_key, security_token)
3. リージョン(us-west-2)、サービス名(execute-api)
4. access_token

上記の値を元に著名を作成しています。
**AWS 側でも著名が作成され、一致すると認証が通ります！！**

作成した関数ですが、以下のように呼び出してください

```jsx
const req_params = {
  api_path: "/reports/2020-09-04/reports",
  method: "GET",
  query: { reportTypes: ["GET_V2_SETTLEMENT_REPORT_DATA_FLAT_FILE"] },
};

const demoFunc = async () => {
  // 1で作成した関数
  const auth_token = await getAccessToken();
  // 2で作成した関数
  const credentials = await createTemporaryAWSCredentials();
  const role_credentials = {
    id: credentials.AccessKeyId,
    secret: credentials.SecretAccessKey,
    security_token: credentials.SessionToken,
  };
  // 3で作成した関数
  let auth = await getAuthorizationHeader(
    auth_token.access_token,
    role_credentials,
    req_params
  );
};
```

```bash
{
  method: 'GET',
  url: 'https://sellingpartnerapi-fe.amazon.com/reports/2020-09-04/reports?reportTypes=GET_V2_SETTLEMENT_REPORT_DATA_FLAT_FILE',
  body: null,
  headers: {
    Authorization: 'AWS4-HMAC-SHA256 Credential=AS.../20210904/us-west-2/execute-api/aws4_request, SignedHeaders=host;x-amz-access-token;x-amz-date, Signature=442...',
    'Content-Type': 'application/json; charset=utf-8',
    host: 'sellingpartnerapi-fe.amazon.com',
    'x-amz-access-token': 'Atza|Iw...',
    'x-amz-security-token': 'FwoGZXI...'
    'x-amz-date': '20210904T114649Z'
  }
}
```

### API を叩く

`getAuthorizationHeader` のレスポンスは、上記のようになるのでこれを使い、いよいよ API を叩きます！！

```jsx
// 3で作成した関数
let auth = await getAuthorizationHeader(
  auth_token.access_token,
  role_credentials,
  req_params
);
const response = await fetch(auth.url, {
  method: auth.method,
  headers: auth.headers,
})
  .then((res) => res.json())
  .catch((err) => console.log(err));
console.log(response);
```

```jsx
{
  payload: [
    {
      reportType: 'GET_V2_SETTLEMENT_REPORT_DATA_FLAT_FILE',
      processingEndTime: '2021-08-31T05:07:49+00:00',
      processingStatus: 'DONE',
      marketplaceIds: [Array],
      reportDocumentId: '...',
      reportId: '...',
      dataEndTime: '2021-08-31T04:26:51+00:00',
      createdTime: '2021-08-31T05:07:49+00:00',
      processingStartTime: '2021-08-31T05:07:49+00:00',
      dataStartTime: '2021-08-17T04:26:52+00:00'
    },
    ...
  ]
}
```

上記のようなレスポンスが帰ってくれば、成功です！

## 終わりに

お疲れ様でした！
叩けるまでが長い、認証が難しい、情報が少ない（日本語特に）で自分自身ものすごく苦労しましたが、これを読んだ方がスムーズに認証を乗り越えるようになってくれれば嬉しいです！
コードはこちらからどーぞ！
[https://github.com/yuuki008/auth_sp-api](https://github.com/yuuki008/auth_sp-api)

**ここまで説明しましたが、[amazon-sp-api](https://www.npmjs.com/package/amazon-sp-api)という便利なライブラリを使うと、面倒な AWS の著名や access_token の交換などが不要で比較的簡単に API を叩くことができるのでおすすめです！！**

### 参考記事

[AWS STS とは？IAM ユーザーとの違いと使い方について紹介！](https://www.fenet.jp/aws/column/tool/1765/#:~:text=AWS%20STS%E3%81%A8%E3%81%AFAWS,%E3%81%99%E3%82%8B%E3%81%93%E3%81%A8%E3%81%8C%E3%81%A7%E3%81%8D%E3%81%BE%E3%81%99%E3%80%82)
[Amazon 出品パートナー API 開発者ガイド - Qiita](https://qiita.com/captainUmaru/items/a39af04124713557cad2)
