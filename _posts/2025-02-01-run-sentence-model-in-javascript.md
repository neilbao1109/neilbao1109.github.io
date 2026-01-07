---
title: 在JavaScript中使用句向量模型
date: 2025-02-01 11:10:00 +0800
categories: [Sentence Transformer]
tags: [javascript]
render_with_liquid: false
---

直接在浏览器或纯JavaScript环境中运行sentence transformer模型可能不太现实，因为这些模型通常很大，计算密集型，并且主要设计用于Python的深度学习框架（如PyTorch或TensorFlow）。

然而，仍然可以通过以下几种方式在JavaScript项目中使用sentence transformer的功能：

**方法一：使用Python后端API**

这是最常见且推荐的方法。你可以使用Python和sentence-transformers库创建一个API服务，然后在JavaScript前端通过网络请求（例如，使用 `fetch` API）来调用这个API。

**步骤详解：**

1. **Python 后端设置 (使用 Flask 或 FastAPI)**
你需要一个Python环境，并安装以下库：
* `sentence-transformers`: 用于句向量计算
* `Flask` 或 `FastAPI`: 用于创建API服务


如果你还没有安装这些库，可以使用pip安装：
```bash
pip install sentence-transformers flask
# 或者使用 FastAPI: pip install sentence-transformers fastapi uvicorn

```


**使用 Flask 创建 API 的示例代码 (app.py):**
```python
from flask import Flask, request, jsonify
from sentence_transformers import SentenceTransformer

app = Flask(__name__)
model = SentenceTransformer('all-mpnet-base-v2') # 选择一个 sentence transformer 模型

@app.route('/encode', methods=['POST'])
def encode_sentence():
    data = request.get_json()
    sentences = data.get('sentences', [])
    if not sentences:
        return jsonify({'error': 'No sentences provided'}), 400

    try:
        embeddings = model.encode(sentences).tolist() # 计算句向量并转换为列表方便JSON序列化
        return jsonify({'embeddings': embeddings})
    except Exception as e:
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    app.run(debug=True, port=5000) # 运行 Flask 应用，debug=True 用于开发阶段

```


**代码解释:**
* **`from sentence_transformers import SentenceTransformer`**: 导入 sentence-transformers 库。
* **`model = SentenceTransformer('all-mpnet-base-v2')`**: 加载一个预训练的 sentence transformer 模型。`'all-mpnet-base-v2'` 是一个常用的模型，你可以根据需求选择其他模型 (例如 `'paraphrase-MiniLM-L6-v2'`,  `'all-MiniLM-L6-v2'` 等)。你可以在 [Sentence Transformers 模型列表](https://www.google.com/url?sa=E&source=gmail&q=https://www.sbert.net/docs/pretrained_models.html) 中找到更多模型。
* **`@app.route('/encode', methods=['POST'])`**: 定义一个路由 `/encode`，只接受 POST 请求。
* **`request.get_json()`**: 获取POST请求中发送的JSON数据。
* **`data.get('sentences', [])`**: 从JSON数据中获取名为 `sentences` 的列表，如果不存在则默认为空列表。
* **`model.encode(sentences).tolist()`**: 使用加载的模型对句子列表进行编码，得到句向量。`.tolist()` 将 NumPy 数组转换为 Python 列表，以便能被 `jsonify` 正确序列化为 JSON。
* **`jsonify({'embeddings': embeddings})`**: 将句向量以 JSON 格式返回给客户端。
* **`app.run(debug=True, port=5000)`**: 启动 Flask 开发服务器，监听 5000 端口。 `debug=True` 模式方便开发时进行调试，但在生产环境中应关闭。


**运行 Python 后端:**
在命令行中，导航到 `app.py` 文件所在的目录，并运行：
```bash
python app.py

```


你应该看到 Flask 应用启动并运行在 `http://127.0.0.1:5000/`。
2. **JavaScript 前端调用 API**
在你的 JavaScript 代码中，你可以使用 `fetch` API 或其他HTTP客户端库（如 `axios`）来向 Python 后端发送 POST 请求，并接收句向量数据。
**JavaScript 示例代码:**
```javascript
async function getSentenceEmbeddings(sentences) {
    const apiUrl = 'http://127.0.0.1:5000/encode'; // API 后端地址
    const requestData = {
        sentences: sentences
    };

    try {
        const response = await fetch(apiUrl, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify(requestData)
        });

        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }

        const data = await response.json();
        return data.embeddings; // 返回句向量列表
    } catch (error) {
        console.error('Error fetching embeddings:', error);
        return null; // 或抛出错误，根据你的应用需求处理
    }
}

// 使用示例
async function main() {
    const sentencesToEncode = [
        "This is sentence one.",
        "Here is sentence two.",
        "This is another sentence."
    ];

    const embeddings = await getSentenceEmbeddings(sentencesToEncode);
    if (embeddings) {
        console.log("Sentence Embeddings:", embeddings);
        // 在这里你可以对句向量进行后续处理，例如相似度计算、语义搜索等
    }
}

main();

```


**代码解释:**
* **`async function getSentenceEmbeddings(sentences)`**: 定义一个异步函数，用于发送请求并获取句向量。
* **`apiUrl = 'http://127.0.0.1:5000/encode'`**:  指定 API 后端地址。确保与你 Python 后端运行的地址和端口一致。
* **`requestData = { sentences: sentences }`**:  构建请求体数据，将要编码的句子列表放在 `sentences` 字段中。
* **`fetch(apiUrl, { ... })`**: 使用 `fetch` 发送 POST 请求到 API 后端。
* `method: 'POST'`：指定请求方法为 POST。
* `headers: { 'Content-Type': 'application/json' }`:  设置请求头，告知服务器发送的是 JSON 数据。
* `body: JSON.stringify(requestData)`: 将请求数据转换为 JSON 字符串并放入请求体中。


* **`response.json()`**:  解析服务器返回的 JSON 响应。
* **`data.embeddings`**: 从 JSON 响应中提取 `embeddings` 字段，这应该就是句向量列表。
* **`main()` 函数和使用示例**:  展示如何调用 `getSentenceEmbeddings` 函数，并处理返回的句向量。



**方法二： 使用 ONNX 和 JavaScript ONNX Runtime (理论上可行，但可能复杂)**

理论上，你可以将 sentence transformer 模型转换为 ONNX 格式，然后使用 JavaScript 的 ONNX Runtime 在浏览器或 Node.js 环境中运行模型。

**步骤概述 (复杂且高级，不推荐初学者):**

1. **模型转换为 ONNX:** 使用 sentence-transformers 库或相关工具将预训练的 sentence transformer 模型导出为 ONNX 格式。这可能需要一些额外的步骤和对模型结构的理解。
2. **JavaScript ONNX Runtime:**  在 JavaScript 项目中引入 `onnxruntime-web` (用于浏览器) 或 `onnxruntime-node` (用于 Node.js)。
3. **加载 ONNX 模型并运行:** 使用 ONNX Runtime 加载转换后的 ONNX 模型，并编写 JavaScript 代码来预处理文本输入、运行模型推理，并后处理输出以得到句向量。

**重要提示:**

* **模型大小和性能:**  Sentence transformer 模型通常很大 (几百MB到几GB)，在浏览器中加载和运行大型模型可能会非常缓慢，甚至导致浏览器崩溃或页面无响应。即使在 Node.js 环境中，性能也可能不如在 GPU 加速的 Python 环境中运行。
* **复杂性:** 将深度学习模型转换为 ONNX 并在 JavaScript 中运行涉及到模型转换、JavaScript 深度学习库的使用，以及对模型输入输出的正确处理，这对于初学者来说可能非常复杂。
* **库的可用性:** JavaScript ONNX Runtime 虽然存在，但其生态系统和对复杂模型（如 transformers）的支持可能不如 Python 的深度学习框架成熟。

**为什么推荐方法一 (Python 后端 API)?**

* **成熟和稳定:** Python 的 sentence-transformers 库和 Flask/FastAPI 框架都非常成熟、稳定且广泛使用。
* **性能:**  Python 后端可以利用服务器端的计算资源（包括 GPU），提供更好的性能。
* **易于维护和部署:**  后端 API 的架构更容易维护、升级和部署。
* **模型管理:**  模型加载和管理在后端处理，前端 JavaScript 代码更简洁。
* **浏览器限制:**  浏览器环境对于运行大型深度学习模型存在天然的限制 (资源限制、性能限制)。

**总结**

虽然理论上可能存在其他方法，但**使用 Python 后端 API 是在 JavaScript 中使用 sentence transformer 最实用、最有效和最推荐的方法**。 这种方法将计算密集型任务放在服务器端处理，并通过轻量级的 API 接口与 JavaScript 前端进行交互，平衡了功能性和性能，并且易于实现和维护。如果你需要在 JavaScript 项目中使用句向量功能，强烈建议采用这种基于后端API的架构。