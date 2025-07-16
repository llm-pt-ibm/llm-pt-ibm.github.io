---
title: "Construindo API para inferências de LLMs em um servidor IBM Power9"
date: 2025-07-02 # ano-mês-dia
authors: ["Caio Silva"] # Pode ser uma lista
tags: ["LLM", "Power9", "API", "FastAPI"]
translationKey: "tutorial_power9_pt3_en"
summary: "Este post faz parte de uma série de tutoriais cujo objetivo final é construir uma API de Modelos de Linguagem em um servidor Power9. Nesta etapa, vamos desenvolver a API usando FastAPI e a biblioteca Transformers."
draft: false # Mude para true se quiser que o post fique como rascunho
---

## Contexto
Este é o terceiro post de uma série de tutoriais cujo objetivo é mostrar passo a passo como construir uma API de Modelos de Linguagem em um servidor Power9, desde a configuração do sistema operacional até a execução remota de inferências. Já configuramos o sistema operacional, os drivers NVIDIA, CUDA e cuDNN no [<span class="link-personalizado">primeiro post</span>]({{< relref "tutorial_power9_pt1_en.md" >}}), e no [<span class="link-personalizado">segundo post</span>]({{< relref "tutorial_power9_pt2_en.md" >}}) instalamos Conda e PyTorch. Nesta etapa, vamos construir a API usando FastAPI e a biblioteca Transformers, baixando modelos do Hugging Face e executando o servidor web com uvicorn.

A API implementada terá as funcionalidades de gerar API Key, carregar modelos, realizar inferências, obter status e desccaregar modelos. 

**FastAPI**: Framework web moderno para construção de APIs com Python 3.8+, baseado em tipagem estática e assíncrona. Foi projetado para ser rápido, fácil de usar e robusto, tornando o desenvolvimento de APIs mais eficiente.

**Transformers**: Biblioteca de código aberto desenvolvida pela Hugging Face. Fornece acesso prático e eficiente a uma ampla coleção de modelos pré-treinados de última geração para Processamento de Linguagem Natural (PLN), visão computacional e áudio.

**Hugging Face**: Hugging Face é uma plataforma focada em inteligência artificial, conhecida por hospedar modelos de NLP e outras tarefas. O Hugging Face Hub é um repositório colaborativo onde desenvolvedores e pesquisadores podem compartilhar, versionar e baixar modelos prontos para uso, facilitando o acesso e integração de modelos.

**Uvicorn**: Servidor web ASGI (Asynchronous Server Gateway Interface). O Uvicorn é um servidor de alta performance para aplicações Python assíncronas.

## TL;DR
- Este post apresenta o passo a passo para implementar uma API que realiza inferências de Grandes Modelos de Linguagem.
- Usaremos FastAPI e Transformers para desenvolver essa API e Hugging Face para baixar os modelos.

## Configuração do Ambiente
#### Estrutura de Diretórios

Primeiro, vamos criar a estrutura básica do projeto:

```txt
model_api/
├── requirements.txt
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── schemas.py
│   ├── auth.py
│   ├── model_manager.py
│   ├── utils.py
│   └── apikey_store.json
└── README.md (opcional)
```

#### Arquivo `requirements.txt`

Vamos usar FastAPI e Transformers para implementar a API. Além disso, usaremos uvicorn para executar o servidor, pydantic para validação de dados de entrada e torch, que já instalamos no [tutorial anterior]({{< relref "tutorial_power9_pt2_en.md" >}}).

Primeiro, vamos instalar as bibliotecas necessárias e depois preencher o arquivo `requirements.txt`. Lembre-se de ativar o ambiente `conda` se você o criou, para garantir o uso correto do `pytorch`.


```
conda activate llm_api
pip install fastapi uvicorn transformers
```
O arquivo `requirements.txt` ficará assim: 

**requirements.txt**
```txt {linenos=inline}
fastapi>=0.104.0
uvicorn>=0.24.0
torch>=2.0.0
transformers>=4.35.0
pydantic>=2.0.0
```
#### Arquivo de Armazenamento de API Keys

O arquivo `apikey_store.json` será usado para armazenar as chaves de API geradas. Vamos iniciá-lo vazio, contendo apenas `{}`.

**apikey_store.json**
```python {linenos=inline}
{}
```

## Schemas e validação de dados
Os schemas são essenciais para validar os dados de entrada e saída da API. Eles garantem que os dados estejam no formato correto e permitem a geração automática de documentação.

Vamos criar o arquivo `app/schemas.py` com todos os modelos de dados. Teremos quatro modelos: `GenerateRequest`, `LoadModelRequest`, `ApiKeyResponse` e `LDAPUserRequest`.

**schemas.py**
```python {linenos=inline}
from pydantic import BaseModel, Field
from typing import Optional

class GenerateRequest(BaseModel):
    model_name: str = Field(..., description="The name of the model to use for generation.")
    prompt: str = Field(..., description="The input text to generate a response for.")
    max_tokens: Optional[int] = Field(300, description="The maximum length of the generated response.")
    temperature: Optional[float] = Field(1.0, description="The sampling temperature for generation.")
    top_p: Optional[float] = Field(1.0, description="The cumulative probability for nucleus sampling.")
    hf_token: Optional[str] = Field(None, description="The Hugging Face tokenizer to use, if applicable.")


class LoadModelRequest(BaseModel):
    model_name: str = Field(..., description="The name of the model to load.")
    device: Optional[str] = Field("cuda", description="The device to load the model on (e.g., 'cpu', 'cuda').")
    hf_token: Optional[str] = Field(None, description="The Hugging Face tokenizer to use, if applicable.")

class ApiKeyResponse(BaseModel):
    api_key: str = Field(..., description="The API key for accessing the model API.")

class LDAPUserRequest(BaseModel):
    username: str = Field(..., description="The username for LDAP authentication.")
```
- Todas as classes herdam da classe `BaseModel` da biblioteca `pydantic`, obtendo funcionalidades de validação, serialização e documentação automática.
- O campo `Field(...)` define um campo obrigatório sem valor padrão.
- O campo `Field(value)` define um campo obrigatório com `value` como valor padrão.
- O tipo `Optional[type]` indica que o campo é opcional, mas deve ser do tipo `type` se fornecido.

Com os schemas definidos, vamos criar o arquivo responsável pela autenticação via API Key.

## Autenticação e API Keys
O sistema de autenticação protege a API, garantindo que apenas usuários autorizados possam acessar os endpoints. Vamos implementar um mecanismo baseado em API Keys.

Vamos criar o arquivo `app/auth.py` com todas as funcionalidades de autenticação.

**auth.py**
```python {linenos=inline}
import secrets 
import json
from fastapi import HTTPException, Request

APIKEY_STORE_FILE = "app/apikey_store.json"

def load_apikeys():
    try:
        with open(APIKEY_STORE_FILE, "r") as f:
            return json.load(f)
    except FileNotFoundError:
        raise HTTPException(
            status_code=404,
            detail=f"Arquivo de API keys não encontrado: {APIKEY_STORE_FILE}")
    
def save_apikeys(keys: dict):
    with open(APIKEY_STORE_FILE, "w") as f:
        json.dump(keys, f, indent=4)

def generate_apikey(user:str) -> str:
    key = secrets.token_hex(32)
    keys = load_apikeys()
    keys[user] = key
    save_apikeys(keys)
    return key

async def verify_apikey(request: Request) -> bool:
    apikey = request.headers.get("x-API-Key")
    if not apikey:
        raise HTTPException(
            status_code=401,
            detail="API key não fornecida.")
    try:
        keys = load_apikeys()
        if apikey in keys.values():
            return True
    
    except json.JSONDecodeError:
        raise HTTPException(
        status_code=403,
        detail="API key inválida.")
```
- A função `load_apikeys` carrega as informações armazenadas no arquivo `app/apikey_store.json`.
- `save_apikeys` é responsável por salvar o conteúdo no formato JSON.
- A função `generate_apikey` cria uma chave para um usuário e a adiciona ao dicionário, usando o username como chave.
- `verify_apikey` será chamada sempre que uma requisição chegar, para realizar a validação.

## Gerenciador de Modelos e GPU
O `app/model_manager.py` é o coração da API, responsável por carregar, gerenciar e executar os modelos de linguagem. Ele otimiza o uso de GPU/CPU e garante eficiência na geração do texto.

**model_manager.py**
```python {linenos=inline}
import torch 
from transformers import AutoTokenizer, AutoModelForCausalLM
from fastapi import HTTPException
import gc
from .utils import is_model_on_gpu

DEVICE = "cuda" if torch.cuda.is_available() else "cpu"

class ModelManager:
    def __init__(self):
        self.model = None
        self.tokenizer = None
        self.model_name = None

    def load_model(self, model_name: str, hf_token:str = None, device: str = DEVICE):
        if self.model_name != None and self.model_name != model_name:
            print("Removendo modelo carregado anteriormente...")

        self.unload_model()        
        print(f"Carregando modelo {model_name} no dispositivo {device}...")
       
        if self.model_name != model_name:
            try:            
                if hf_token:           
                    self.tokenizer = AutoTokenizer.from_pretrained(model_name, token=hf_token)
                    self.model = AutoModelForCausalLM.from_pretrained(model_name, device_map="balanced", token=hf_token)
                else:
                    self.tokenizer = AutoTokenizer.from_pretrained(model_name)
                    self.model = AutoModelForCausalLM.from_pretrained(model_name, device_map="balanced")
                self.model.eval()
                self.model_name = model_name
                print(is_model_on_gpu(self.model.hf_device_map, self.model_name))
                
            except Exception as e:
                raise HTTPException(status_code=500, detail=f"Erro ao carregar modelo: {str(e)}")
        else:
            print(f"O modelo {model_name} já está carregado.")

    def generate(self, model_name:str, hf_token: str, prompt:str, max_tokens:int = 300, temperature:float = 1.0, top_p:float = 1.0) -> str:
        
        if self.model_name != model_name:
            self.load_model(model_name, hf_token, device=DEVICE)

        if self.model is None or self.tokenizer is None:
            raise HTTPException(status_code=400, detail="Nenhum modelo carregado.")

        try:
            inputs = self.tokenizer(prompt, return_tensors="pt").to(self.model.device)
            with torch.no_grad(): 
                outputs = self.model.generate(**inputs, max_new_tokens=max_tokens,temperature=temperature, top_p=top_p, eos_token_id=self.tokenizer.eos_token_id)
            return self.tokenizer.decode(outputs[0], skip_special_tokens=True)
        except Exception as e:
            raise HTTPException(status_code=500, detail=f"Erro ao gerar texto: {str(e)}")
    
    def get_status(self) -> str:        
        if self.model is None:
            self.unload_model()
            return "Nenhum modelo carregado."       
        return is_model_on_gpu(self.model.hf_device_map, self.model_name)

    def unload_model(self):
        self.model = None
        self.tokenizer = None
        old_model = self.model_name if self.model_name else False
        self.model_name = None

        gc.collect()
        torch.cuda.empty_cache()
        return f"Modelo {old_model} descarregado com sucesso." if old_model else "Nenhum modelo carregado para descarregar."

manager = ModelManager()
``` 
- A função `load_model` carrega o novo modelo na memória, removendo algum modelo que foi carregado anteriormente.
- `generate` é a principal função da API, ela é responsável por realizar a inferência do modelo. Permite alterar os parâmetros: temperature, top_p e max_tokens.
- `get_status` é responsável por informar se existe modelo carregado e se está em GPU ou CPU. 
- A função `unload_model` remove o modelo da memória, limpando o cache do CUDA e utilizando o garbage collector do python para não restar resquícios que possam atrapalhar futuros carregamentos.

## Endpoints da API FastAPI
O arquivo `app/main.py` é onde todos os componentes se conectam. Nele definimos todos os endpoints e a lógica de roteamento da API.

**main.py**
```python {linenos=inline}
from fastapi import FastAPI, Request, HTTPException, Depends
from fastapi.responses import JSONResponse
from app import schemas, model_manager, auth

app = FastAPI()

async def require_api_key(request: Request) -> schemas.LDAPUserRequest:
    user = await auth.verify_apikey(request)
    if not user:
        raise HTTPException(status_code=401, detail="API key invalida.")
    return user

@app.post("/generate_apikey")
async def generate_apikey(payload: schemas.LDAPUserRequest) -> JSONResponse:
    key = auth.generate_apikey(payload.username)
    return JSONResponse(status_code=200, content={"api_key": key})

@app.post("/load_model", dependencies=[Depends(require_api_key)])
async def load_model(payload: schemas.LoadModelRequest) -> JSONResponse:
    try:
        model_manager.manager.load_model(payload.model_name, payload.hf_token, payload.device)
        return JSONResponse(content={"message": f"Modelo {payload.model_name} carregado com sucesso."})
    except Exception as e:
        raise HTTPException(status_code=500, content={"error": str(e)})
    
@app.post("/generate", dependencies=[Depends(require_api_key)])
async def generate(payload: schemas.GenerateRequest)-> JSONResponse:
    try:
        result = model_manager.manager.generate(payload.model_name, payload.hf_token,payload.prompt, payload.max_tokens, payload.temperature, payload.top_p)
        return {"result": result}
    except Exception as e:
        return JSONResponse(status_code=500, content={"error": str(e)})
    
@app.get("/status", dependencies=[Depends(require_api_key)])
async def status()-> JSONResponse:
    str_status = model_manager.manager.get_status()
    return JSONResponse(content={"status": str_status})

@app.post("/unload_model", dependencies=[Depends(require_api_key)])
async def unload_model() -> JSONResponse:
    try:
        str_unload = model_manager.manager.unload_model()
        return JSONResponse(content={"message":str_unload})
    except Exception as e:
        raise HTTPException(status_code=500, content={"error": str(e)})
```

- A função `require_api_key` verifica a API Key sempre que chega uma requisição e retorna o usuário autenticado ou gera erro 401.
- `generate_apikey` gera e retorna uma nova chave de API para o usuário informado.
- `load_model` carrega o modelo especificado. Caso o modelo necessite de um token Hugging Face, a função também recebe esse parâmetro.
- A função `generate` é responsável por fazer o modelo realizar a inferência a partir do prompt e os parâmetros passados.
- Ao chamar o endpoint `status` o usuário recebe o status atual do gerenciador de modelos.
- `unload_model` descarrega o modelo atualmente carregado e retorna uma mensagem de sucesso caso tenha concluído corretamente. 

## Arquivo `utils.py`
O arquivo `app/utils.py` contém a função que verifica se o modelo carregado está totalmente/parcialmente em GPU ou foi carregado em CPU.

**utils.py**
```python {linenos=inline}
def is_model_on_gpu(hf_device_map: dict, model_name: str) -> str:
    if '' in hf_device_map.keys() and hf_device_map[''] == 'cpu':
        return f"Modelo {model_name} carregado totalmente na CPU."
    elif 'cpu' in hf_device_map.values():
        return f"Algumas camadas do modelo {model_name} estão carregadas na CPU."
    else:
        return f"Modelo {model_name} carregado totalmente na GPU."
```

## Executando a API
Para executar a API com o `uvicorn` é muito simples, basta executar um comando com as informações de host e porta para o serviço rodar.

```
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```
- `app:main` se refere ao arquivo `app/main.py` responsável por conectar todos os componentes e receber as requisições realizadas pelo usuário.

- `--host 0.0.0.0` define o endereço IP no qual o servidor Uvicorn irá escutar as requisições. O valor `0.0.0.0` define que este servidor estará acessível de qualquer interface de rede disponível na máquina Power9.

- `--port 8000` especifica a porta na qual o servidor irá escutar as requisições.

- `--reload` flag para ser utilizada em desenvolvimento. Recarrega a aplicação sempre que uma mudança é realizada.

Seguindo estas implementações, você terá uma API capaz de realizar inferências com Modelos de Linguagem baixados do Hugging Face. No [<span class="link-personalizado">próximo tutorial</span>]({{< relref "tutorial_power9_pt4_en.md" >}}) será demonstrado como enviar requisições para a API via curl e python.  