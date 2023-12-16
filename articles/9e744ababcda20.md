---
title: "HuggingFaceEmbeddingsのモデルがコンテナ実行時にダウンロードされるのを防ぐ"
emoji: "🤗"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [huggingface, python, docker, langchain]
published: true
---

# はじめに

タイトルの通りだけれど、HuggingFaceEmbeddings のモデルがコンテナ実行時にダウンロードされるのを防ぐ方法を考えた。

# HuggingFaceEmbeddings を使ったベクターストア

詳しくは説明しないけど、LangChain とかで RAG (Retrieval-Augmented Generation)するとき、API を使わずに Embedding しようとするとこんな感じで書ける。

```python
client = chromadb.PersistentClient(path=directory)
embedding = HuggingFaceEmbeddings(model_name='intfloat/multilingual-e5-large')
return Chroma(
    collection_name=collection_name,
    embedding_function=embedding,
    client=client,
    )
```

これを、Docker コンテナで動かしてたけど、ビルド時じゃなくてコンテナ起動時にダウンロードが走っていて、それにかなりの時間がかかっていた。
なので、理由を調べて解決策を考えてみた。

# モデルをダウンロードするタイミング

クラスの中身をみるとこんな感じで、コンストラクタの中で import してる部分があった。

```python
class HuggingFaceEmbeddings(BaseModel, Embeddings):
    """省略"""
    def __init__(self, **kwargs: Any):
        """Initialize the sentence_transformer."""
        super().__init__(**kwargs)
        try:
            import sentence_transformers

        except ImportError as exc:
            raise ImportError(
                "Could not import sentence_transformers python package. "
                "Please install it with `pip install sentence-transformers`."
            ) from exc

        self.client = sentence_transformers.SentenceTransformer(
            self.model_name, cache_folder=self.cache_folder, **self.model_kwargs
        )
```

ここの記載によると、`$HOME/.cache/torch/sentence_transformers`にダウンロードされて使われるみたい。実際、自分の環境（Mac）ではダウンロードされてた。

https://github.com/UKPLab/sentence-transformers/issues/1869

それをコンテナビルド時にコピーしたら良いかもしれないが、環境によって使うモデルも変わるかもしれない（よくわかってないけど量子化したり GPU 使ったり変わりそうな気がしてる）。

# 対策

ビルド時に簡単なスクリプトを実行しちゃえば良いのかと気づいて、以下のようなクラス呼び出すだけのコードをコンテナビルド時に動かすことにした。

```python:download_embedding_model.py
from langchain.embeddings import HuggingFaceEmbeddings

embedding = HuggingFaceEmbeddings(model_name='intfloat/multilingual-e5-large')
print(embedding)
```

こんな感じで、ビルド時に実行することで、コンテナ起動時に実行されるのを防いだ。

```Dockerfile
FROM python:3.11.6-slim-bookworm

WORKDIR /workspace
COPY requirements.txt *.py /workspace/
RUN pip3 install --upgrade pip && pip3 install -r /workspace/requirements.txt
RUN python3 download_embedding_model.py

EXPOSE 8080
ENTRYPOINT ["python3"]
CMD ["main.py"]
```

実際、試してみると期待通りダウンロードのタイミングを制御することができた。

# おわりに

地味に困っていたことを解決できてスッキリした。
