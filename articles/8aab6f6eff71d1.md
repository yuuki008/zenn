---
title: "Amazon Advertising APIを叩くまで by JS"
emoji: "🌐"
type: "tech"
topics:
  - "メモ"
  - "amazon"
  - "advertisingapi"
published: true
published_at: "2021-08-07 23:04"
---

## はじめに

業務にて Amazon の advertising-api を調べていたのですが、まだまだ日本語記事が出回っておらず、公式の英語ドキュメントのみでした。
私自身苦労したので、ここで一部公式を抜粋しながら認証を通して、実際に API を叩くところまで紹介していきます！！

**なお今回は、js を使い、ウェブブラウザからサードパーティとして、認証情報を取得して API を叩きます**

## Advertising API って？

> The Amazon Advertising API lets you programmatically manage your advertising operations.

**いわゆる広告操作をプログラムで管理できるということですね！**
以下公式ドキュメント
[https://advertising.amazon.com/API/docs/en-us/what-is/amazon-advertising-api](https://advertising.amazon.com/API/docs/en-us/what-is/amazon-advertising-api)

##### API の種類

- sponsored brand -> 商品検索結果の上部に掲載される広告
- sponsored display -> 設定したターゲティングに基づきさまざまな場所に表示される広告
- sponsored product -> 設定したキーワードが検索されると表示される広告
- Amazon DSP -> Amazon 以外の外部広告

今回は、API を叩くまでが目標なので詳細は、省かせていただきます。
**Amazon 広告ついて詳しく知りたい方は、以下の記事からどうぞ！！**
[【Amazon 広告のキホン！】メリットや種類・仕組みを徹底解説！](https://www.hideandseek.co.jp/archives/2347)

## セキュリティプロファイルの作成

[https://developer.amazon.com/loginwithamazon/console/site/lwa/overview.html](https://developer.amazon.com/loginwithamazon/console/site/lwa/overview.html)

1. 上記のリンクからセキュリティプロファイルを作成
2. 作成した、プロファイルの設定ボタンからウェブ設定を選択し、編集をクリック
3. その後、許可されたオリジンと許可された返信 URL に任意の URL を登録
   **今回は、テストケースになるので、どちらとも`http://localhost:3000`で登録します。**
   **なおこの URL は、複数登録が可能です！**
4. クライアント ID とクライアントシークレットをコピー（後程使用）
   ![](https://storage.googleapis.com/zenn-user-upload/111316fe1629ac023dd1f6e0.png)

セキュリティプロファイルの作成方法詳細は、以下のリンクから
[https://developer.amazon.com/ja/docs/login-with-amazon/register-web.html](https://developer.amazon.com/ja/docs/login-with-amazon/register-web.html)

# 実装

![](https://d3a0d0y2hgofx6.cloudfront.net/en-us/_images/setting-up/ad-api-oauth20-flow.png)

1. login with amazon ボタンをユーザーがクリック後、同意フォームにリダイレクト
   （ユーザーが認証を許可後、`YOUR_RETURN_URL`にリダイレクトされる。この時、`params`には`code`が含まれる）
2. `params`の`code`を使い、`access_token`と`refresh_token`を取得
3. `access_token`を使い、ユーザーの`profile`を取得
4. `access_token`と`profileId`を使い、api を叩く

この流れで、認証を打破していきます！

**login with amazon ボタンをサイト内に設置**
クリック後、以下の URL（同意フォーム）にリダイレクトさせます
`https://www.amazon.com/ap/oa?client_id=YOUR_LWA_CLIENT_ID&scope=advertising::campaign_management&response_type=code&redirect_uri=YOUR_RETURN_URL`

ユーザーが認証を許可すると以下の状態で`YOUR_RETURN_URL`にリダイレクトされます。
`http://localhost:3000?code=xxxxxxxxxxxxxxxxxxx&scope=cpc_advertising%3Acampaign_management`

params の中に`access_token`と交換できる`code`が含まれているため、以下のように`access_token`を取得します。

```
curl  \
    -X POST \
    -H "Content-Type:application/x-www-form-urlencoded;charset=UTF-8" \
    --data "grant_type=authorization_code&code=AUTH_CODE&redirect_uri=YOUR_RETURN_URL&client_id=YOUR_CLIENT_ID&client_secret=YOUR_SECRET_KEY" \
    https://api.amazon.co.jp/auth/o2/token
```

```js
// nodejs
const exchangeAccessToken = async (code) => {
  const response = await fetch("https://api.amazon.co.jp/auth/o2/token", {
    method: "post",
    headers: {
      "Content-Type": "application/json;charset=UTF-8",
    },
    body: JSON.stringify({
      grant_type: "authorization_code",
      code: YOUR_AUTH_CODE,
      redirect_uri: "http://localhost:3000",
      client_id: YOUR_CLIENT_ID,
      client_secret: YOUR_CLIENT_SECRET,
    }),
  })
    .then((res) => res.json())
    .catch((err) => false);
  return response;
};
```

その`access_token`を使って以下のように`profile`を取得

```
  curl \
    -X GET \
    -H "Content-Type:application/json" \
    -H "Authorization: Bearer YOUR\_ACCESS\_TOKEN" \
    -H "Amazon-Advertising-API-ClientId: YOUR_CLIENT_ID" \
    https://advertising-api-fe.amazon.com/v2/profiles
```

```js
// nodejs
const getProfile = async(access_token) => {
  const response = await fetch('https://advertising-api-fe.amazon.com/v2/profiles', {
    method: 'get',
    headers: {
      "Content-Type": "application/json",
      "Authorization": "Bearer " + YOUR_ACCESS_TOKEN,
      "Amazon-Advertising-API-ClientId": YOUR_CLIENT_ID
    }
  }).then(res => res.json()).catch(error => false)
  return response
)}
```

**`access_token`は、１時間のみ有効
`refresh_token`は、対象ユーザーに対して期間関係なく使えるもので、以下のように`access_token`を交換することができます**

```
curl \
    -X POST \
    -H "Content-Type:application/x-www-form-urlencoded;charset=UTF-8" \
    --data "grant_type=refresh_token&client_id=YOUR_CLIENT_ID&refresh_token=YOUR_REFRESH_TOKEN&client_secret=YOUR_CLIENT_SECRET" \
    https://api.amazon.com/auth/o2/token
```

`profileId`と`access_token`を取得できたら、以下のようにすることで API が叩けます

```
  curl \
    -X GET \
    -H "Content-Type:application/json" \
    -H "Authorization: Bearer YOUR\_ACCESS\_TOKEN" \
    -H "Amazon-Advertising-API-ClientId: YOUR_CLIENT_ID" \
    -H "Amazon-Advertising-API-Scope": YOUR_PROFILE_ID
    https://advertising-api-fe.amazon.com/v2/sp/campaigns
```

```js
const getSpCampaigns = async () => {
  const response = await fetch(
    "https://advertising-api-fe.amazon.com/v2/sp/campaigns",
    {
      method: "get",
      headers: {
        "Content-Type": "application/json",
        Authorization: "Bearer " + YOUR_ACCESS_TOKEN,
        "Amazon-Advertising-API-ClientId": YOUR_CLIENT_ID,
        "Amazon-Advertising-API-Scope": YOUR_PROFILE_ID,
      },
    }
  )
    .then((res) => res.json())
    .catch((error) => false);
  return response;
};
```

### 終わりに

実際に API を叩くまで長いですよね。
私自身まだ駆け出しなので、苦労しましたが、ドキュメントを早く理解して、すぐ実装できるようになりたいです！切実に、、、
[https://advertising.amazon.com/API/docs/en-us/what-is/amazon-advertising-api](https://advertising.amazon.com/API/docs/en-us/what-is/amazon-advertising-api)
