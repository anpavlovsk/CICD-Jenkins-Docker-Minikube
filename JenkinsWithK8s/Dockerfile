FROM jenkinsci/blueocean

USER root

# Install kubectl
# RUN curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.16.2/bin/linux/amd64/kubectl
RUN curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
RUN chmod u+x kubectl && mv kubectl /bin/kubectl

# Copy config file for kubectl from the cluster from $HOME/.kube
COPY ./kubeconfig /root/.kube
#Copy autorization certificates from $HOME/.minikube
COPY ./minikConfig  /root/.minikube

# Command to run container 
# docker run -u root --rm -d -p 8080:8080 -p 50000:50000 --name jenkcont -v jenkins-data:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock --network minikube jenk
