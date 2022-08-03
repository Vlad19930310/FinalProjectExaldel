# Environment
  - Cloud: GCP(Google Cloud Platform);
  - CI/CD tool: GitLab;
  - Application: Wagtail(https://habr.com/ru/post/582898/).

# Stages 
  - Git integration;
  - Setup/configure CI/CD;
  - Application/s should be containerized;
  - Scheduled backups for DB and all critical data;
  - Logging and monitoring for your services;
  - Security;
  - Use Kubernetes as an orchestration (cloud provider is recommended);
  - The project must be documented, step-by-step guides to deploy from scratch; 
  - EXTRA: SonarQube integration.

## Prepare development environment (Raman Pitselmakhau)
ос - Linux or Windows
apps - Docker, Google cli, helm3, kubectl, git, VScode or Pycharm


## Setup k8s cluster in GCP
### Step 1 (install gclooud CLI and kubectl) 

- Download and install the gcloud command line tool at its [install page](https://cloud.google.com/sdk/docs/install). It will help you create and communicate with a Kubernetes cluster. 
- Install kubectl (reads kube control), it is a tool for controlling Kubernetes clusters in general. From your terminal, enter:  
```
sudo apt install kubectl
```
### Step 2 (setup cluster by the next commands):
```
gcloud services enable container.googleapis.com 
gcloud container clusters create CLUSTER_NAME --enable-autoscaling --num-nodes 2 --min-nodes 2 --max-nodes 5 --region=europe-central2-a
```
- Connect to your cluster:
```
gcloud container clusters get-credentials kuber 
``` 
- Add another node pool. It will be used to setup monitoring and logging tools.
```
gcloud container node-pools create pool-1 --cluster=kuber --enable-autoscaling --min-nodes=2 --max-nodes=5 --machine-type e2-standard-4 --region=europe-central2-a
```
#### As a result we have a cluster with 2 node pools, with 2 nodes and Autoscalling in everyone:  
[![Screenshot-from-2022-08-01-16-39-47.png](https://i.postimg.cc/15HCkLwZ/Screenshot-from-2022-08-01-16-39-47.png)](https://postimg.cc/jLD4N3M8)


## Initial cluster setting
### Step 1 (deploy the ingress controller)
- deploy the ingress controller and other recources we need with the following command:
```
#create namespace ngress-nginx and deploy in it ingress-controller and other recources 
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.3.0/deploy/static/provider/cloud/deploy.yaml
```
- get your external ip by the command:
```
kubectl get svc -n ingress-nginx
```
[![Screenshot-from-2022-08-01-19-13-32.png](https://i.postimg.cc/Znxcn1wr/Screenshot-from-2022-08-01-19-13-32.png)](https://postimg.cc/4HntFFkd)


### Step 2 (registration domain and subdomains)
- Registrate domain with subdomains and create DNS A-records with your external IP in any service:
[![Screenshot-from-2022-08-01-22-37-43.png](https://i.postimg.cc/CxtvKc9d/Screenshot-from-2022-08-01-22-37-43.png)](https://postimg.cc/VJq9GWfc)


### Step 3 (install all cert-manager) 
- Install all cert-manager components:
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.yaml
```

### Step 4 (install Cluster Issuer)
- Isntall Cluster Issuer from Helm Chart in namespace ingress-nginx/ Don't forget to change field 'e-mail':
```
helm install cluster-issuer ClusterIssuer-helmChart -n ingress-nginx
```

### Step 5 (add information in ingresses)
- Add information in every ingress file


## Configure CI/CD (Raman Pitselmakhau)
### Step 1 (create account and project)
  - Создайте аккаунт на [GitLab](https://docs.gitlab.com/ee/user/profile/account/create_accounts.html);
  - Создайте [проект](https://docs.gitlab.com/ee/user/project/working_with_projects.html).
### Step 2 (deploy gitlab runner to cluster)
  - Выбирете ваш проект в "Menu".
  - Зайдите в секцию "Settings".
  - Выбирете вкладку "CI/CD".
  - Найдите раздел "Runners" и разверните его.
  - Выбирете "Show runner installation instructions" в разделе "Specific runners".
  - Переключитесь на вкладку "Kubernetes".
  - Нажмите на "View installation instructions".
  - В верхней части открывшейся странице вы увидете заголовок "Installing GitLab Runner using the Helm Chart".
  - Добавьте репозиторий GitLab Helm командой: 
    ```
      helm repo add gitlab https://charts.gitlab.io
    ```
  - Пропустите шаг с командой "helm init" так как она нужна для Helm 2 а мы работай с Helm 3
  - Перед третьим пунктом надо создать NAMESPACE и изменить CONFIG_VALUES_FILE
    #### Create namespace 
      - Согласно Best practice при деплое нового компонента в кластер стоит создать отдельный namespace так как в случае фатальной ошибки вы сможете убрать все ваши изменения просто удалив namespace, а так же это помогает лучше ориентироваться в кластере, контролировать права внутри конкретного namespace. Так что создадим новый namespace кломандой:
      ``` 
      kubectl create namespace gitlab-runner
      ```
    #### Change values.yaml 
      - Теперь настало время изменить CONFIG_VALUES_FILE. Для этого, командой показанной ниже, скопируйте файл к себе локально:
      ```
      mkdir test
      cd test
      helm fetch gitlab/gitlab-runner
      ```
      - После зайдите в папку "test/gitlab-runner" и откройте файл "values.yaml". Далее найдите и замените значения так как в приведенные ниже файле:
      [diff file](docs/values.diff)
      - runnerRegistrationToken: "And this registration token:" В это поле вставьте токен из GitLab -> "project name" -> Settings -> CI/CD -> Runners -> Specific runners -> "And this registration token:"
    #### Create google service account
    Для настройки кластера и создания service-account будем использовать несколько инструкций
    [workload identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity#migrate_applications_to)
    [csi-driver](https://cloud.google.com/kubernetes-engine/docs/how-to/persistent-volumes/filestore-csi-driver)
    [container registry](https://blog.container-solutions.com/using-google-container-registry-with-kubernetes)
    [configure kubernetes](https://blog.atomist.com/kubernetes-ingress-nginx-cert-manager-external-dns/)
    ```
      gcloud container clusters update kuber --update-addons=GcpFilestoreCsiDriver=ENABLED

      # enable workload identity 
      gcloud container node-pools update default --cluster=kuber --workload-metadata=GKE_METADATA

      # create service-accounts for gitlab-runner
      gcloud iam service-accounts create gitlab-runner --project=internship87task8

      # allow gitlab-runner push images to container registry
      gcloud projects add-iam-policy-binding internship87task8 --member="serviceAccount:gitlab-runner@internship87task8.iam.gserviceaccount.com" --role=roles/storage.admin

      gcloud iam service-accounts add-iam-policy-binding "gitlab-runner@internship87task8.iam.gserviceaccount.com" \
        --member="serviceAccount:internship87task8.svc.id.goog[gitlab-runner/gitlab-runner]" \
        --role=roles/iam.workloadIdentityUser --project=internship87task8
     
      # install gitlab-runner
      helm install --namespace gitlab-runner gitlab-runner -f <CONFIG_VALUES_FILE> gitlab/gitlab-runner

      # annotate kubernetes service account to use external google service-account form the project  
      kubectl annotate serviceaccount gitlab-runner --namespace gitlab-runner iam.gke.io/gcp-service-account=gitlab-runner@internship87task8.iam.gserviceaccount.com
      kubectl create clusterrolebinding gitlab-crb --clusterrole cluster-admin --serviceaccount gitlab-runner:gitlab-runner

      # restart gitlab-runner
      kubectl scale deployment gitlab-runner --replicas=0 -n gitlab-runner
      kubectl scale deployment gitlab-runner --replicas=1 -n gitlab-runner

      # add permissions to manage cluster for users 
      for user in sampleuser1@gmail.com sampleuser2@gmail.com sampleuser3@gmail.com; do
        kubectl create clusterrolebinding "cluster-admin-${user%@*}" clusterrole=cluster-admin --user="$user"     
      done
    ```

  -  Теперь в разделе "Specific runners" в "Available specific runners" вы увидете свой ранер. Так же можете увидеть namespase и поду с помощью команд:
  ```
  kubectl get namespace
  kubectl get po -n gitlab-runner
  ```


## Deploy database and make backups (Raman Pitselmakhau)      
### Step 1 (create namespace, secret, SA)
  - First, you need to create a namespace for DB:
  ```
  kubectl create namespace postgres-app
  ```
  - Then make a secret file:
  ```
  apiVersion: v1
  stringData:
    postgresPassword: "****"
    password: "****"
    replicationPassword: "****"
  kind: Secret
  metadata:
    name: postgre-secret
  type: Opaque
  ```
  - And push this secret to your namespace with this command:
  ```
  kubectl apply -f postgre-secret.yaml -n postgres-app
  ```
  - Create another SA in google cloud with permissions to read image in the container registry. Its backend storage is a backet with name "artifacts.internship87task8.appspot.com"
  ```
  gcloud iam service-accounts create image-pull --project=internship87task8 --display-name=image-pull
  ```
  - Google SA must have the role to read objects from the bucket, the role we need is "storage.objectViewer" and you can set it to your SA by the following command:
  ```
  # for Windows
  #gcloud projects add-iam-policy-binding internship87task8 --member="serviceAccount:image-pull@internship87task8.iam.gserviceaccount.com" --role=roles/storage.objectViewer --condition=title="read-container-registry",expression="resource.name.startsWith(""projects/_/buckets/artifacts.internship87task8.appspot.com"")"
  # for Linux
  gcloud projects add-iam-policy-binding internship87task8 --member="serviceAccount:image-pull@internship87task8.iam.gserviceaccount.com" --role=roles/storage.objectViewer --condition=title="read-container-registry",expression='resource.name.startsWith("projects/_/buckets/artifacts.internship87task8.appspot.com")'
  
  # create json key
  gcloud iam service-accounts keys create pull-secret.json --iam-account=image-pull@internship87task8.iam.gserviceaccount.com
  
  # create image-pull secret from json key
  kubectl create secret docker-registry gcr-json-key --docker-server=eu.gcr.io --docker-username=_json_key --docker-password="$(cat pull-secret.json)" --docker-email=image-pull@internship87task8.iam.gserviceaccount.com -n postgres-app
 
  # create a service account in the google project 
  gcloud iam service-accounts create backup --project=internship87task8 --display-name=backup
  
  # connect SA in the kubernetes to SA in google project
  gcloud iam service-accounts add-iam-policy-binding "backup@internship87task8.iam.gserviceaccount.com" --member="serviceAccount:internship87task8.svc.id.goog[postgres-app/backup]" --role=roles/iam.workloadIdentityUser --project=internship87task8 --condition=title="write-backup",expression='resource.name.startsWith("projects/_/buckets/backuppostgre")'

  # create service account for the database backup cronjob:
  kubectl create sa backup -n postgres-app

  # annotate SA 
  kubectl annotate serviceaccount backup --namespace postgres-app iam.gke.io/gcp-service-account=backup@internship87task8.iam.gserviceaccount.com
  ```
### Step 2 (install database)
  - Apply this command to fetch  helm chart to your pc:
  ```
  helm fetch bitnami/postgresql
  ```
  - Then go to values.yaml and change it according to my [file](manual/postgres/values.yaml)
  - Install database
  ```
  helm install postgres bitnami/postgresql -f values.yaml -n postgres-app
  ```
  - Then the CI/CD will deploy cronjob from the helm chart in the repo  
  - My congrats now you have PostgreSQL database in your cluster and backups for it


## Installing Wagtail application wich working with postgresql ubuntu 20.04
# Install wagtail  
1. Update system  ```apt-get update``` verify installed python  
2. Install pip ```python get-pip.py```
- pip package manager for python application 
3. Install wagtail ```pip install wagtail```
4. wagtail start mysite
6. cd mysite
7. add to ```requirements.txt``` strings ```psycopg2-binary>=2.8<2.9``` ```gunicorn==20.0.4```
- psycopg2 driver for working with postrgresql
- gunicorn converting requests received from Nginx into a format that your web application can use, and executing code as needed.
8. pip install -r requirements.txt
- requirements.txt store information about: python version, Django, wagtail, psycopg2-binary, gunicorn
9. Docker must be installed on system
10. Configure ```Dockerfile``` with your configuration
11. Buld image ```docker build -t wagtail .``` in root derictory of project
12. Run Postgresql image with parametrs: 
- ```docker run --name <future name your conteiner>``` \ 
- ```-e POSTGRES_PASSWORD=<your password> -d -p 5432:5432 postgres```
13. Testing application and pushing to docker hub
14. Run container from your builded image with wagtaill app
15. Testing image on local machine
16. Push Dockerfile and all dependeces in git repo
17. For deploying application Database container in pod must be installed.
18. For configuring type of Database and connection setting you can modify ```base.py``` in ```mysite/settings``` 
19. For securing connection to Database using Gitlab secret, and kubectl secret.  


## Setup application from Helm Chart
### Step 1 (create namespaces)
- Create namespaces, which we will use for installation versions of our application by the commands:
```
kubectl create namespace dev
kubectl create namespace stage
kubectl create namespace prod
```
### Step 2(install start versions of application)
- Install start versiona of application by the next commands:
```
helm install app-dev app-helmChart --set igress.app.host=dev.prtest.tech -n dev
helm install app-stage app-helmChart --set ingress.app.host=stage.prtest.tech -n stage
helm install app-prod app-helmChart --set ingress.app.host=prtest.tech -n prod
```
Don`t forget to change subdomains on your own.


## Install and setup Sonar Qube

### Step 1 (install Sonar Qube)

- Create namespace ops:
```
kubectl create namespace ops
```

- Install Sonar Qube using Helm Chart:
```
install sonar-qube sonarqube-helmChart -n sonarqube
```
- After some minutes check sonarqube on it's host. 
Credentials to log in at the first time:
login: admin
password: admin

### Step 2 (integrate Sonar Qube with your Gitlab)
- Choose Gitlab 

[![Screenshot-from-2022-07-28-21-19-21.png](https://i.postimg.cc/vH26LSrz/Screenshot-from-2022-07-28-21-19-21.png)](https://postimg.cc/6yR3Bc12)

- Choose your Gitlab project 

[![Screenshot-from-2022-07-28-21-25-34.png](https://i.postimg.cc/KvTRBvYZ/Screenshot-from-2022-07-28-21-25-34.png)](https://postimg.cc/HVpYCm5K)
 
- choose your CI tool 

[![Screenshot-from-2022-07-28-21-25-50.png](https://i.postimg.cc/bYD0JtKr/Screenshot-from-2022-07-28-21-25-50.png)](https://postimg.cc/23YL9VjN)

- choose technology of your project and copy generated code

- create file sonar-project.properties and past there your code

- go to Settings-CI/CD-Variables in Gitlab and add the next variables:

[![Screenshot-from-2022-07-28-21-27-54.png](https://i.postimg.cc/vHSc6qfd/Screenshot-from-2022-07-28-21-27-54.png)](https://postimg.cc/9rZW6BLx)

[![Screenshot-from-2022-07-28-21-28-35.png](https://i.postimg.cc/c4NNkQW8/Screenshot-from-2022-07-28-21-28-35.png)](https://postimg.cc/Yj8VhLhp)
 
 [![Screenshot-from-2022-07-28-21-28-45.png](https://i.postimg.cc/ryNhcWW4/Screenshot-from-2022-07-28-21-28-45.png)](https://postimg.cc/233d710j)

 - update your ci configuration file

 [![Screenshot-from-2022-07-28-21-29-48.png](https://i.postimg.cc/s2f01CG3/Screenshot-from-2022-07-28-21-29-48.png)](https://postimg.cc/75cNQcNR)


## Logging and monitoring (Raman Pitselmakhau)
### Step 1(ECK)
  - To let monitoring work, a cluster must have kubernetes-monitoring-server installed. This component   provides metrics for internal cluster resources.
    And a tool that will collect metrics from populated metric endpoints. There are several of them:
      - Prometheus
      - ELK
      - etc....
  - I chose ELK with special deployment method for Kubernetes. It is ECK stack -Elastic Cloud on Kubernetes.
  The tool is built on the Kubernetes Operator pattern that extends Kubernetes orchestration capabilities 
  and simplifies setup, upgrades, snapshots, scaling, high availability, security of Elasticsearch and Kibana on Kubernetes.
  There is a [guide](https://www.elastic.co/guide/en/cloud-on-k8s/current/index.html)
  - Installation includes several steps:
       1. Deploy ECK operator via helm chart into cluster 
       2. Create and deploy a manifest file for Elasticsearch & Kibana.  You can see my manifest [here](manual/ECK/eck.yaml)
       3. Then apply the port forward command to make a possibility to open Kibana on localhost and the second command to take the secret with password:
      ```
      kubectl port-forward service/eck-kibana-kb-http 5601 -n elastic-system.
       kubectl get secret eck-es-es-elastic-user -o go-template='{{.data.elastic | base64decode }}'
      ```
       Open the link [https://localhost:5601](https://localhost:5601/), enter a default username "elastic" and the password from  the command above.
       4. Go to "Integrations" -> "Kubernetes" -> turn on a necessary features, install the integration and get a config file for agents. 
          Also, you need to change this config to use ECK operator custom resource to install agents in the cluster. 
       5. Here is [mine](manual/ECK/elastic-agent-standalone-kubernetes2.yaml).

