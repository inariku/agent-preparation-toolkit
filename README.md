# Agent Preparation Toolkit (APT)

## これは？

生成 AI における Agent をすぐに体感できるサンプル実装です。
Amazon Bedrock Agents を使ってすぐに Agent を動かすことができるほか、使用している Knowledge Bases のデータや Lambda 関数を差し替えたり付け加えたりすることで自社用の Agent に改造できます。

## 使い方

> [!NOTE]  
> AWS のリージョンは `us-west-2` で動作確認してあり、`bin/agents-preparation-toolkit.ts` にて region をハードコーディングしています。  
> 他リージョンにデプロイする場合は、エージェントのモデルの対応を見てから設定を変えて(e.g. ハードコーディングを消すなど)からデプロイしてください。

> [!NOTE]
> node.js 及び python, boto3 がインストールされている必要があります。
> 環境に応じてインストールしてください。

> [!TIP]
> 後述の内包している Agents は`parameter.ts` の各 Agent の設定の enabled の部分を `true` もしくは `false` にすることで有効無効の設定をした上でデプロイできます。  
> デフォルトでは python-coder だけ有効化されています。
> bedrock-logs-watcher は後述する別途の設定が必要なことに注意してください。  

```typescript
  pythonCoder: {
    enabled: true,
  },
  hrAgent: {
    enabled: false,
  },
  productSupportAgent: {
    enabled: false,
  },
  contractSearcher: {
    enabled: false,
  },
  bedrockLogWatcher: {
    enabled: false,
    config: {
      bedrockLogsBucket: '',
      bedrockLogsPrefix: '',
    },
  },
```

```shell
# リポジトリの Clone
git clone https://github.com/aws-samples/agent-preparation-toolkit

# カレントディレクトリをリポジトリに移す
cd agent-preparation-toolkit

# Agent で利用する Lambda のパッケージダウンロード
npm install && cd custom-resources && npm ci && cd ..
cd ./action-groups/python-coder/lambda && pip install -r requirements.txt -t lib/ && cd ../../../
cd ./action-groups/bedrock-logs-watcher/lambda && pip install -r requirements.txt -t lib/ && cd ../../../

# CDK Bootstrap 
cdk bootstrap

# Agent のデプロイ
npm run cdk:deploy

# (待つ)
# CDK の出力にあるStackName = の後ろの値をコピーする
# 例: Dev-AgentPreparationToolkitStack.StackName = dev-AgentPreparationToolkitStack

# DataSource の同期 
# {YOUR_STACK_NAME} には dev-AgentPreparationToolkitStack などを入力
python 1_sync.py -s {YOUR_STACK_NAME} -r us-west-2 # DataSource の同期が走る。region を変えた場合は region 名を修正する

# Agent 呼び出しサンプル
python 2_invoke.py -r us-west-2 # region を変えた場合は region 名を修正する。詳細のトレースがほしい場合は --raw オプションを入れる
```

## 内包する Agents

### Python Coder

ユーザーは Python Coder にコーディングして欲しい内容を与えると、Python Coder は自分でコードを書き、自動でテストし、コードとテスト結果を返します。  
試しに `３次元ベクトルの外積を計算するコードを書いて` などと依頼するとその通りのコード及びテストコードと結果を返してくれます。  
![python-coder-sample](./image/python-coder-sample.png)  

コードは単一ファイルで実行できる前提で、リポジトリ丸ごと作成する処理はできません。
![python-coder-architecture](./image/python-coder.png)

カスタマイズする場合のメインのカスタマイズ箇所は以下です。

* [AWS リソース定義](./lib/agents-preparation-toolkit-stack.ts) の `const BASE_AGENT_NAME:string = 'python-coder';` 以下
* Action Group
  * [Lambda 関数](./action-groups/python-coder/lambda/index.py)
  * [OpenAPI スキーマ](./action-groups/python-coder/schema/api-schema.yaml)
* プロンプト
  * [デフォルトプロンプト](./lib/prompts/default-prompts.ts)
  * [Python Coder 用プロンプト](./lib/prompts/custom-prompts.ts)

### Human Resource Agent

Knowledge Base に会社の年休付与規則と Database (Lambda 内で動く SQLite) に社員の入社日が格納されています。  
各社員の今年の年休付与日数を問い合わせることができます。
試しに `Kazuhito Go の今年度の年休付与日数は？` と問い合わせると、本日の日付を取得して Kazuhito Go の入社日から在籍日数を計算し、Knowledge Base から年休付与日数を算出します。
![human-resource-sample](./image/human-resource-sample.png)
![human-resource-agent-architecture](./image/human-resource-agent.png)

カスタマイズする場合のメインのカスタマイズ箇所は以下です。

* [AWS リソース定義](./lib/agents-preparation-toolkit-stack.ts) の `const BASE_HR_AGENT_NAME:string = 'human-resource-agent';` 以下
* Action Group
  * [Lambda 関数](./action-groups/hr/lambda/index.py)
  * [OpenAPI スキーマ](./action-groups/hr/schema/api-schema.yaml)
* [Knowledge Base のデータソース](./data-source/hr/)
* プロンプト
  * [デフォルトプロンプト](./lib/prompts/default-prompts.ts)
  * [Human Resource Agent 用プロンプト](./lib/prompts/custom-prompts.ts)

### Product Support Agent

プリンタのエラーコードを持つ Knowledge Base と、Database (Lambda 内で動く SQLite) にエラーコードごとの対応履歴が格納されています。  
エラーコードを与えるとどんなことをすれば直る可能性があるかを教えてくれます。
試しに `E-03` と検索すると、過去の E-03 の詳細及び過去の対応から何をすればいいかを出力します。
![product-support-sample](./image/product-support-sample.png)
![human-resource-agent-architecture](./image/product-support-agent.png)

カスタマイズする場合のメインのカスタマイズ箇所は以下です。

* [AWS リソース定義](./lib/agents-preparation-toolkit-stack.ts) の `const BASE_PS_AGENT_NAME:string = 'product-support-agent';` 以下
* Action Group
  * [Lambda 関数](./action-groups/product-support/lambda/index.py)
  * [OpenAPI スキーマ](./action-groups/product-support/schema/api-schema.yaml)
* [Knowledge Base のデータソース](./data-source/product-support/)
* プロンプト
  * [デフォルトプロンプト](./lib/prompts/default-prompts.ts)
  * [Human Resource Agent 用プロンプト](./lib/prompts/custom-prompts.ts)

### Contract Searcher

契約書を探す Agent です。  
やりたいことを入力すると結ぶべき契約書を教えてテンプレートを出してくれます。  
試しに `人に仕事を任せたい` と入力すると、どんな契約書が必要かを教えてくれます。  
そこから業務委託契約書に誘導すると、業務委託契約書のテンプレートを出してくれます。  
サンプルのデータには業務委託契約書が 2 つ格納されていますが、より新しいものを正としてテンプレートダウンロード URL を出力します。
![contract-searcher-sample](./image/contract-searcher-sample.png)
![contract-searcher-architecture](./image/contract-searcher.png)

### Bedrock Logs Watcher

Amazon Bedrock では[モデルの呼び出しログを S3 に保存することができます](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html)。  

Log を Glue のテーブルに取り込んだ上、Agent が SQL を発行して Lambda から Athena 経由でクエリを実行できます。  
例えば、`input token, output token の上位 2 identity について、年月ごとに input token, output token を identity ごとに集計して折れ線グラフにして` という自然言語のプロンプトを入力すれば、今まで Input/Output Tokens の使用量 Top 2 の IAM ユーザー/ロールの特定と tokens の推移を可視化できます。  
裏側ではユーザーのプロンプトから LLM が SQL を発行/実行し、返された csv データに対して、LLM が作成した python のコードを実行して図にして返してくれます。

Bedrock Logs Watcher を有効化するには事前の設定が必要で、[parameter.ts](./parameter.ts) の以下部分 2 箇所を設定をした後、再度デプロイコマンドを順に実行してください。

```typescript
export const BEDROCK_LOGS_CONFIG: BedrockLogsConfig = {
  bedrockLogsBucket: "", // Bedrock のログを保存しているバケット。ログの有効化をしていない場合はマネジメントコンソールから S3 にログを出力するように設定する必要があります。
  bedrockLogsPrefix: "", //デフォルトだとこちら → /AWSLogs/{ACCOUNT}/BedrockModelInvocationLogs/{REGION}/
};
```

![bedrock-logs-watcher-sample](./image/bedrock-logs-watcher-sample.png)
![bedrock-logs-watcher](./image/bedrock-logs-watcher.png)

> [!IMPORTANT]
> Human Resource Agent 及び Product Support Agent, Bedrock Logs Watcher には Amazon Bedrock Agents で LLM が SQL を考えて Action Group に登録されている AWS Lambda の Lambda 関数が SQL を実行する仕組みが入っています。  
> 本サンプルでは Lambda 関数上に立てている SQLite の DB に対してクエリを投げており、Lambda 関数上で INSERT や DROP の命令を除外する仕組みが入っています。  
> 実際には RDS や Athena などの DB に対してクエリを投げるはずですが、そのときは Lambda のロールや、DB のユーザーに対して、SELECT (READ) 系の実行しかできないよう権限の制御をかけてください。

## Generative AI Use Cases (通称: GenU) 連携

このリポジトリに GUI は無いですが、[AWS マネジメントコンソールの GUI](https://us-west-2.console.aws.amazon.com/bedrock/home?region=us-west-2#/agents) を使うことができる他、[GenU](https://github.com/aws-samples/generative-ai-use-cases) を利用することで、簡単に GUI を作成できます。  
(上記のスクリーンショットは GenU を用いたものです)  
`1_sync.py` を実行したあと、`genu.txt` というテキストファイルが出来上がります。  
GenU の `./packages/cdk/parameter.ts` もしくは `./packages/cdk/cdk.json` の agents パラメータの配列の中に json ファイルの中身を格納してください。  
`./packages/cdk/cdk.json` を使用する場合は key のダブルクオーテーションを追加する必要がある点に注意してください。  
詳細は [手動で作成した Agent を追加](https://github.com/aws-samples/generative-ai-use-cases/blob/main/docs/ja/DEPLOY_OPTION.md#%E6%89%8B%E5%8B%95%E3%81%A7%E4%BD%9C%E6%88%90%E3%81%97%E3%81%9F-agent-%E3%82%92%E8%BF%BD%E5%8A%A0) を参照ください。

## Bedrock Engineer 連携

開発に利用できる AI エージェント Bedrock Engineer を GUI とすることもできます。  
[ツールの設定](https://github.com/aws-samples/bedrock-engineer?tab=readme-ov-file#select-tools--customize-tools) から APT で作成した Bedrock Agent の agent id と alias id を指定してください。

## 削除

以下コマンドで削除してください。  

```shell
npm run cdk:destroy
```
