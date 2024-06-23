# Example Voting App

A simple distributed application running across multiple Docker containers.
- Video demo: https://youtu.be/pnmHKrfwmIU

## Architecture Design

![GitOps-azure-devops-cicd drawio](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/a206551e-24cb-47ec-a738-0c5aa40fea3b)

## Implemtation

### Continuos Integration
#### Create new project on Azure DevOps

![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/a6b3533c-6413-4167-8713-959516d730f1)

- Import your repository
  ```
  https://github.com/dockersamples/example-voting-app.git
  ```
- Go go `Container Registry` to create new repo `my-repo-registry` on Azure/GCP cloud platform

#### Set up agent for runnimg job pipeline
- Go to project setting -> Choose `Agent pools` -> Add agent pool with pool type `Self-hosted`
- Click on this agent -> Click `New agent`, you will see some command to setup agent
- SSH to ubuntu virtual machine
  - Update
    ```
    sudo apt update
    ```
  - Run this command

    ```
    mkdir myagent && cd myagent
    ```
  - Download agent

    ```
    wget https://vstsagentpackage.azureedge.net/agent/3.240.1/vsts-agent-linux-x64-3.240.1.tar.gz
    ```
  - Extract agent

    ```
    tar zxvf ~/Downloads/vsts-agent-linux-x64-3.240.1.tar.gz
    ```
  - Run this command to config

    ```
    ./config.sh
    ```

  - For `Enter server URL`, enter `https://dev.azure.com/{NameOfOrganization}`
  - Enter authentication type (press enter for PAT) > Press on Enter
  - For `Enter personal access token`, go to Setting -> Choose `Personal Access Tokens` -> Click on `New Token` with option Full Access and copy this token
  - For `Enter agent pool (press enter for default)` -> azurehost
  - Press Enter for other
  - Finally, run this command
    ```
    ./run.sh
    ```
  - Go to check that

    ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/bef77662-8ab2-4e78-919e-53db85aa9ac2)

- Next, install docker

  ```
  sudo apt install docker.io -y
  ```
- Grant azureuser permission for docker

  ```
  sudo usermod -aG docker samelnguyen08
  sudo systemctl restart docker
  ```

#### Create a pipeline
- Create a pipeline for `vote-service`

  ```yaml
  # Docker
  # Build and push an image to Azure Container Registry
  # https://docs.microsoft.com/azure/devops/pipelines/languages/docker
  
  trigger:
   paths:
     include:
       - vote/*
  
  resources:
  - repo: self
  
  variables:
    # Container registry service connection established during pipeline creation
    dockerRegistryServiceConnection: '8b19a049-e8e2-443b-b21e-a1cbae6f9713'
    imageRepository: 'votingapp'
    containerRegistry: 'cicdapprepo.azurecr.io'
    dockerfilePath: '$(Build.SourcesDirectory)/result/Dockerfile'
    tag: '$(Build.BuildId)'
  
  pool:
   name: 'azurehost'
  
  stages:
  - stage: Build
    displayName: Build 
    jobs:
    - job: Build
      displayName: Build
      steps:
      - task: Docker@2
        displayName: Build an image
        inputs:
          containerRegistry: '$(dockerRegistryServiceConnection)'
          repository: '$(imageRepository)'
          command: 'build'
          Dockerfile: 'vote/Dockerfile'
          tags: '$(tag)'
  
  - stage: Push
    displayName: Push 
    jobs:
    - job: Push
      displayName: Push
      steps:
      - task: Docker@2
        displayName: Push an image
        inputs:
          containerRegistry: '$(dockerRegistryServiceConnection)'
          repository: '$(imageRepository)'
          command: 'push'
          tags: '$(tag)'
  
  # - stage: Update
  #   displayName: Update 
  #   jobs:
  #   - job: Update
  #     displayName: Update
  #     steps:
  #     - task: ShellScript@2
  #       inputs:
  #         scriptPath: 'scripts/updateK8sManifests.sh'
  #         args: 'vote $(imageRepository) $(tag)'  
  ```
- Similar pipeline for `vworker-service` and `result-service`

### Continuos Delivery
#### Create kubernetes cluster on Azure/GCP 
- Choose name `gitops-cluster`
- Enable autoscaling node for min `1` and max `2`, enable for IP of pod

#### Connect to Kubernetes cluster
- For Azure Kubernetes services

  ```
  az aks get-credentials --resource-group resourcegroupname --name clustername
  ```
- For Google Kubernetes Engine

  ```
  gcloud container clusters get-credentials CLUSTER_NAME --region=CLUSTER_REGION
  ```
#### Setup ArgoCD on Kubernetes Cluster with namespace argocd
- Refer this [doc](https://argo-cd.readthedocs.io/en/stable/getting_started/) to download ArgoCD
- Do this command

  ```
  kubectl create namespace argocd
  kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  ```
- Check status of pod

  ```
  kubectl get pods -n argocd
  ```
- Get password of argoCD, to list secrets

  ```
  kubectl get secrets -n argocd
  ```
- To show password
  ```
  kubectl get secrets argocd-initial-admin-secret -n argocd -o yaml
  ```
  ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/62c44333-e5f1-45d9-913c-5bdd65017090)

- To decode this password

  ```
  echo [password] | base64 --decode
  ```
- Expose End-point for ArgoCD, list service

  ```
  kubectl get svc -n argocd
  ```

- Edit change to `NodePort` for `argocd-server`
  ```
  kubectl edit svc argocd-server -n argocd
  ```

- Rememeber this port

  ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/9b48205e-c9d5-4b8f-8ed4-ddcd6f5c2a49)

- To get external IP for node

  ```
  kubectl get nodes -o wide
  ```
> To access ArgoCD app, you need to allow port in bound rule 
#### Configure ArgoCD
- Login with username `admin` and password in previous step
- Connect to repo, go to `Settings` -> Choose `Repositories` -> Click on `Connect Repo`

  ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/ee654476-7bc9-4139-a2cf-a28b40dc9615)

- You need to create token from Azure DevOps and specify this URL https repo like this

  ```
  https://[token-devops-azure]@dev.azure.com/hieukato321/test-DevOps/_git/test-DevOps
  ```

  ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/f4b6ed88-19a8-4de2-b3d4-d384d6ef7324)

> Save this token in note pad

- Make sure connection is success

  ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/cb57a0b3-5756-4af1-b419-22991965ae7e)

#### Deploy workload on Kubernetes cluster
- Check that manifets file in Azure repo

  ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/cb5456f2-bb1d-4b70-b882-f3c0fb896eda)

- Create new application, go to `Applications` -> Click on `New App`
- With `Application Name`, type `voteapp-service`, project name `default`
- make sure `Sync policy` set to `Automatic`
- Sleect soruce repo link and specify this folder path `k8s-specifications`

  ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/ba259e69-a696-4cb9-b4df-5ab797a6f29b)

- Select your Kubernetes cluster with namespace `default`

  ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/ab58e62f-9ba1-4c0f-b403-690943d801c8)

#### Create script to update new image for deployment vote
- In Azure repo, create new file updateK8sManifests.sh in `scripts` folder

  ```bash
  #!/bin/bash
  set -e  # Exit on any error
  set -x  # Print each command being executed
  
  # Set the repository URL
  REPO_URL="https://[azuredevops_token]@dev.azure.com/hieukato321/test-DevOps/_git/test-DevOps"
  
  # Clone the git repository into the /tmp directory
  TEMP_DIR="/tmp/temp_repo"
  TEMP_DIR="${TEMP_DIR%$'\r'}"  # Remove any trailing \r character
  
  if [ -d "$TEMP_DIR" ]; then
      rm -rf "$TEMP_DIR"
  fi
  
  git clone "$REPO_URL" "$TEMP_DIR"
  
  # Navigate into the cloned repository directory
  cd "$TEMP_DIR"
  
  # Make changes to the Kubernetes manifest file(s)
  # Example: Update image tag in a deployment.yaml file
  sed -i "s|image:.*|image: cicdapprepo.azurecr.io/$2:$3|g" k8s-specifications/$1-deployment.yaml
  
  # Add the modified files
  git add .
  
  # Configure Git user
  git config --global user.email "user@example.com"
  git config --global user.name "user"
  
  # Commit the changes
  git commit -m "Update Kubernetes manifest"
  
  # Push the changes back to the repository
  git push
  
  # Cleanup: remove the temporary directory
  rm -rf "$TEMP_DIR"

  ```

- Update new stage `update` for `vote-service` pipeline

  ```yaml
  - stage: Update
    displayName: Update 
    jobs:
    - job: Update
      displayName: Update
      steps:
      - script: |
          if command -v apt-get >/dev/null; then
              sudo apt-get update && sudo apt-get install -y dos2unix
          elif command -v yum >/dev/null; then
              sudo yum install -y epel-release && sudo yum install -y dos2unix
          else
              echo "Neither apt-get nor yum found. Please install dos2unix manually."
              exit 1
          fi
  
          # Convert the script to Unix format
          dos2unix scripts/updateK8sManifests.sh
  
          # Make the script executable
          chmod +x scripts/updateK8sManifests.sh
  
          # Run the script with arguments
          ./scripts/updateK8sManifests.sh vote $(imageRepository) $(tag)
        displayName: Run updateK8sManifests.sh
  ```
- Run the pipeline and check the result

  ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/7b872870-d3b2-4261-b070-61b1cfc4d455)

- Go to this file for check new image update

  ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/dff9d945-fda8-4e2b-a7a1-21bfb0a9cbb9)

- You can go back gitops application to check new update for image

  ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/cfbc811d-9d38-4920-b843-e8eb955ac05a)

#### Modify the Application Reconciliation Timeout in Argo CD
- Run this command

  ```
  # kubectl describe configmaps argocd-cm -n argocd
  kubectl edit cm argocd-cm -n argocd
  ```
- Add this at the end of the file

  ```
  data:
    timeout.reconciliation: 10s
  ```
  
#### Fix error for ImagePullBackOff permission pull image from private container registry
- You will see the problem like this

  ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/ffeecf7c-9f06-4308-9189-9a64ad9aa556)

- Go to Container Reigistry to get crenditail

  ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/bf4feb8d-18b6-4572-b4ca-d6663eb8683c)

- Command to create ACR ImagePullSecret

  ```
  kubectl create secret docker-registry <secret-name> \
    --namespace <namespace> \
    --docker-server=<container-registry-name>.azurecr.io \
    --docker-username=<service-principal-ID> \
    --docker-password=<service-principal-password>
  ```
- I had username `cicdapprepo` and password `3AA9/vwi3cjJkPnAchu8nJVRuJJKiEsKOATs81LUfH+ACRBebRJI0`
- Update new change in `vote-deployment.yaml`

  ```
  imagePullSecrets:
      - name: acr-secret
  ```

  ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/0f08e422-fec8-4504-bd58-4955ea16be2d)

- Run this command
  ```
  kubectl create secret docker-registry acr-secret \
    --docker-server=cicdapprepo.azurecr.io \
    --docker-username=cicdapprepo \
    --docker-password=3AA9/vwi3cjJkPnAchu8nJVRuJJKiEsKOATs81LUfH+ACRBebRJI0
  ```

#### Expose vote service to access application
- Run this command to find external IP

  ```
  kubectl get nodes -o wide
  ```

- Run this command to find `Port` of vote service

  ```
  kubectl get svc
  ```

  ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/2488f025-2683-4443-8c97-83d19435f5a7)

> Make sure to allow port 31000 in firewall rule

### Test application
- Make change in repo to `snow` and `sunny`

  ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/12a72666-360c-4ed9-9c7a-a9424ada8284)

- The pipeline will run automatically

  ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/5f339254-45e9-4556-b03d-54ccae203147)

- Check that new image update with lastest ID build

  ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/2e8d1fa4-64ef-4c2f-b3f8-4f73fe5e6d72)

- Go to ArgoCD check that new pod update

  ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/30857ee9-0b6d-4ae1-a87e-1fe79f64b4d0)

- Go back to application to verify the result

  ![image](https://github.com/hieunguyen0202/GitOps-azure-devops-cicd/assets/98166568/62369859-5be7-427a-9e6a-a03450a1f16e)
