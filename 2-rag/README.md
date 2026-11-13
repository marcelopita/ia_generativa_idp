# Laboratório: Retrieval-Augmented Generation (RAG)

Este laboratório tem como objetivo **construir um ambiente local de RAG
(Retrieval-Augmented Generation)** utilizando ferramentas do
ecossistema de IA generativa.

Ao final do laboratório, os alunos terão um sistema funcional capaz de:

-   Indexar documentos
-   Recuperar trechos relevantes
-   Utilizar modelos de linguagem para responder perguntas com base
    nesses documentos

A stack utilizada inclui:

-   Ollama -- gerenciamento de modelos locais (LLMs e embeddings)
-   Open WebUI -- interface gráfica para interação com LLMs
-   ChromaDB -- banco de dados vetorial
-   Docling -- extração de conteúdo de documentos
-   OpenRouter -- modelos de visão (VLM) usados pelo Docling

------------------------------------------------------------------------

# Estrutura da Aula

## 1. Apresentação Teórica

A aula começa com uma apresentação conceitual sobre **RAG
(Retrieval-Augmented Generation)** abordando:

-   Limitações de LLMs sem contexto
-   Conceito de recuperação semântica
-   Embeddings vetoriais
-   Bancos vetoriais
-   Pipeline RAG
-   Estratégias de re-ranking

------------------------------------------------------------------------

# 2. Laboratório

O laboratório consiste em montar **uma stack completa de RAG rodando
localmente**.

Arquitetura:

    Usuário
       │
       ▼
    Open WebUI
       │
       ├── Ollama (LLM + Embeddings + Reranker)
       │
       ├── ChromaDB (Vector Database)
       │
       └── Docling (Extração de conteúdo)

------------------------------------------------------------------------

# 2.1 Pré-requisitos

Instale previamente, conforme o seu sistema operacional:

| Ferramenta | Windows | macOS | Linux |
|---|---|---|---|
| Git | [git-scm.com](https://git-scm.com/download/win) ou `winget install Git.Git` | `brew install git` (ou Xcode Command Line Tools) | `sudo apt install git` (Debian/Ubuntu) ou `sudo dnf install git` (Fedora) |
| Docker | [Docker Desktop](https://docs.docker.com/desktop/setup/install/windows-install/) com backend **WSL 2** | [Docker Desktop](https://docs.docker.com/desktop/setup/install/mac-install/) | [Docker Engine](https://docs.docker.com/engine/install/) + plugin Compose |
| VSCode | [code.visualstudio.com](https://code.visualstudio.com/) | [code.visualstudio.com](https://code.visualstudio.com/) | [code.visualstudio.com](https://code.visualstudio.com/) |

Observações:

-   **Windows:** abra o Docker Desktop e aguarde ele indicar que está em
    execução antes de continuar. Use o **PowerShell** como terminal
    (no VSCode: `Terminal → Novo Terminal`).
-   **macOS:** abra o Docker Desktop ao menos uma vez para concluir a
    instalação.
-   **Linux:** para usar o Docker sem `sudo`, adicione seu usuário ao
    grupo `docker` e reinicie a sessão:

    ``` bash
    sudo usermod -aG docker $USER
    ```

Confirme que tudo está instalado:

``` bash
git --version
docker --version
docker compose version
```

------------------------------------------------------------------------

# 2.2 Clonando o Repositório

Clone o repositório da disciplina (o comando é o mesmo nos três sistemas):

``` bash
git clone https://github.com/marcelopita/ia_generativa_idp.git
```

O diretório clonado será a **raiz dos laboratórios da disciplina**.

Neste laboratório utilizaremos o diretório:

    2-rag

------------------------------------------------------------------------

# 2.3 Instalando Ollama (Bare Metal)

O **Ollama será instalado diretamente no host** (fora do Docker),
permitindo uso completo da GPU.

Isso é especialmente importante para:

-   GPUs NVIDIA
-   Chips Apple Silicon, que não suportam GPU em containers Docker.

## Instalação

### Windows

Opção 1 -- instalador gráfico: baixe o `OllamaSetup.exe` em
<https://ollama.com/download/windows> e execute.

Opção 2 -- pelo PowerShell:

``` powershell
winget install Ollama.Ollama
```

Após a instalação, o Ollama roda em segundo plano (ícone na bandeja do
sistema) e já expõe o servidor em `http://localhost:11434`.

### macOS

Opção 1 -- aplicativo: baixe em <https://ollama.com/download/mac>,
arraste para `Aplicativos` e abra.

Opção 2 -- Homebrew:

``` bash
brew install --cask ollama
```

Abra o aplicativo Ollama (ícone na barra de menus). Ele já inicia o
servidor em `http://localhost:11434`.

### Linux

``` bash
curl -fsSL https://ollama.com/install.sh | sh
```

O script instala o Ollama como serviço `systemd`, que já inicia
automaticamente. Verifique com:

``` bash
systemctl status ollama
```

## Baixando os modelos

Em qualquer sistema (PowerShell, Terminal ou shell Linux):

``` bash
ollama pull qwen3-embedding:0.6b
ollama pull llama3.1:8b
```

### Modelos utilizados

| Modelo                | Tipo            | Função no RAG                              | Parâmetros | Dimensão |
|----------------------|-----------------|---------------------------------------------|-----------|----------|
| `qwen3-embedding:0.6b` | Embedding Model | Converte textos em vetores para busca semântica | 600M      | 1024     |
| `llama3.1:8b`         | Language Model  | Gera respostas a partir do contexto recuperado | 8B        | —        |

## Variáveis importantes do Ollama

| Variável                   | Valor | Função                                      |
|----------------------------|-------|---------------------------------------------|
| `OLLAMA_NUM_PARALLEL`      | `4`   | Requisições simultâneas por modelo          |
| `OLLAMA_MAX_LOADED_MODELS` | `2`   | Modelos carregados ao mesmo tempo (LLM + embedding) |

Como definir (depois, **reinicie o Ollama** para valer):

### Windows

Feche o Ollama pela bandeja do sistema (`Quit`) e, no PowerShell:

``` powershell
setx OLLAMA_NUM_PARALLEL 4
setx OLLAMA_MAX_LOADED_MODELS 2
```

Depois abra o Ollama novamente pelo menu Iniciar. (Alternativa gráfica:
`Editar as variáveis de ambiente da conta` no menu Iniciar.)

### macOS

``` bash
launchctl setenv OLLAMA_NUM_PARALLEL 4
launchctl setenv OLLAMA_MAX_LOADED_MODELS 2
```

Feche (`Quit Ollama` na barra de menus) e abra o aplicativo novamente.

### Linux

``` bash
sudo systemctl edit ollama.service
```

No editor que abrir, adicione:

    [Service]
    Environment="OLLAMA_NUM_PARALLEL=4"
    Environment="OLLAMA_MAX_LOADED_MODELS=2"
    Environment="OLLAMA_HOST=0.0.0.0"

Salve e reinicie o serviço:

``` bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

> `OLLAMA_HOST=0.0.0.0` é necessário no Linux para que os containers
> Docker consigam acessar o Ollama (veja a seção 2.4).

## Verificando o servidor

O Ollama já deve estar rodando (aplicativo no Windows/macOS, serviço no
Linux). Confirme:

``` bash
curl http://localhost:11434
```

Resposta esperada: `Ollama is running`. (No PowerShell use
`curl.exe http://localhost:11434` ou abra a URL no navegador.)

Só execute `ollama serve` manualmente se o servidor **não** estiver em
execução. Se aparecer o erro `address already in use`, é porque ele já
está rodando -- não é necessário fazer nada.

------------------------------------------------------------------------

# 2.4 Subindo o restante da stack com Docker

O serviço **Docling** usa a API do **OpenRouter** para extrair o conteúdo
dos documentos. Obtenha uma chave em <https://openrouter.ai/keys> e
coloque-a em um arquivo `.env` dentro do diretório `2-rag`.

Entre no diretório:

``` bash
cd ia_generativa_idp/2-rag
```

Crie o arquivo `.env` a partir do exemplo:

### Windows (PowerShell)

``` powershell
Copy-Item .env.example .env
notepad .env
```

### macOS e Linux

``` bash
cp .env.example .env
# edite o .env e preencha OPENROUTER_API_KEY
```

O Docker Compose lê esse arquivo automaticamente e substitui
`${OPENROUTER_API_KEY}` no `docker-compose.yaml`. O `.env` está no
`.gitignore`, então a chave não vai para o repositório.

Alternativamente, exporte a variável no shell antes de subir a stack:

-   **Windows (PowerShell):** `$env:OPENROUTER_API_KEY = "sk-or-v1-..."`
-   **macOS / Linux:** `export OPENROUTER_API_KEY=sk-or-v1-...`

## Linux: acesso do container ao Ollama

No Docker Desktop (Windows e macOS), o nome `host.docker.internal` já
aponta para o host. No **Linux (Docker Engine)** isso **não** é
automático. Edite o `docker-compose.yaml` e, no serviço `open-webui`,
adicione:

``` yaml
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

Lembre-se também de ter configurado `OLLAMA_HOST=0.0.0.0` (seção 2.3).

## Subindo os serviços

Execute (igual nos três sistemas):

``` bash
docker compose up -d
```

Saída esperada:

    [+] up 4/4
    ✔ Network 2-rag_default     Created
    ✔ Container docling-service Started
    ✔ Container chromadb        Started
    ✔ Container open-webui      Started

Acesse o Open WebUI em <http://localhost:3000>. O primeiro acesso pede
a criação de uma conta de administrador (local).

Para acompanhar os logs e parar os serviços:

``` bash
docker compose logs -f open-webui
docker compose down
```

## Problemas comuns

| Sintoma | Sistema | Solução |
|---|---|---|
| `Cannot connect to the Docker daemon` | Todos | Abra o Docker Desktop (Windows/macOS) ou `sudo systemctl start docker` (Linux). |
| Open WebUI não lista modelos do Ollama | Linux | Confira `extra_hosts` e `OLLAMA_HOST=0.0.0.0`. |
| Porta 3000, 5001 ou 8000 já em uso | Todos | Encerre o processo que usa a porta ou altere o mapeamento em `ports` no `docker-compose.yaml`. |
| Docker lento ou sem iniciar | Windows | Verifique se a virtualização está habilitada na BIOS e se o WSL 2 está instalado (`wsl --install`). |
| Pouca memória / containers sendo encerrados | Windows/macOS | Aumente a memória em `Docker Desktop → Settings → Resources`. |
| `permission denied` ao usar `docker` | Linux | Adicione o usuário ao grupo `docker` (seção 2.1). |
| Ollama em CPU, sem usar a GPU | Windows/Linux | Atualize os drivers NVIDIA/AMD e reinicie o Ollama. |

------------------------------------------------------------------------

# 2.5 Adicionando modelos do OpenRouter no Open WebUI

Acesse o painel administrativo:

    Usuário → Painel do Admin → Configurações → Conexões → API OpenAI

Adicionar nova conexão:

    URL Base da API:
    https://openrouter.ai/api/v1

    Chave da API:
    sk-or-v1-...

------------------------------------------------------------------------

# 2.6 Configurando Docling no Open WebUI

    Usuário → Painel do Admin → Configurações → Documentos

Selecionar **Docling** como mecanismo de extração.

Configuração:

``` json
{
  "pipeline": "vlm",
  "vlm_pipeline_preset": "default",
  "do_ocr": false,
  "do_table_structure": false,
  "images_scale": 2.0,
  "page_range": [1, 200]
}
```

O pipeline `vlm` é o que consome a API key do OpenRouter. O preset
`default` aponta para o preset `openrouter` definido no
`docker-compose.yaml`, que usa o modelo `google/gemini-2.5-flash`.

------------------------------------------------------------------------

# 2.7 Configurando Chunking

    Usuário → Painel do Admin → Configurações → Documentos

Parâmetros:

    Tamanho de chunk: 800
    Sobreposição: 150

------------------------------------------------------------------------

# 2.8 Configurando Embeddings

    Usuário → Painel do Admin → Configurações → Documentos

Modelo:

    qwen3-embedding:0.6b

------------------------------------------------------------------------

# 2.9 Configurando Recuperação

    Usuário → Painel do Admin → Configurações → Documentos → Re-ranking

Parâmetros:

    Pesquisa híbrida: ativada
    Enriquecer texto da pesquisa: ativado
    Motor de reclassificação: Padrão (SentenceTransformers)
    Top K: 30
    Top K reclassificado: 5

------------------------------------------------------------------------

# 2.10 Configurando o Prompt RAG

    Usuário → Painel do Admin → Configurações → Documentos → Modelo RAG

Prompt:

    ### Tarefa
    
    Responda à pergunta do usuário com base **exclusiva** no contexto fornecido.
    
    ---
    
    ### Regras
    
    * Use o contexto como fonte principal. Não priorize conhecimento externo.
    * Se a resposta **não estiver no contexto**, diga explicitamente: "A resposta não está no contexto fornecido."
    * Se usar conhecimento externo, sinalize claramente que está fora do contexto.
    * Não invente informações nem preencha lacunas com suposições.
    * Se o contexto for insuficiente, incompleto ou confuso, informe isso.
    * Se a pergunta for ambígua, peça esclarecimento antes de responder.
    * Parafraseie -- não copie trechos longos.
    
    ---
    
    ### Citações
    
    * Inclua citações imediatamente após cada afirmação suportada pelo contexto.
    * Não use tags XML na resposta final.
    
    ---
    
    ### Saída
    
    Resposta direta, objetiva e sem rodeios, baseada no contexto abaixo:
    
    <context>
    {{CONTEXT}}
    </context>

------------------------------------------------------------------------

# 2.11 Indexando a Base de Conhecimento

No Open WebUI:

    Workspace → Conhecimento → Novo Conhecimento

Configurar:

    Nome: PL 2338/2023
    Objetivo: Legislação da IA no Brasil

Adicionar documentos:

    Adicionar conteúdo → Carregar arquivos

Selecionar arquivos do diretório:

    docs_rag

------------------------------------------------------------------------

# 2.12 Perguntas de Teste

## Teste 1 -- Sem base de conhecimento

Pergunte no chat:

    Qual é o objetivo do PL 2338/2023?

------------------------------------------------------------------------

## Teste 2 -- Com base de conhecimento

Abra um chat **com a base selecionada**.

### Pergunta A

    Qual é o objetivo do PL 2338/2023?

Resposta esperada:

-   Regulação da IA
-   Proteção de direitos fundamentais
-   Promoção de inovação

------------------------------------------------------------------------

### Pergunta B

    Quais são as categorias de risco de sistemas de IA previstas no projeto?

Resposta esperada:

-   risco inaceitável (ou excessivo)
-   alto risco
-   risco mínimo (ou limitado)

------------------------------------------------------------------------

### Pergunta C

    O que caracteriza um sistema de IA de alto risco?

Resposta esperada:

Um sistema de IA é considerado **de alto risco** quando seu uso pode
afetar:

-   direitos fundamentais
-   segurança
-   acesso a serviços essenciais

especialmente em **contextos decisórios relevantes ou setores
sensíveis**.

------------------------------------------------------------------------

### Pergunta D

    Quem será a autoridade reguladora da IA segundo o projeto?

Resposta esperada:

O projeto propõe a criação de um **Sistema Nacional de Regulação e
Governança de Inteligência Artificial**, em vez de definir uma
autoridade única já existente.

------------------------------------------------------------------------

### Pergunta E

    Quais são as penalidades (mudar para "sanções" em seguida) previstas?

Resposta esperada:

-   Advertência
-   Multa simples
-   Multa diária
-   Publicização da infração
-   Bloqueio do sistema de IA
-   Proibição parcial ou total do sistema

------------------------------------------------------------------------

# Resultado Esperado

Ao final do laboratório, o aluno terá:

-   Um **ambiente RAG completo rodando localmente**
-   Pipeline de **extração → chunking → embeddings → busca → re-ranking
    → geração**
-   Capacidade de **consultar documentos com LLMs**

