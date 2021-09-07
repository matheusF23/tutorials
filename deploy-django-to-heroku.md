# **COMO FAZER DEPLOY DE UMA APLICAÇÃO DJANGO PARA O HEROKU USANDO GITLAB E DOCKER**

## **Requisitos:**

- Aplicação django básica
- Conta no heroku e no gitlab
- Projeto criado no gitlab e app criado no Heroku
- Conhecimento de git e docker

## **Etapas:**

### **Configurar variáveis de ambiente**

#### *Heroku*

Uma vez criado o app no Heroku, va para a aba de *settings* e na parte de *config Vars* click em *Reveal Config Vars* para adicionar as variáveis de ambiente.

Adicione a `SECRET_KEY` do projeto, caso seja um servidor de desenvolvimento adicione a variável `DEBUG`com valor `True`. 

Caso vá utilizar um banco de dados do Heroku, adicione na aba de *Resources* em *Add-ons* o banco  (Ex: heroku postgres). Daí ja será criada uma variável de ambiente chamada `DATABASE_URL`; basta utilizá-la na configuração do seu banco de dados no `settings.py`.

Adicione ainda a variável `DISABLE_COLLECTSTATIC` com o valor 1.

#### *Gitlab*

No gitlab, dentro do projeto vá em `settings->CI/CD`. Na parte de variáveis, click em *expand*. 

Adicione a variável `SECRET_KEY`.

Daí será preciso adicionar variáveis referentes ao Heroku.

Adicione o nome do seu app em, por exemplo, uma variável chamada `HEROKU_APP_NAME`.

Daí será preciso a sua API KEY do Heroku. Para isso, vá nas configurações da sua conta do Heroku, clicando na sua foto, e então desça até *API KEY* e clique em *Reveal* para mostrá-la. Então copie e cole como valor de uma variável de ambiente do gitlab, chamada, por exemplo, de `HEROKU_API_KEY`

### **Configurar pipeline**

Agora, vamos fazer a configuração do arquivo `gitlab-ci.yml`.

Abaixo encontra-se um exemplo básico de configuração, com apenas o estágio de deploy. Você pode colocar mais estágios, como estágio de teste, análise de código, dividir o deploy para um deploy em ambiente de homologação e ambiente de produção...

```yml
before_script:
  - echo "deb http://toolbelt.heroku.com/ubuntu ./" > /etc/apt/sources.list.d/heroku.list
  - wget -O- https://toolbelt.heroku.com/apt/release.key | apt-key add -
  - apt-get update
  - apt-get install -y heroku-toolbelt
  - gem install dpl
  - apt-get install libjpeg-dev zlib1g-dev

stages:
  - deploy

deploy:
  stage: deploy
  script:
    - dpl --provider=heroku --app=$HEROKU_APP_NAME --api-key=$HEROKU_API_KEY
  only:
    - homolog

```

### **Arquivos de configuração do Heroku**

Como desejamos subir um aplicação com docker, ou seja, rodar nossa aplicação em um container docker que será executado no Heroku, precisamos antes dizer para o heroku que nossa aplicação será um container.

Para isso, instale o CLI do heroku e execute os seguintes comandos:
```sh
heroku login  # Faça login em sua conta Heroku
heroku stack:set container -a ${APP_NAME}   # Set seu app para ser um container docker, passando o nome do seu app na flag -a
```

Pronto, agora basta configurar o arquivo o [heroku.yml](https://devcenter.heroku.com/articles/build-docker-images-heroku-yml) para podermos usar o docker. Por exemplo:

```yml
build:
  docker:
    web: app/Dockerfile

release:
  command:
    - python3 manage.py migrate
  image: web

run:
  web: gunicorn core.wsgi:application -b 0.0.0.0:${PORT}

```

Caso não estiver querendo usar o docker, apenas aqui muda. Você precisará ter dois arquivos na raiz do projeto. O `runtime.txt` (para indicar que é uma aplicação usando python e qual sua versão) e o `Procfile` (para indicar qual comando rodar ao iniciar a aplicação), contendo o seguinte:

`runtime.txt`
```
python-3.8.7
```

`Procfile`
```
web: gunicorn core.wsgi --log-file - 
```

É importante que o arquivo `requirements.txt` também esteja na raiz do projeto. Tive erro em uma configuração de pastas em que o requirements não estava na raiz.

### Outras dicas

Para monitorar os logs da sua aplicação no heroku, use:

```sh
heroku logs --tail -a=${APP_NAME}
```

Para entrar na sua máquina que está rodando no Heroku:

```sh
heroku run sh -a=${APP_NAME}
```
