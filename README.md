# 学習ログ

今日やったことを一言投稿すると、みんなの投稿が新しい順に一覧で見られる小さなWebアプリです。
AWSのサーバーレス構成を、実際に作って理解するために作りました。

![構成図](architecture.svg)

## 構成

画面はS3に置いたHTMLファイルで、CloudFront経由で配信しています。
投稿と一覧の取得は、ブラウザからAPI Gateway(HTTP API)を呼び、Lambdaが処理してDynamoDBに保存・取得します。
Lambdaでエラーが起きたときは、CloudWatchアラームとSNSでメールが届きます。

使っているサービスは、S3、CloudFront、API Gateway、Lambda(Python 3.13)、DynamoDB、CloudWatch、SNS、CloudFormationです。

## 工夫したところ

- DynamoDBのキーは、固定の値(`LOG`)と投稿時刻の組み合わせにしました。こうするとScanを使わず、Queryだけで新しい順に取れます。
- Lambdaは投稿用と取得用に分け、投稿用はPutItemだけ、取得用はQueryだけを許可しました。対象もこのアプリのテーブルだけです。
- S3は公開せず、CloudFrontからだけ読めるようにしました。
- APIのCORSは、このアプリのCloudFrontのURLだけを許可しています。
- 投稿用Lambdaで入力チェックをしています。名前は30文字、内容は500文字までで、空の入力や形式の違うJSONは400を返します。
- Lambdaのエラーが出たらメールで気づけるように、アラームを入れました。
- 最初は手作業で作り、あとからCloudFormationのテンプレートにしました。更新する前に変更セットを見て、テーブルが置き換わらない(データが消えない)ことを確認しています。

## 費用

どれも使った分だけ払うサービスなので、個人で使う程度ならほとんどかからない見込みです。
念のため、月5ドルの予算アラートを設定しています。

## デプロイ手順

1. CloudFormationで、`template.yaml` から新しいスタックを作ります。パラメータの `AlarmEmail` には、エラー通知を受け取るメールアドレスを入れます。IAMリソースの作成も承認します。
2. SNSから確認メールが届くので、「Confirm subscription」を押します。
3. スタックの出力にある `ApiUrl` を、`index.html` の `API_URL` に書き込みます。
4. 出力の `BucketName` のバケットに、`index.html` をアップロードします。
5. 出力の `SiteUrl` をブラウザで開きます。

削除するときは、先に `BucketName` のバケットの中身を空にしてから、スタックを削除します。

## つまずいたところ

IAMポリシーを作ったとき、リソースのARNに書いたテーブル名のハイフンとアンダースコアを間違えて、`AccessDeniedException` になりました。
直したあとは今度は `ResourceNotFoundException` が出て、テーブル名の食い違いにも気づいて直しました。
エラー文には、どの操作をどのリソースにしようとして失敗したのかが書かれているので、まずそこをよく読むのが大事だと思いました。

もうひとつ、HTMLをファイルとして直接開いたら、CORSのエラーが出ました。リクエスト元(origin)が `null` になっていたためです。
CloudFrontで公開してからは、そのURLで動作を確認しています。

## 今後やりたいこと

- 認証の追加(Cognitoなど)
- ページング(今は最新50件だけ取得しています)
- WAFやスロットリングによる悪用対策
- CI/CDでのテンプレートの検証とデプロイ

## ファイル

- `template.yaml`:CloudFormationのテンプレート
- `index.html`:フロント(HTMLとJavaScriptの1ファイル)
- `architecture.svg`:構成図
