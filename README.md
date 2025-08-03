<li>SPA（シングルページアプリケーション）で問い合わせ管理画面を実現</li>
<li>バックエンド1：AWS Lambda + DynamoDB（Python）</li>
<li>バックエンド2：Azure Functions + Cosmos DB（Python）</li>
<li>将来、フロントエンドは同じまま、バックエンドのみ切り替え可能な構成</li>
<li>フロントエンドはReact（例として）</li>
</ul>
<hr>
<h1>2. アーキテクチャ図（Mermaid）</h1>
<pre><code class="language-mermaid">flowchart TD
    subgraph Frontend
        A[SPA (React)]
    end

    subgraph AWS
        B[API Gateway]
        C[AWS Lambda (Python)]
        D[DynamoDB]
    end

    subgraph Azure
        E[Azure Functions (Python)]
        F[Cosmos DB]
    end

    A -- REST API --&gt; B
    B -- Lambda Proxy --&gt; C
    C -- boto3 --&gt; D

    A -- REST API --&gt; E
    E -- azure-cosmos --&gt; F
</code></pre>
<hr>
<h1>3. フロントエンド例（React・問い合わせ一覧）</h1>
<pre><code class="language-jsx">// src/components/InquiryList.js
import React, { useEffect, useState } from &quot;react&quot;;
import axios from &quot;axios&quot;;

const API_URL = process.env.REACT_APP_API_URL || &quot;http://localhost:8000/inquiries&quot;;

function InquiryList() {
  const [inquiries, setInquiries] = useState([]);

  useEffect(() =&gt; {
    axios.get(API_URL)
      .then(res =&gt; setInquiries(res.data))
      .catch(err =&gt; alert(&quot;取得失敗&quot;));
  }, []);

  return (
    &lt;table&gt;
      &lt;thead&gt;
        &lt;tr&gt;
          &lt;th&gt;件名&lt;/th&gt;&lt;th&gt;説明&lt;/th&gt;&lt;th&gt;締め切り&lt;/th&gt;&lt;th&gt;優先度&lt;/th&gt;&lt;th&gt;ステータス&lt;/th&gt;
        &lt;/tr&gt;
      &lt;/thead&gt;
      &lt;tbody&gt;
        {inquiries.map(i =&gt; (
          &lt;tr key={i.id}&gt;
            &lt;td&gt;{i.title}&lt;/td&gt;
            &lt;td&gt;{i.description}&lt;/td&gt;
            &lt;td&gt;{i.deadline}&lt;/td&gt;
            &lt;td&gt;{i.priority}&lt;/td&gt;
            &lt;td&gt;{i.status}&lt;/td&gt;
          &lt;/tr&gt;
        ))}
      &lt;/tbody&gt;
    &lt;/table&gt;
  )
}
export default InquiryList;
</code></pre>
<hr>
<h1>4. バックエンド例</h1>
<h2>(1) AWS Lambda + DynamoDB（Python）</h2>
<p><strong>Lambda関数例（API Gateway REST Proxy対応）</strong></p>
<pre><code class="language-python">import boto3
import json
from decimal import Decimal

dynamodb = boto3.resource(&#39;dynamodb&#39;)
table = dynamodb.Table(&#39;Inquiries&#39;)

def lambda_handler(event, context):
    if event[&#39;httpMethod&#39;] == &#39;GET&#39;:
        # 一覧取得
        items = table.scan()[&#39;Items&#39;]
        # DynamoDBのDecimal型をfloat/strに変換
        for item in items:
            for k, v in item.items():
                if isinstance(v, Decimal):
                    item[k] = float(v)
        return {
            &#39;statusCode&#39;: 200,
            &#39;headers&#39;: {&#39;Content-Type&#39;: &#39;application/json&#39;},
            &#39;body&#39;: json.dumps(items)
        }
    elif event[&#39;httpMethod&#39;] == &#39;POST&#39;:
        data = json.loads(event[&#39;body&#39;])
        table.put_item(Item=data)
        return {&#39;statusCode&#39;: 201, &#39;body&#39;: &#39;Created&#39;}
    else:
        return {&#39;statusCode&#39;: 405, &#39;body&#39;: &#39;Method Not Allowed&#39;}
</code></pre>
<p><strong>DynamoDBテーブル例</strong></p>
<ul>
<li>テーブル名: <code>Inquiries</code></li>
<li>パーティションキー: <code>id</code> (string)</li>
</ul>
<hr>
<h2>(2) Azure Functions + CosmosDB（Python）</h2>
<p><strong>function_app/inquiry_api/<strong>init</strong>.py</strong></p>
<pre><code class="language-python">import azure.functions as func
import json
from azure.cosmos import CosmosClient
import os

endpoint = os.environ[&#39;COSMOS_ENDPOINT&#39;]
key = os.environ[&#39;COSMOS_KEY&#39;]
client = CosmosClient(endpoint, key)
db = client.get_database_client(&#39;InquiryDB&#39;)
container = db.get_container_client(&#39;Inquiries&#39;)

def main(req: func.HttpRequest) -&gt; func.HttpResponse:
    if req.method == &quot;GET&quot;:
        items = list(container.read_all_items())
        return func.HttpResponse(json.dumps(items, default=str), mimetype=&quot;application/json&quot;)
    elif req.method == &quot;POST&quot;:
        data = req.get_json()
        container.create_item(body=data)
        return func.HttpResponse(status_code=201)
    else:
        return func.HttpResponse(status_code=405)
</code></pre>
<p><strong>Cosmos DB</strong></p>
<ul>
<li>データベース名: <code>InquiryDB</code></li>
<li>コンテナ名: <code>Inquiries</code></li>
<li>パーティションキー: <code>/id</code></li>
</ul>
<hr>
<h1>5. データモデル例（共通）</h1>
<pre><code class="language-json">{
  &quot;id&quot;: &quot;uuid&quot;,
  &quot;title&quot;: &quot;Pythonの習得&quot;,
  &quot;description&quot;: &quot;講座終了&quot;,
  &quot;deadline&quot;: &quot;2025/08/06&quot;,
  &quot;priority&quot;: &quot;高&quot;,
  &quot;status&quot;: &quot;着手中&quot;
}
</code></pre>
<hr>
<h1>6. デプロイ方法の概要</h1>
<ul>
<li>フロントエンドは <code>.env</code> でAPIのエンドポイントを切り替えられるようにしておく</li>
<li>Lambda用、Function用でAPI URLを変えるだけで同じSPAが利用可能</li>
</ul>
<hr>
<h1>7. まとめ</h1>
<ul>
<li><strong>SPA</strong>（Reactなど） + **API（Lambda/Dynamo or Function/Cosmos）**の構成</li>
<li>Pythonでどちらのバックエンドも実装可能</li>
<li>フロントエンドはAPIエンドポイントを切り替えるだけ</li>
<li>サンプルコード、アーキテクチャ図付き</li>
</ul>
<hr>
<p>必要に応じて、<strong>実際のリポジトリ構成やデプロイ設定例、追加機能</strong>もご案内可能です。<br>さらに詳しい実装や個別ファイルの雛形が必要ならご要望ください！</p>
