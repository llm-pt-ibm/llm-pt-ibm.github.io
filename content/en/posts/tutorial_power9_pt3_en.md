---
title: "Building an API for LLM inferences on IBM Power9 servers"
date: 2025-07-02 # ano-mês-dia
authors: ["Caio Silva"] # Pode ser uma lista
tags: ["LLM", "Power9", "API", "FastAPI"]
projects: ["llm-eval"]
translationKey: "tutorial_power9_pt3_en"
summary: "This is the third post in a tutorial series that walks through the process of building an LLM API on an IBM Power9 server. In this stage, we will develop the API using FastAPI and the Transformers library."
draft: false # Mude para true se quiser que o post fique como rascunho
---

## Background
This is the third post in a tutorial series designed to show step by step how to build a LLM API on a Power9 server, from operating system setup to remote inference execution. We already configured the operating system, NVIDIA drivers, CUDA, and cuDNN in the [<span class="link-personalizado">first post</span>]({{< relref "tutorial_power9_pt1_en.md" >}}), and installed Conda and PyTorch in the [<span class="link-personalizado">second post</span>]({{< relref "tutorial_power9_pt2_en.md" >}}). In this stage, we will build the API using FastAPI and the Transformers library, downloading models from Hugging Face and running the web server with uvicorn.

The implemented API will support generating API keys, loading models, performing inferences, checking status, and unloading models.

**FastAPI**: a modern web framework for building APIs with Python 3.8+, based on static typing and async programming. It is designed to be fast, easy to use, and robust, making API development more efficient.

**Transformers**: an open-source library developed by Hugging Face. It offers easy and efficient access to a wide collection of state-of-the-art pretrained models for Natural Language Processing (NLP), computer vision, and audio.

**Hugging Face**: Hugging Face is a platform focused on artificial intelligence, known for hosting NLP models and other tasks. The Hugging Face Hub is a collaborative repository where developers and researchers can share, version, and download ready-to-use models, making access and integration easier.

**Uvicorn**: ASGI (Asynchronous Server Gateway Interface) web server. Uvicorn is a high-performance server for asynchronous Python applications.


## TL;DR
- This post provides a step-by-step guide to implementing an API that performs LLM inferences.
- We will use FastAPI and Transformers to develop this API and Hugging Face to download the models.


## Environment Setup
#### Directory Structure

Start by creating the basic project structure:

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
└── README.md (optional)
```

#### `requirements.txt` File

We will use FastAPI and Transformers to build the API. Additionally, we will use uvicorn to run the server, pydantic for input data validation, and torch, which we installed in the [previous tutorial]({{< relref "tutorial_power9_pt2_en.md" >}}).

First, we'll install the required libraries and then populate the `requirements.txt` file. Remember to activate your `conda` environment if you created one, to ensure proper use of `pytorch`.
```
conda activate llm_api
pip install fastapi uvicorn transformers
```
The `requirements.txt` file will look like this:

**requirements.txt**
```txt {linenos=inline}
fastapi>=0.104.0
uvicorn>=0.24.0
torch>=2.0.0
transformers>=4.35.0
pydantic>=2.0.0
```

#### API Key Storage File

The `apikey_store.json` file will store the generated API keys. We will start with it empty, containing only `{}`.

**apikey_store.json**
```python {linenos=inline}
{}
```
## Schemas and Data Validation
Schemas are essential for validating the API's input and output data. They ensure data is in the correct format and enable automatic documentation generation.

We will create the `app/schemas.py` file containing all the data models. We will define four models: `GenerateRequest`, `LoadModelRequest`, `ApiKeyResponse`, and `LDAPUserRequest`.

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

- All classes inherit from `pydantic`'s `BaseModel`, gaining validation, serialization, and automatic documentation features.
- The `Field(...)` declaration defines a required field with no default value.
- The `Field(value)` declaration defines a required field with `value` as its default.
- The `Optional[type]` annotation indicates the field is optional but must be of type `type` if provided.

With the schemas defined, let's create the file responsible for API Key authentication.

## Authentication and API Keys
The authentication system protects your API by ensuring that only authorized users can access the endpoints. We will implement a mechanism based on API Keys.

Let's create the `app/auth.py` file with all the authentication functionalities.

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
            detail=f"API keys file not found: {APIKEY_STORE_FILE}")
    
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
            detail="API key not provided.")
    try:
        keys = load_apikeys()
        if apikey in keys.values():
            return True
    
    except json.JSONDecodeError:
        raise HTTPException(
        status_code=403,
        detail="Invalid API Key")
```

- The `load_apikeys` function loads the information stored in the `app/apikey_store.json` file.
- `save_apikeys` is responsible for saving the content in JSON format.
- The `generate_apikey` function creates a key for a user and adds it to the dictionary using the provided username as the key.
- `verify_apikey` will be called whenever a request arrives, to perform validation.

## Model and GPU Manager
The `app/model_manager.py` is the core of the API, responsible for loading, managing, and running llm. It optimizes GPU/CPU usage and ensures efficient text generation.

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
            print("Removing previously loaded model...")

        self.unload_model()        
        print(f"Loading model {model_name} on device {device}...")
       
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
            print(f"The model {model_name} is already loaded.")

    def generate(self, model_name:str, hf_token: str, prompt:str, max_tokens:int = 300, temperature:float = 1.0, top_p:float = 1.0) -> str:
        
        if self.model_name != model_name:
            self.load_model(model_name, hf_token, device=DEVICE)

        if self.model is None or self.tokenizer is None:
            raise HTTPException(status_code=400, detail="No model loaded.")

        try:
            inputs = self.tokenizer(prompt, return_tensors="pt").to(self.model.device)
            with torch.no_grad(): 
                outputs = self.model.generate(**inputs, max_new_tokens=max_tokens,temperature=temperature, top_p=top_p, eos_token_id=self.tokenizer.eos_token_id)
            return self.tokenizer.decode(outputs[0], skip_special_tokens=True)
        except Exception as e:
            raise HTTPException(status_code=500, detail=f"Error generating text:{str(e)}")
    
    def get_status(self) -> str:        
        if self.model is None:
            self.unload_model()
            return "No model loaded."       
        return is_model_on_gpu(self.model.hf_device_map, self.model_name)

    def unload_model(self):
        self.model = None
        self.tokenizer = None
        old_model = self.model_name if self.model_name else False
        self.model_name = None

        gc.collect()
        torch.cuda.empty_cache()
        return f"Model {old_model} successfully unloaded." if old_model else "No model loaded to unload."

manager = ModelManager()
``` 

- The `load_model` function loads a new model into memory, removing any previously loaded model.
- `generate` is the main function of the API, responsible for performing model inference. It allows adjusting the parameters: temperature, top_p, and max_tokens.
- `get_status` reports whether there is a loaded model and whether it is on the GPU or CPU.
- The `unload_model` function removes the model from memory, clears the CUDA cache, and invokes Python’s garbage collector to avoid leftovers that could interfere with future loads.

## FastAPI API Endpoints
The `app/main.py` file is where all the components come together. In it, we define all the endpoints and the API’s routing logic.

**main.py**
```python {linenos=inline}
from fastapi import FastAPI, Request, HTTPException, Depends
from fastapi.responses import JSONResponse
from app import schemas, model_manager, auth

app = FastAPI()

async def require_api_key(request: Request) -> schemas.LDAPUserRequest:
    user = await auth.verify_apikey(request)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid API Key")
    return user

@app.post("/generate_apikey")
async def generate_apikey(payload: schemas.LDAPUserRequest) -> JSONResponse:
    key = auth.generate_apikey(payload.username)
    return JSONResponse(status_code=200, content={"api_key": key})

@app.post("/load_model", dependencies=[Depends(require_api_key)])
async def load_model(payload: schemas.LoadModelRequest) -> JSONResponse:
    try:
        model_manager.manager.load_model(payload.model_name, payload.hf_token, payload.device)
        return JSONResponse(content={"message": f"Model {payload.model_name} loaded successfully."})
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
- The `require_api_key` function checks the API Key on each request and returns the authenticated user or raises a 401 error.
- `generate_apikey` creates and returns a new API key for the specified user.
- `load_model` loads the specified model. If needed, it also accepts a Hugging Face token.
- The `generate` function makes the model perform inference using the given prompt and parameters.
- Calling the `status` endpoint returns the current status of the model manager.
- `unload_model` unloads the currently loaded model and returns a success message if completed properly.

## `utils.py` File
The `app/utils.py` file contains the function that checks whether the loaded model is fully or partially on the GPU, or if it was loaded on the CPU.

**utils.py**
```python {linenos=inline}
def is_model_on_gpu(hf_device_map: dict, model_name: str) -> str:
    if '' in hf_device_map.keys() and hf_device_map[''] == 'cpu':
        return f"Model {model_name} fully loaded on CPU."
    elif 'cpu' in hf_device_map.values():
        return f"Some layers of the model {model_name} are loaded on the CPU."
    else:
        return f"Model {model_name} fully loaded on GPU."
```

## Running the API
To run the API with `uvicorn`, simply execute a command specifying the host and port for the service to start.

```
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

- `app:main` refers to the `app/main.py` file, which connects all components and handles user requests.

- `--host 0.0.0.0` sets the IP address on which the Uvicorn server will listen. The value `0.0.0.0` allows the server to be accessible from any network interface on the Power9 machine.

- `--port 8000` specifies the port on which the server will listen for requests.

- `--reload` is a flag for development use. It automatically reloads the server whenever changes are made.


BBy following this guide, you'll have a working API capable of running LLM inference using models downloaded from Hugging Face. In the [<span class="link-personalizado">next tutorial</span>]({{< relref "tutorial_power9_pt4_en.md" >}}), we will show how to send requests to the API using curl and Python.
