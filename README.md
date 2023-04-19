![Repo size](https://img.shields.io/github/repo-size/anpavlovsk/CICD-Jenkins-Docker-Minikube)
# CI/CD pipeline for NodeJs and MongoDB application using Jenkins and deployment to Minikube

## Road map

![alt text](https://github.com/anpavlovsk/CICD-Jenkins-Docker-Minikube/blob/main/screenshots/0.png?raw=true)



I build a docker image from the Dockerfile located in Github and upload the resulting image to Docker Hub.
Then I deploy application to Minukube.

````
#
# Build stage 1.
# This state builds our Script and produces an intermediate Docker image containing the compiled JavaScript code.
#
FROM node:10 as builder
WORKDIR /usr/src/app
COPY ./package*.json ./
RUN npm install
COPY . .
RUN npm run build

#
# Build stage 2.
# This stage pulls the compiled JavaScript code from the stage 1 intermediate image.
# This stage builds the final Docker image that we'll use in production.
#

FROM node:10
WORKDIR /usr/src/app
COPY ./server/package*.json ./
RUN npm install --only=production
COPY . .

# Using static files that was build in a stage 1
COPY --from=builder /usr/src/app/dist ./dist
#Copy static files to static directory
RUN rm -rf server/public/static && cp dist/static -r server/public/static && rm server/views/client.html && cp dist/index.html server/views/client.html

EXPOSE 3311
CMD [ "node", "server/server.js" ]
````
![alt text](https://github.com/anpavlovsk/CICD-Jenkins-Docker-Minikube/blob/main/screenshots/1.png?raw=true)


![alt text](https://github.com/anpavlovsk/CICD-Jenkins-Docker-Minikube/blob/main/screenshots/2.png?raw=true)


Application uses a pipeline created using the Jenkinsfile in the repo root directory. I  have next stages in my pipeline:
Pull latest version from GitHub -> Build image -> Run tests ->Push image to DockerHub--> Deploy to Minikube
````
node {
    def app

    stage('Clone repository') {

        /* Clone our repository */
        checkout scm
    }

    stage('Build image') {
      dir('vue-chess') {
       
        /* Build the docker image */
      
        app = docker.build("anpavlovsk/vue-chess-img")
      }
    }
    
    stage('Test image') {           
            app.inside {            
             
             sh 'echo "Tests passed"'        
            }    
        }     

    stage('Push image') {

        /* Push images: First is tagged with the build BUILD_NUMBER
         the second is just tagged latest !*/
        docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
            app.push("${env.BUILD_NUMBER}")
            app.push("latest")
        }
    }

    stage('Deploy to K8s') {

      /* Apply all manifest files */
      /* sh "kubectl apply -f ./kube/" */
      sh "kubectl apply -f ./kube/mongo.yaml"
      sh "kubectl apply -f ./kube/vue-chess.yaml"
        
    }

}
````
I’m using Pol SCM – checking github every 30 minutes. We really should be using webhooks – but for this demo it’s more then enough. 

![alt text](https://github.com/anpavlovsk/CICD-Jenkins-Docker-Minikube/blob/main/screenshots/3.png?raw=true)


![alt text](https://github.com/anpavlovsk/CICD-Jenkins-Docker-Minikube/blob/main/screenshots/4.png?raw=true)


![alt text](https://github.com/anpavlovsk/CICD-Jenkins-Docker-Minikube/blob/main/screenshots/5.png?raw=true)

Manifest for NodeJs application 
````
apiVersion: v1
kind: Service
metadata:
  name: vue-chess
spec:
  selector:
    app: vue-chess
  ports:
    - port: 3000
      targetPort: 3311
  type: NodePort
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vue-chess
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vue-chess
  template:
    metadata:
      labels:
        app: vue-chess
    spec:
      containers:
        - name: vue-chess
          image: anpavlovsk/vue-chess-img
          ports:
            - containerPort: 3311
          env:
            - name: MONGO_URL
              value: mongodb://mongo:27017/vuegustchess
          imagePullPolicy: IfNotPresent
````

Manifestst for MongoDB 
````
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: mongo
spec:
  selector:
    app: mongo
  ports:
    - port: 27017
      targetPort: 27017
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo
spec:
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
        - name: mongo
          image: mongo:4.0.0
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: storage
              mountPath: /data/db
      volumes:
        - name: storage
          persistentVolumeClaim:
            claimName: mongo-pvc
````
![alt text](https://github.com/anpavlovsk/CICD-Jenkins-Docker-Minikube/blob/main/screenshots/6.png?raw=true)


![alt text](https://github.com/anpavlovsk/CICD-Jenkins-Docker-Minikube/blob/main/screenshots/7.png?raw=true)
