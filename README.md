# LlamaEdge-RAG API Server

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [LlamaEdge-RAG API Server](#llamaedge-rag-api-server)
  - [Introduction](#introduction)
    - [Endpoints](#endpoints)
      - [`/v1/models` endpoint](#v1models-endpoint)
      - [`/v1/chat/completions` endpoint](#v1chatcompletions-endpoint)
      - [`/v1/files` endpoint](#v1files-endpoint)
      - [`/v1/chunks` endpoint](#v1chunks-endpoint)
      - [`/v1/embeddings` endpoint](#v1embeddings-endpoint)
      - [`/v1/create/rag` endpoint](#v1createrag-endpoint)
  - [Setup](#setup)
  - [Build](#build)
  - [Execute](#execute)

<!-- /code_chunk_output -->

## Introduction

LlamaEdge-RAG API server provides a group of OpenAI-compatible web APIs for the Retrieval-Augmented Generation (RAG) applications. The server is implemented in WebAssembly (Wasm) and runs on [WasmEdge Runtime](https://github.com/WasmEdge/WasmEdge).

### Endpoints

#### `/v1/models` endpoint

`rag-api-server` provides a POST API `/v1/models` to list currently available models.

<details> <summary> Example </summary>

You can use `curl` to test it on a new terminal:

```bash
curl -X POST http://localhost:8080/v1/models -H 'accept:application/json'
```

If the command runs successfully, you should see the similar output as below in your terminal:

```json
{
    "object":"list",
    "data":[
        {
            "id":"llama-2-chat",
            "created":1697084821,
            "object":"model",
            "owned_by":"Not specified"
        }
    ]
}
```

</details>

#### `/v1/chat/completions` endpoint

Ask a question using OpenAI's JSON message format.

<details> <summary> Example </summary>

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
-H 'accept:application/json' \
-H 'Content-Type: application/json' \
-d '{"messages":[{"role":"system", "content": "You are a helpful assistant."}, {"role":"user", "content": "Who is Robert Oppenheimer?"}], "model":"llama-2-chat"}'
```

Here is the response.

```json
{
    "id":"",
    "object":"chat.completion",
    "created":1697092593,
    "model":"llama-2-chat",
    "choices":[
        {
            "index":0,
            "message":{
                "role":"assistant",
                "content":"Robert Oppenheimer was an American theoretical physicist and director of the Manhattan Project, which developed the atomic bomb during World War II. He is widely regarded as one of the most important physicists of the 20th century and is known for his contributions to the development of quantum mechanics and the theory of the atomic nucleus. Oppenheimer was also a prominent figure in the post-war nuclear weapons debate, advocating for international control and regulation of nuclear weapons."
            },
            "finish_reason":"stop"
        }
    ],
    "usage":{
        "prompt_tokens":9,
        "completion_tokens":12,
        "total_tokens":21
    }
}
```

</details>

#### `/v1/files` endpoint

In RAG applications, uploading files is a necessary step.

<details> <summary> Example </summary>

The following command upload a text file [paris.txt](https://huggingface.co/datasets/gaianet/paris/raw/main/paris.txt) to the API server via the `/v1/files` endpoint:

```bash
curl -X POST http://127.0.0.1:8080/v1/files -F "file=@paris.txt"
```

If the command is successful, you should see the similar output as below in your terminal:

```json
{
    "id": "file_4bc24593-2a57-4646-af16-028855e7802e",
    "bytes": 2161,
    "created_at": 1711611801,
    "filename": "paris.txt",
    "object": "file",
    "purpose": "assistants"
}
```

The `id` and `filename` fields are important for the next step, for example, to segment the uploaded file to chunks for computing embeddings.

</details>

#### `/v1/chunks` endpoint

To segment the uploaded file to chunks for computing embeddings, use the `/v1/chunks` API.

<details> <summary> Example </summary>

The following command sends the uploaded file ID and filename to the API server and gets the chunks:

```bash
curl -X POST http://localhost:8080/v1/chunks \
    -H 'accept:application/json' \
    -H 'Content-Type: application/json' \
    -d '{"id":"file_4bc24593-2a57-4646-af16-028855e7802e", "filename":"paris.txt"}'
```

The following is an example return with the generated chunks:

```json
{
    "id": "file_4bc24593-2a57-4646-af16-028855e7802e",
    "filename": "paris.txt",
    "chunks": [
        "Paris, city and capital of France, ..., for Paris has retained its importance as a centre for education and intellectual pursuits.",
        "Paris’s site at a crossroads ..., drawing to itself much of the talent and vitality of the provinces."
    ]
}
```

</details>

#### `/v1/embeddings` endpoint

To compute embeddings for user query or file chunks, use the `/v1/embeddings` API.

<details> <summary> Example </summary>

The following command sends a query to the API server and gets the embeddings as return:

```bash
curl -X POST http://localhost:8080/v1/embeddings \
    -H 'accept:application/json' \
    -H 'Content-Type: application/json' \
    -d '{"model": "e5-mistral-7b-instruct-Q5_K_M", "input":["Paris, city and capital of France, ..., for Paris has retained its importance as a centre for education and intellectual pursuits.", "Paris’s site at a crossroads ..., drawing to itself much of the talent and vitality of the provinces."]}'
```

The embeddings returned are like below:

```json
{
    "object": "list",
    "data": [
        {
            "index": 0,
            "object": "embedding",
            "embedding": [
                0.1428378969,
                -0.0447309874,
                0.007660218049,
                ...
                -0.0128974719,
                -0.03543198109,
                0.03974733502,
                0.00946635101,
                -0.01531364303
            ]
        },
        {
            "index": 1,
            "object": "embedding",
            "embedding": [
                0.0697753951,
                -0.0001159032545,
                0.02073983476,
                ...
                0.03565846011,
                -0.04550019652,
                0.02691745944,
                0.02498772368,
                -0.003226313973
            ]
        }
    ],
    "model": "e5-mistral-7b-instruct-Q5_K_M",
    "usage": {
        "prompt_tokens": 491,
        "completion_tokens": 0,
        "total_tokens": 491
    }
}
```

</details>

#### `/v1/create/rag` endpoint

`/v1/create/rag` endpoint provides users a one-click way to convert a text or markdown file to embeddings directly. The effect of the endpoint is equivalent to running `/v1/files` + `/v1/chunks` + `/v1/embeddings` sequently.

<details> <summary> Example </summary>

The following command uploads a text file [paris.txt](https://huggingface.co/datasets/gaianet/paris/raw/main/paris.txt) to the API server via the `/v1/create/rag` endpoint:

```bash
curl -X POST http://127.0.0.1:8080/v1/create/rag -F "file=@paris.txt"
```

The embeddings returned are like below:

```json
{
    "object": "list",
    "data": [
        {
            "index": 0,
            "object": "embedding",
            "embedding": [
                0.1428378969,
                -0.0447309874,
                0.007660218049,
                ...
                -0.0128974719,
                -0.03543198109,
                0.03974733502,
                0.00946635101,
                -0.01531364303
            ]
        },
        {
            "index": 1,
            "object": "embedding",
            "embedding": [
                0.0697753951,
                -0.0001159032545,
                0.02073983476,
                ...
                0.03565846011,
                -0.04550019652,
                0.02691745944,
                0.02498772368,
                -0.003226313973
            ]
        }
    ],
    "model": "e5-mistral-7b-instruct-Q5_K_M",
    "usage": {
        "prompt_tokens": 491,
        "completion_tokens": 0,
        "total_tokens": 491
    }
}
```

</details>

## Setup

Llama-RAG API server runs on WasmEdge Runtime. According to the operating system you are using, choose the installation command:

<details> <summary> For macOS (apple silicon) </summary>

```console
# install WasmEdge-0.13.4 with wasi-nn-ggml plugin
curl -sSf https://raw.githubusercontent.com/WasmEdge/WasmEdge/master/utils/install.sh | bash -s -- --plugin wasi_nn-ggml

# Assuming you use zsh (the default shell on macOS), run the following command to activate the environment
source $HOME/.zshenv
```

</details>

<details> <summary> For Ubuntu (>= 20.04) </summary>

```console
# install libopenblas-dev
apt update && apt install -y libopenblas-dev

# install WasmEdge-0.13.4 with wasi-nn-ggml plugin
curl -sSf https://raw.githubusercontent.com/WasmEdge/WasmEdge/master/utils/install.sh | bash -s -- --plugin wasi_nn-ggml

# Assuming you use bash (the default shell on Ubuntu), run the following command to activate the environment
source $HOME/.bashrc
```

</details>

<details> <summary> For General Linux </summary>

```console
# install WasmEdge-0.13.4 with wasi-nn-ggml plugin
curl -sSf https://raw.githubusercontent.com/WasmEdge/WasmEdge/master/utils/install.sh | bash -s -- --plugin wasi_nn-ggml

# Assuming you use bash (the default shell on Ubuntu), run the following command to activate the environment
source $HOME/.bashrc
```

</details>

## Build

```bash
# Clone the repository
git clone https://github.com/LlamaEdge/rag-api-server.git

# Change the working directory
cd rag-api-server

# Build `rag-api-server.wasm` with the `http` support only, or
cargo build --target wasm32-wasi --release

# Build `rag-api-server.wasm` with both `http` and `https` support
cargo build --target wasm32-wasi --release --features full

# Copy the `rag-api-server.wasm` to the root directory
cp target/wasm32-wasi/release/rag-api-server.wasm .
```

<details> <summary> To check the CLI options, </summary>

To check the CLI options of the `rag-api-server` wasm app, you can run the following command:

  ```bash
  $ wasmedge rag-api-server.wasm -h

  Usage: rag-api-server.wasm [OPTIONS] --model-name <MODEL_NAME> --prompt-template <PROMPT_TEMPLATE>

  Options:
    -m, --model-name <MODEL_NAME>
            Sets names for chat and embedding models. The names are separated by comma without space, for example, '--model-name Llama-2-7b,all-minilm'
    -a, --model-alias <MODEL_ALIAS>
            Model aliases for chat and embedding models [default: default,embedding]
    -c, --ctx-size <CTX_SIZE>
            Sets context sizes for chat and embedding models. The sizes are separated by comma without space, for example, '--ctx-size 4096,384'. The first value is for the chat model, and the second is for the embedding model [default: 4096,384]
    -p, --prompt-template <PROMPT_TEMPLATE>
            Prompt template [possible values: llama-2-chat, mistral-instruct, mistrallite, openchat, codellama-instruct, codellama-super-instruct, human-assistant, vicuna-1.0-chat, vicuna-1.1-chat, vicuna-llava, chatml, baichuan-2, wizard-coder, zephyr, stablelm-zephyr, intel-neural, deepseek-chat, deepseek-coder, solar-instruct, phi-2-chat, phi-2-instruct, gemma-instruct]
    -r, --reverse-prompt <REVERSE_PROMPT>
            Halt generation at PROMPT, return control
    -b, --batch-size <BATCH_SIZE>
            Batch size for prompt processing [default: 512]
        --rag-prompt <RAG_PROMPT>
            Custom rag prompt
        --qdrant-url <QDRANT_URL>
            URL of Qdrant REST Service [default: http://localhost:6333]
        --qdrant-collection-name <QDRANT_COLLECTION_NAME>
            Name of Qdrant collection [default: default]
        --qdrant-limit <QDRANT_LIMIT>
            Max number of retrieved result [default: 3]
        --qdrant-score-threshold <QDRANT_SCORE_THRESHOLD>
            Minimal score threshold for the search result [default: 0.4]
        --log-prompts
            Print prompt strings to stdout
        --log-stat
            Print statistics to stdout
        --log-all
            Print all log information to stdout
        --socket-addr <SOCKET_ADDR>
            Socket address of LlamaEdge API Server instance [default: 0.0.0.0:8080]
        --web-ui <WEB_UI>
            Root path for the Web UI files [default: chatbot-ui]
    -h, --help
            Print help
    -V, --version
            Print version
  ```

</details>

## Execute

LlamaEdge-RAG API server requires two types of models: chat and embedding. The chat model is used for generating responses to user queries, while the embedding model is used for computing embeddings for user queries or file chunks.

For the purpose of demonstration, we use the [Llama-2-7b-chat-hf-Q5_K_M.gguf](https://huggingface.co/second-state/Llama-2-7B-Chat-GGUF/resolve/main/Llama-2-7b-chat-hf-Q5_K_M.gguf) and [all-MiniLM-L6-v2-ggml-model-f16.gguf](https://huggingface.co/second-state/All-MiniLM-L6-v2-Embedding-GGUF/resolve/main/all-MiniLM-L6-v2-ggml-model-f16.gguf) models as examples.

- Start an instance of LlamaEdge-RAG API server

  ```bash
  # Assume that the `rag-api-server.wasm` and the model files are in the root directory of the repository
  wasmedge --dir .:. --nn-preload default:GGML:AUTO:Llama-2-7b-chat-hf-Q5_K_M.gguf \
      --nn-preload embedding:GGML:AUTO:all-MiniLM-L6-v2-ggml-model-f16.gguf \
      rag-api-server.wasm \
      --model-name Llama-2-7b-chat-hf-Q5_K_M,all-MiniLM-L6-v2-ggml-model-f16 \
      --ctx-size 4096,384 \
      --prompt-template llama-2-chat \
      --system-prompt "Use the following pieces of context to answer the user's question.\nIf you don't know the answer, just say that you don't know, don't try to make up an answer.\n----------------\n" \
      --log-prompts \
      --log-stat
  ```
