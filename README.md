# Flask API Dockerized

Este repositório contém uma aplicação Flask para gerenciar comentários, que é executada dentro de um container Docker usando o Gunicorn como servidor WSGI. A configuração é feita para ser facilmente gerenciada com Docker Compose.

## Requisitos

Antes de começar, certifique-se de ter os seguintes softwares instalados na sua máquina:

- [Docker](https://www.docker.com/get-started)
- [Docker Compose](https://docs.docker.com/compose/install/)

## Estrutura do Projeto

A estrutura básica do projeto é a seguinte:
.
├── app.py # Arquivo principal da aplicação Flask
├── Dockerfile # Dockerfile para construir a imagem da aplicação
├── docker-compose.yml # Configuração do Docker Compose
├── requirements.txt # Dependências da aplicação Python
└── README.md # Documentação do projeto (este arquivo)


## Aplicação Flask (`app.py`)

O código da aplicação Flask é o seguinte:

```python
from flask import Flask, jsonify, request
import time
import logging

app_name = 'comentarios'
app = Flask(app_name)
app.debug = True

comments = {}

# Configuração para coleta de métricas #
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@app.before_request
def before_request():
    request.start_time = time.time()

@app.after_request
def after_request(response):
    if hasattr(request, 'start_time'):
        runtime = time.time() - request.start_time
        status_code = response.status_code
        method = request.method
        path = request.path
        remote_addr = request.remote_addr
        content_length = response.calculate_content_length()
        logger.info("%s %s %s %s %s %s", method, path, status_code, runtime, remote_addr, content_length)
    return response

@app.route('/api/comment/new', methods=['POST'])
def api_comment_new():
    request_data = request.get_json()

    email = request_data['email']
    comment = request_data['comment']
    content_id = '{}'.format(request_data['content_id'])

    new_comment = {
        'email': email,
        'comment': comment,
    }

    if content_id in comments:
        comments[content_id].append(new_comment)
    else:
        comments[content_id] = [new_comment]

    message = 'comment created and associated with content_id {}'.format(content_id)
    response = {
        'status': 'SUCCESS',
        'message': message,
    }
    return jsonify(response)

@app.route('/api/comment/list/<content_id>')
def api_comment_list(content_id):
    content_id = '{}'.format(content_id)

    if content_id in comments:
        return jsonify(comments[content_id])
    else:
        message = 'content_id {} not found'.format(content_id)
        response = {
            'status': 'NOT-FOUND',
            'message': message,
        }
        return jsonify(response), 404

@app.route('/')
def index():
    return 'Application running', 200

if __name__ == "__main__":
    app.run()


## Dependências (requirements.txt)
As dependências da aplicação estão listadas no arquivo requirements.txt:

click==7.1.2
Flask==1.1.2
gunicorn==20.0.4
itsdangerous==1.1.0
Jinja2==2.11.2
MarkupSafe==1.1.1
Werkzeug==1.0.1

## Configuração do Docker
### Dockerfile
O Dockerfile define a imagem Docker para a aplicação Flask:

FROM python:3.8-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["gunicorn", "app:app"]

Nota: A porta 5000 é a porta padrão usada pelo Flask. A aplicação estará disponível nesta porta dentro do container.

### Docker Compose (docker-compose.yml)
O arquivo docker-compose.yml define o serviço Flask e as configurações necessárias:

version: '3.8'

services:
  flask_app:
    build: .
    image: my-app:tag
    ports:
      - "5000:5000"
    volumes:
      - .:/app
    command: gunicorn app:app --bind 0.0.0.0:5000

### Executando a Aplicação
Construir a Imagem Docker
Para construir a imagem Docker, execute:

docker-compose build

### Iniciar a Aplicação
Para iniciar a aplicação, use o Docker Compose:

docker-compose up

A aplicação estará disponível em http://localhost:5000.

### Parar a Aplicação
Para parar a aplicação e remover os containers, use:

docker-compose down

### Reiniciar com Novo Código
Se você fez alterações no código, pode reiniciar a aplicação com:

docker-compose down
docker-compose up --build

### Verificação de Logs
Se precisar verificar os logs da aplicação, execute:

docker-compose logs

## Exemplo Requisição POST para Adicionar um Comentário

Aqui está um exemplo de como podemos fazer uma requisição usando o cURL para adicionar um comentário à aplicação Flask.
1. Requisição POST para Adicionar um Comentário
Use o comando abaixo no seu terminal para enviar uma requisição POST com o cURL:
curl -X POST http://localhost:5000/api/comment/new \
-H "Content-Type: application/json" \
-d '{"email": "exemplo@teste.com", "comment": "Um comentário de teste via cURL.", "content_id": "123"}'

### Explicação:

-X POST: Especifica que o método da requisição é POST.
http://localhost:5000/api/comment/new: URL para a qual a requisição será enviada.
-H "Content-Type: application/json": Define o cabeçalho Content-Type como application/json, indicando que o corpo da requisição está no formato JSON.
-d '{"email": "exemplo@teste.com", "comment": "Este é um comentário de teste via cURL.", "content_id": "123"}': Define o corpo da requisição, que contém o e-mail, o comentário e o content_id.
2. Requisição GET para Listar Comentários
Para listar os comentários de um determinado content_id, você pode usar o seguinte comando cURL:
curl http://localhost:5000/api/comment/list/123

### Explicação:

http://localhost:5000/api/comment/list/123: URL para a qual a requisição será enviada, onde 123 é o content_id cujo os comentários você deseja listar.
Resultado Esperado
Requisição POST: Se a requisição POST for bem-sucedida, você verá uma resposta JSON confirmando que o comentário foi criado. Algo como:

{
    "status": "SUCCESS",
    "message": "comment created and associated with content_id 123"
}

Requisição GET: A requisição GET retornará uma lista de comentários associados ao content_id fornecido. Se houver comentários, você verá algo como:

[
    {
        "email": "exemplo@teste.com",
        "comment": "Este é um comentário de teste via cURL."
    }
]

Se não houver comentários ou o content_id não existir, você receberá uma mensagem de erro.

Esses exemplos mostram como interagir com sua API usando cURL no terminal, o que pode ser útil para testar rapidamente suas rotas.

