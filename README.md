# MLOps no GCP: Tutorial passo-a-passo

<p align="center">
  <img src="https://user-images.githubusercontent.com/68903879/139598908-fe0bba9c-7d6a-4824-8a8e-60596d8ae364.png">
</p>

Para uma breve explicação da solução, [clique aqui](https://github.com/pedroafleite/mlops_exercise/tree/main/diagram).

## Pré-requisitos:
- Anaconda instalado.
- Google Cloud SDK Shell instalado.

## Criando um ambiente virtual

Primeiramente, seguiremos boas práticas se criarmos uma virtualenv. Esta virtualenv terá Python 3.8 (downgrade da versão atual 3.9), que será necessária para rodar alguns recursos mais para frentes. Como possuo Anaconda no meu computador, posso facilmente executar estes passos com:

`conda create -n "venv" python=3.8.0 ipython`

Podemos verificar se a nova venv foi criada junto às demais com:

`conda env list`

E ativar a venv que acabamos de criar com:

`conda activate venv`

## Configurações iniciais e permissões

Dentro do Google Cloud SDK Shell, podemos acessar nossa conta através de:

`gcloud auth login` 

Iremos criar uma nova conta com uma PROJECT_ID distinta. O Google Cloud irá recomendar algumas IDs disponíveis. No nosso caso, utilizamos a PROJECT_ID **mlops-1635587444840** (Este não é um projeto existente, apenas um exemplo. Em projetos que serão enviados para produção, as variáveis devem ser propriamente escondidas através da declaração de variáveis de ambientes.)

`gcloud projects create mlops-1635587444840`

Com projeto criado, iremos criar uma nova service account, aonde designaremos as permissões necessárias para acessar os serviços. Então, criaremos uma service account chamada **service-mlops-1382162467** com um display name **service-mlops**.

`gcloud iam service-accounts create service-mlops-1382162467 --description="service account for mlops exercise" --display-name="service-mlops"`

Podemos visualizar todas as service accounts criadas:

`gcloud iam service-accounts list`

Aqui, designamos à conta de serviço que criamos acima uma role de admin do PubSub como política de acesso:

`gcloud projects add-iam-policy-binding mlops-1635587444840 --member="serviceAccount:service-mlops-1382162467@mlops-1635587444840.iam.gserviceaccount.com" --role="roles/pubsub.admin"`

Em um teste como o nosso, podemos escolher uma role bastante abrangente (no caso, admin), ou até mesmo mais abrangente do que isso (owner, por exemplo). No entanto, em projetos que seguirão para a produção, é importante restringirmos ao máximo o papel das roles de cada service account para tornar o app seguro. Agora, criamos um json com as credenciais para acessarmos os serviços:

`gcloud iam service-accounts keys create mlops-485cf5c206db.json --iam-account=service-mlops-1382162467@$mlops-1635587444840.iam.gserviceaccount.com` 

Este json deve estar propriamente escondido atrás do [.gitignore](https://github.com/pedroafleite/mlops_exercise/blob/main/.gitignore), já que estamos compartilhando este repositório no Github.

## Preparando o Storage

Vamos criar um novo bucket no Storage com o nome **ml_input**:

`gsutil mb gs://ml_input`

Permita que a service account possa acessar o Storage como admin:

`gcloud projects add-iam-policy-binding mlops-1635587444840 --member="serviceAccount:service-mlops-1382162467@mlops-1635587444840.iam.gserviceaccount.com" --role="roles/storage.admin"`

## Configurando o PubSub

Dentro da venv, instale as dependências do PubSub para acessá-lo no Python:

`pip install --upgrade google-cloud-pubsub`

Crie um novo tópico chamado **topic-brigade**:

`gcloud pubsub topics create topic-brigade`

E uma subscrição à este tópico chamada **sub-brigade**:

`gcloud pubsub subscriptions create sub-brigade --topic topic-brigade`

## Configurando o Dataflow

Instale as dependências:

`pip install apache_beam[gcp]`

Permita que sua conta instale apitools e certifique-se que a APÌ do Dataflow está ativa.

Com [pubsub_gcs.py](https://github.com/pedroafleite/mlops_exercise/blob/main/pubsub_gcs.py), iremos executar o Dataflow:

`python pubsub_gcs.py --project=mlops-1635587444840 --region=us-central1 --input_topic=projects/mlops-1635587444840/topics/topic-brigade --output_path=gs://ml_input/samples/output --runner=DataflowRunner --window_size=2 --num_shards=1 --temp_location=gs://ml_input/temp`


