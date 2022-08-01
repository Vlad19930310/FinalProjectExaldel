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
ос - лучше линукс(убунту) но можно и винда
поставить докер, google cli, helm3, kubectl, git, vscode 


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


### Step 3 ()
### Step 4 ()
### Step 5 ()

