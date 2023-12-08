# poc-node-actions

Se ha creado un ejemplo de proceso basico de integración continua y despliegue continuo, en este ejemplo se crea un app en node.js para generar en ejemplo y las capacidades que tiene Github Actions en la migración que se quieren realizar.

## Configuración de la App

### Prerequisitos

- Realizar la instalación [node.js](https://nodejs.org/en/download/package-manager)
- ```node --version```


Crear el folder donde se creara la aplicación

```bash
mkdir poc-node-actions
cd poc-node-actions
code . # debe tener instalado y configurado visual studio code
```

Instalar express generator

```bash
npm install -g express-generator
express poc-node-actions --view pug

cd poc-node-actions
npm install

```

Se puede ejecutar localmente el proyecto para verificar

```bash
npm start
```

Para mas información ver [enlace](https://code.visualstudio.com/docs/nodejs/nodejs-tutorial)

### Agregando docker file

Se agrega el ``Dockerfile`` con la siguiente estructura:

```Docker
FROM node:16
# Create app directory
WORKDIR /usr/src/app
# Install app dependencies
# A wildcard is used to ensure both package.json AND package-lock.json are copied
# where available (npm@5+)
COPY package*.json ./
RUN npm install
# If you are building your code for production
# RUN npm install --only=production
# Bundle app source
COPY . .
EXPOSE 3000
CMD [ "npm", "start" ]
```

### Agregando actions al proyecto

Se agrega workflow para ejecuta, costruir y cargar la imagen docker un DockerHub usando GitHub Actions en el proyecto, para esto se debe agregar el archivo ``node.js.yml`` en el folder ``.github/workflows``

```yml
# This is a basic workflow to help you get started with Actions
name: CI-CD

# Controls when the action will run. 
on:
  # Triggers the workflow on push or pull request events but only for the main branch
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:
  
# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains multiple jobs
  build_test:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [16.x]
        
    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@v4

      - name: setup node
        uses: actions/setup-node@master
        with:
          node-version: ${{ matrix.node-version }}

      # install applicaion dependencies
      - name: Install dependencies
        run: |
          npm install
          npm ci 
      # build and test the apps     
      # - name: build
      #   run: |
      #     npm run build
      #     npm run test
  push_to_Docker_Hub:
      # The type of runner that the job will run on
      runs-on: ubuntu-latest
      # build docker image and push to docker hub
      # only if the app build and test successfully
      needs: [build_test]

      steps:
        - name: checkout repo
          uses: actions/checkout@v4

        - name: Set up QEMU
          uses: docker/setup-qemu-action@v3
      
        - name: Set up Docker Buildx
          uses: docker/setup-buildx-action@v3

        - name: Login to DockerHub
          uses: docker/login-action@v3
          with:
            username: ${{ secrets.DOCKERHUB_USERNAME }}
            password: ${{ secrets.DOCKERHUB_TOKEN }}
      
        - name: Build and push
          uses: docker/build-push-action@v5
          with:
            context: ./
            file: ./Dockerfile
            push: true
            tags: ${{ secrets.DOCKERHUB_USERNAME }}/pocnode-app:latest
          
        - name: Run the image in a container
          uses: addnab/docker-run-action@v3
          with:
            image: ${{ secrets.DOCKERHUB_USERNAME }}/pocnode-app:latest
            run: |
              echo "runing the docker image"
              echo "Testing the nodejs  app endpoints"
              echo ${{ steps.docker_build.outputs.digest }}
```

Para esto se crea la cuenta en DockerHub para realizar ``push`` de la imagen, el usuario y token se encuentran en los ``Secrets`` en las configuraciones del repositorio.
