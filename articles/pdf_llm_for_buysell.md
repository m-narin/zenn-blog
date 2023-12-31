---
title: "ChatGPT x LangChainで会社紹介PDFを基にした回答を得る"
emoji: "📚"
type: "tech"
topics:
  - ChatGPT
  - LangChain
  - OpenAI
  - Python
  - googlecolaboratory
published: true
published_at: 2024-01-04 08:00
---

# はじめに
ChatGPTは自然な言葉のようになんでも教えてくれるツールですが、あくまでネット上に大規模に置いてある一般的な知識の回答を返します。
そのため、最新の情報や独自の情報をこちらから教えて回答を返してもらうような仕組みが求められてきました。

解の一つとして、PythonのLLMライブラリであるLangChainはベクトルストアによる検索戦略機能を提供しています。
この記事では、会社紹介のPDFをChatGPTに教えることを通して、使用感を見てみたいと思います。

:::message
OpenAIのAPIを利用するため、事前にOpenAIのアカウントを作成しAPI_KEYを発行する必要があります。
:::

# colaboratory
Google Colabaoratoryを開き、下記のようにコードを入力して実際に試してみます。

## 準備
必要なライブラリをインストールします。
```shell:colab
!pip install openai chromadb langchain tiktoken pypdf
```

必要なライブラリをまとめてimportします。
```py:colab
import os
import platform

import openai
import chromadb
import langchain

from langchain.embeddings.openai import OpenAIEmbeddings
from langchain.vectorstores import Chroma
from langchain.text_splitter import CharacterTextSplitter
from langchain.chat_models import ChatOpenAI
from langchain.chains import ConversationalRetrievalChain
from langchain.document_loaders import PyPDFLoader
```

API_KEYを入力し、OpenAIに問い合わせるLLMのモデル(gpt-4)を指定します。
```py:colab
os.environ["OPENAI_API_KEY"] = '<ここにAPI_KEYを入力します>'
openai.api_key = os.getenv("OPENAI_API_KEY")

llm = ChatOpenAI(temperature=0, model_name="gpt-4")
```

PDFファイルを読み込みます。
株式会社BuySell Technologiesの会社説明資料を利用してみます。
```py:colab
loader = PyPDFLoader("https://search.sbisec.co.jp/v2/popwin/info/home/seminar/home_seminar_briefing_220405_buysell-technologies.pdf")
```

ページごとに分割します。また、分割された内容を確認します。
```py:colab
pages = loader.load_and_split()
pages[3].page_content
```

読み込んだ情報をベクトルストアに保存します。
```py:colab
embeddings = OpenAIEmbeddings()
vectorstore = Chroma.from_documents(pages, embedding=embeddings, persist_directory=".")
vectorstore.persist() 
```

文章をベクトル化(数字の配列)に置き換えることで、類似度の評価(計算)ができるようになります。
下記画像のように、OpenAIのAPIに問い合わせる前に、質問文と最も類似度の高い情報を抽出します。
その情報を一緒にOpenAIのAPIに問い合わせることで、独自情報を教えて回答を返してもらうことができるようになります。
![vector_store.jpeg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/614347/69dc3052-1082-281e-18cc-f60651947338.jpeg)

理論的な背景はこちらの記事が分かりやすいです。
https://weel.co.jp/media/vector-database

作成したベクトルストアから類似情報を抽出してQAを実行させる機能を作成します。
```py:colab
qa = ConversationalRetrievalChain.from_llm(llm, vectorstore.as_retriever(), return_source_documents=True)
```

以上で独自情報を盛り込んだQAbotの作成ができました。いくつか質問をしてみましょう。

## 質問

まずはPDFの4ページ目にあるミッションを答えてくれるのかを質問してみます。
![スクリーンショット 2023-12-31 18.16.09.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/614347/ec882f84-91d1-7023-9333-81d26fcc5e7b.png)

`qa`を使って実際に問い合わせます。
```py:colab
query = "バイセルのミッションは？"
chat_history = []

result = qa({"question": query, "chat_history": chat_history})

result["answer"]
```

筆者が試したところ、以下のように読み込ませたPDFの内容を踏まえて回答をしてくれました✨
> バイセルのミッションは、「人を超え、時を超え、たいせつなものをつなぐ架け橋となる」ことです。彼らは、モノを超えた力があると信じており、その使命を続けるために、世界中から価値あるものを掘り起こし、テクノロジーを使って人、企業、国をつなぎ、社会全体に還流させていくと述べています。また、地球が回るように循環させ、永遠に未来へたいせつなものをつないでいくとも述べています。

ちなみに、普通のChatGPT(gpt-4)に聞いてみると次のような回答になりました。
> バイセル（BuySell Technologies Co., Ltd.）は、日本のリユース業界で活動する企業です。バイセルのミッションは、公式ウェブサイトや企業情報によって定められていますが、私の知識のカットオフ日時点では、その具体的なミッションステートメントは把握していません。一般的に、リユース企業は持続可能な消費を促進し、環境への影響を減らすことを目指していることが多いですが、バイセルの正確なミッションを知りたい場合は、直接バイセルの公式ウェブサイトを参照するか、企業に問い合わせることをお勧めします。

会社自体は認識しているようですが、具体的なミッションの内容は分からないとのことです。回答内容に明確な違いが出ましたね。

他にもいくつか質問をしてみます。

```py:colab
query = "バイセルはどのような事業をやっている？"
chat_history = []

result = crc({"question": query, "chat_history": chat_history})

result["answer"]
```

> バイセルは、出張訪問買取サービスを提供しています。これは、整理や処分のニーズがあるシニア層を主なターゲットとしています。また、出張訪問買取に抵抗がある顧客を店舗へ誘導する戦略も展開しています。さらに、バイセルは店舗展開を加速させるために、グループ全体のシナジーを活用しています。\n\nバイセルはまた、リユースプラットフォーム（Cosmos）をローンチし、これをSaaS化して外部業者に提供することで新たな収益源を創出しています。このプラットフォームは、買取から販売まで一気通貫にあらゆる機能を提供し、様々なリユース業者が利用できるようになっています。

内容的にも間違いは見受けられず、非常に的を射た回答となりました。

```py:colab
query = "バイセルのテクノロジー投資について教えてください"
chat_history = []

result = crc({"question": query, "chat_history": chat_history})

result["answer"]
```

> バイセルテクノロジーズは、テクノロジー領域への投資を強化しています。これは、テクノロジーの活用による生産性向上やデータ基盤整備によるデータドリブン経営の深化を通じて、事業成長を加速させるためです。具体的な取り組みとしては、リアル（人・物）とテクノロジー（IT・DX）の融合を推進し、リユーステックカンパニーへの転換を図っています。\n\nまた、リユースプラットフォームの確立や、出張訪問時の査定時間短縮による一人あたり査定件数の増加など、業務効率の改善にも取り組んでいます。さらに、リユースとAIをテーマにした研究開発の強化や、全社データ基盤整備に伴うデータ深化によるデータドリブン経営の加速も進めています。\n\nテクノロジー組織への優先投資により、エンジニアの採用や人材育成システム、採用ブランディングを強化しています。また、商品価格予測、オークション開始価格予測、チャットボット、商品・型番判定、真贋判定、査定価格の自動算出、訪問ルート最適化、在庫アロケーション最適化など、AI技術とデータを活用した研究開発も行っています。

テクノロジー戦略に関しては、例えばPDF資料の31ページ目に書かれてますが、内容を十分に理解して回答してくれました。
![スクリーンショット 2023-12-31 18.26.39.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/614347/a1d0ea29-f197-254c-38d1-be5760f70255.png)

# おわりに
会社紹介のPDF資料を読み込ませ、それに沿った内容の回答が得られることを確認できました。
LangChainの機能を使うことで、質問に対する類似情報を検索して、その内容を基に回答を返すような仕組みを簡単に実装することができます。

長い資料を読み込んだりするのは骨が折れるので、有効活用できると非常に便利ですよね！
この機能を利用したい場面として筆者がパッと思いつくユースケースは、製品マニュアル、大学のシラバス、企業のIR資料、論文などが浮かびました。
独自情報を持ったbotのような形でアプリ化できると、とても有用だな思いました。