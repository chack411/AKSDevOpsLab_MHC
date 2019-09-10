# Azure DevOps を利用した AKS へのマルチコンテナーアプリケーションの CI/CD 構成

> Original lab in English is: [Deploying a multi-container application to Azure Kubernetes Services](https://github.com/microsoft/azuredevopslabs/tree/master/labs/vstsextend/kubernetes)

## 概要

[**Azure Kubernetes Service (AKS)**](https://azure.microsoft.com/en-us/services/kubernetes-service/) は、Azure で Kubernetes を使用する最も簡単な方法です。**Azure Kubernetes Service (AKS)** は、ホストされた Kubernetes 環境を管理し、コンテナー オーケストレーションの専門知識を持たずに、コンテナー化されたアプリケーションを迅速かつ簡単にデプロイおよび管理できるようにします。また、アプリケーションをオフラインにすることなく、リソースをオンデマンドでプロビジョニング、アップグレード、およびスケーリングすることにより、継続的な運用と保守の負担を軽減します。Azure DevOps は、継続的なビルド オプションを使用して、デプロイと信頼性を向上させる Docker イメージの作成に役立ちます。
AKS を使用する最大の利点の 1 つは、クラウドでリソースを作成する代わりに、デプロイとサービス マニフェスト ファイルを通じて Azure Kubernetes クラスター内にリソースとインフラストラクチャを作成できることです。

### ラボ シナリオ

このラボでは、Docker 化された ASP.NET Core Web アプリケーションである **MyHealthClinic** (MHC) を使用し、**Azure DevOps** を使用して**Azure Kubernetes Service (AKS)** で実行されている **Kubernetes** クラスターにデプロイします。
> フロントの **Load Balancer** やバックエンドの **Redis Cache** などのデプロイメントとサービスをスピンアップするための定義で構成される **mhc-aks.yaml** マニフェスト ファイルがあります。MHC アプリケーションは、ロード バランサーと共に **mhc-front** ポッドで実行されます。
       
![AKS Workflow](images/aksworkflow.png)

Kubernetes を初めて使用する方向けに、この実習で使用する用語説明のリンクが [こちら](documentation/readme.md) にまとめられています。

### このラボで取り上げる内容

次のタスクを実習します。

* Azure Container Registry (ACR), AKS and Azure SQL server の作成

* Azure DevOps Demo Generator ツールを利用した .NET Core アプリケーションの Azure DevOps チーム プロジェクトをプロビジョニングする

* Azure DevOps の継続的デプロイ (CD) を使用して、アプリケーションとデータベースのデプロイを構成する

* Azure Boards でのワークアイテム作成から Pull Request によるソースコード修正によるアプリケーションの自動ビルドとデプロイの基本的なワークフローをおこなう

## 環境準備

1. [はじめに](../Setup/) を参照して、この演習に必要な事前準備を確認します。

2. [Azure DevOps デモ ジェネレータ](http://azuredevopsdemogenerator.azureweb.net/?TemplateId=77372&Name=AKS) リンクをクリックし、ベースとなるデモプロジェクトを **Azure DevOps** にプロビジョニングします。

このラボでは、上記のリンクをクリックすると既に選択されている **Azure Kubernetes Service** テンプレートが使用されます。この演習には、いくつかの追加の拡張機能が必要であり、プロセス中に自動的にインストールできます。

![AKS Template|50%](images/akstemplate.png)

## 環境のセットアップ

このラボでは次の Azure のリソースの構成が必要になります。

|Azure resources                      | Description|
|-------------------------------------|------------|
|![Azure Container Registry](images/container_registry.png) Azure Container Registry | プライベートな Docker イメージの格納に使用します |
|![AKS](images/aks.png) AKS | Azure 上に構成されるマネージドな Kubernetes サービスで、AKS 内のポッドに Docker イメージがデプロイされます|
|![Azure SQL Database](images/sqlserver.png) Azure SQL Dtabase | Azure 上に構成されるマネージドな SQL データベース サービスです|

### Azure Cloud Shell を使った Azure リソースの準備

1. Azure 管理ポータルで [Azure Cloud Shell](https://docs.microsoft.com/en-in/azure/cloud-shell/overview) を開き、**Bash** を選択します。

2. **CLI を使った AKS の作成**:

   i. ご希望のリージョンで利用可能な最新の Kubernetes バージョンを bash 変数に取得します。`<region>` を使用したいデータセンターリージョン (例えば `japaneast` ) に置き換えます。

      ```bash
     version=$(az aks get-versions -l <region> --query 'orchestrators[-1].orchestratorVersion' -o tsv)
      ```
   
   ii. リソースグループ `akshandsonlab` を作成します。

    ```bash
     az group create --name akshandsonlab --location <region>
    ```

   iii. AKS を作成します。

    ```bash
    az aks create --resource-group akshandsonlab --name <unique-aks-cluster-name> --enable-addons monitoring --kubernetes-version $version --generate-ssh-keys --location <region>　--node-vm-size Standard_DS1_v2
    ```
    
    `<unique-aks-cluster-name>` に一意の AKS クラスター名を入力します。AKS 名には、3 - 31 文字数で、文字、数字、およびハイフンのみを含めることができます。名前は文字で始まる必要があり、文字または数字で終わる必要があります。AKS の展開には 10 - 15 分かかる場合があります。
    
3. **Azure Container Registry(ACR) の作成**:
   
   次のコマンドを実行して、Azure コンテナー レジストリ (ACR) を使用したプライベート コンテナー レジストリを作成します。

    ```bash
    az acr create --resource-group akshandsonlab --name <unique-acr-name> --sku Standard --location <region>
    ```
    `<unique-acr-name>` に一意の ACR 名を入力します。ACR 名には英数字のみを含める場合があり、5 - 50 文字の間でなければなりません。

4. **AKS サービス プリンシパル アクセスを ACR に付与する**:
   
   AKS サービス プリンシパルを使用して、AKS クラスターが Azure コンテナー レジストリ (ACR) に接続することを承認します。`<aks-cluster-name>` と `<acr-name>` を先に作成した名前に置き換え、以下のコマンドを実行します。

    ```bash
    # Get the id of the service principal configured for AKS
    CLIENT_ID=$(az aks show --resource-group akshandsonlab --name <aks-cluster-name> --query "servicePrincipalProfile.clientId" --output tsv)

    # Get the ACR registry resource id
    ACR_ID=$(az acr show --name <acr-name> --resource-group akshandsonlab --query "id" --output tsv)

   # Create role assignment
   az role assignment create --assignee $CLIENT_ID --role acrpull --scope $ACR_ID
   ```

   > 詳細な情報はこちらのドキュメントも参照してください：[Authenticate with Azure Container Registry from Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-auth-aks)

5. **Azure SQL Database の作成**:
   
   次のコマンドを実行して Azure SQL Database サーバーを作成します。
   
   `<unique-sqlserver-name>` には一意の Azure SQL Database サーバー名を入力します。サーバー名は大文字を使用した命名規則 (Camel 形式など) をサポートしていないため、全て小文字を使用します。
    
    ```bash
    az sql server create -l <region> -g akshandsonlab -n <unique-sqlserver-name> -u sqladmin -p P2ssw0rd1234
    ```
    
    続いて、次のコマンドを実行してデータベースを作成します。

    `<unique-sqlserver-name>` には、先に作成した SQL Database サーバー名を指定します。

    ```bash
    az sql db create -g akshandsonlab -s <unique-sqlserver-name> -n mhcdb --service-objective Basic --max-size 100M
    ```

    次に、SQL Database サーバーのファイアウォールの設定を行います。次のコマンドを実行して、Azure サービスからのアクセスを許可します。

    ```bash
    az sql server firewall-rule create -g akshandsonlab -s <unique-sqlserver-name> -n AllowAllWindowsAzureIps --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0
    ```

6. 以上で次のコンポーネント - **Azure コンテナー レジストリ (ACR)**、**Azure Kubernetes Service (AKS)**、**Azure SQL Database** が展開されます。これらの各コンポーネントに個別にアクセスし、演習 1 で使用する詳細をメモします。
   
   ![Deploy to Azure](images/azurecomponents.png)

7. SQL Database の **mhcdb** を選択して、**サーバー名** をメモします。

   ![Deploy to Azure](images/getdbserverurl.png)

8. リソース グループに移動し、作成されたコンテナー レジストリを選択し、**ログイン サーバー** 名をメモします。

    ![Deploy to Azure](images/getacrserver.png)

これで、このラボの演習をおこなうために必要なすべての Azure リソースの準備が完了しました。

## 演習 1: ビルド & リリース パイプラインの構成

[Azure DevOps デモ ジェネレータ](http://azuredevopsdemogenerator.azurewebweb.net/?TemplateId=77372&Name=AKS) を通じて、Azure DevOps 組織で AKS プロジェクトを作成していることを確認します (詳細は環境準備の章を参照してください)。AKS や Azure コンテナー レジストリ (ACR) などの Azure リソースをビルド定義とリリース定義に手動でマップします。

1. Azure DevOps で作成したチームプロジェクトを開き、**Pipelines → Pipelines** を表示します。

      ![build](images/pipelines.png)

2. **MyHealth.AKS.build** pipeline を選択して **Edit** をクリックします。
   
   ![build](images/editbuild.png)

3. **Run services** タスクで、**Azure subscription** ドロップダウンリストから使用する Azure サブスクリプションを選択して、**Authorize** をクリックします。

    ![azureendpoint](images/endpoint.png)

    Azure 資格情報との接続の認証を求められます。[OK] ボタンをクリックした後に空白の画面が表示された場合は、ブラウザでポップアップ ブロッカーを無効にし、手順を再試行してください。

    これにより、サービス プリンシパル認証 (SPA) を使用して Microsoft Azure サブスクリプションへの接続を定義し、合わせてセキュリティで保護する **Azure Resource Manager Service Endpoint** が作成されます。このエンドポイントは、**Azure DevOps** および **Azure** の接続に使用されます。

    > サブスクリプションが一覧表示されていない場合、または既存のサービス プリンシパルを指定する場合は、[サービス プリンシパルの作成](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/connect-to-azure?view=vsts) を参照してください。

4. 認証が成功した後、ドロップダウン - **Azure subscription** および **Azure Container Registry** で適切な値を選択します。

5. パイプライン内の **Build services**、**Push services**、および **Lock services** タスクについてもこの手順を繰り返します。

    ![updateprocessbd](images/acr.png)

    各ビルドタスクの概要は下記の通りです。

    |Tasks|Usage|
    |-----|-----|
    |**Replace tokens**| **mhc-aks.yaml** 内の ACR 名と **appsettings.json** 内のデータベース接続文字列を置き換えます|
    |![icon](images/icon.png) **Run services**| aspnetcore-build:1.0-2.0 などの必要なイメージをプルして、**.csproj** で記述されているパッケージを復元することで、ビルドに適切な環境を準備します|
    |![icon](images/icon.png) **Build services**| **docker-compose.yml** ファイルで指定された Docker イメージをビルドし、**$(Build.BuildId)** と **latest** のタグを追加します|
    |![icon](images/icon.png) **Push services**| Docker イメージ **myhealth.web** を Azure Container Registry (ACR) にプッシュします|
    |![publish-build-artifacts](images/publish-build-artifacts.png) **Publish Build Artifacts**| **mhc-aks.yaml** と **myhealth.dacpac** ファイルを Azure DevOps 内の Artifact ドロップ場所にプッシュし、リリース定義で使用できるようにします|

    > **applicationsettings.json** ファイルには、この演習の冒頭で作成された Azure Database への接続に使用されるデータベース接続文字列の詳細が含まれています。

   > **mhc-aks.yaml** マニフェスト ファイルには、Azure Kubernetes Service (AKS) にデプロイされる **deployments**、**services**、**pods** の構成の詳細が含まれています。マニフェスト ファイルは次のようになります。

      ![](images/aksmanifest.png)

   > Deployment Manifest についての詳細な情報はこちらをご参照ください：[AKS Deployments and YAML manifests](https://docs.microsoft.com/en-us/azure/aks/concepts-clusters-workloads#deployments-and-yaml-manifests)

6. **Variables** タブをクリックします
      
    ![](images/variables.png)

    **Pipeline Variables** で使用する **ACR** および **SQLserver** の値を、環境のセットアップの章でサービスを作成した時に指定した名前で更新します。

    ![updateprocessbd](images/updatevariablesbd.png)

7. **Save & queue** の **Save** (Save & queue ではない) をクリックして変更を保存します

    ![updateprocessbd](images/savebuild.png)

8. **Pipelines \| Releases** に移動し、 **MyHealth.AKS.Release** pipeline を選択して **Edit** をクリックします

   ![release](images/release.png)

9. **Dev** stage を選択して **View stage tasks** をクリックし、パイプライン タスクを表示します

   ![releasetasks](images/viewstagetasks.png)

10. **Dev** 環境では、**DB deployment** フェーズの **Execute Azure SQL: DacpacTask** タスクにある **Azure Service Connection Type** のドロップダウンから **Azure Resource Manager** を選択し、
**Azure Subscription** ドロップダウンから **Available Azure service connections** に表示される Azure サブスクリプションへの接続名を選択します。

    ![update_CD3](images/dbdeploytask.png)

1. **AKS deployment** フェーズの **Create Deployments & Services in AKS** タスクを選択します。
      
    ![](images/aksdeploytask.png)

    **Azure subscription**、**Resource Group**、**Kubernetes cluster** の各ドロップダウンリストの値を更新します。続いて **Secrets** セクションを開き、 **Azure subscription** と **Azure container registry** の値をドロップダウンから更新します。
    
    **Update image in AKS** タスクにおいても同様の設定を行います。
     
     ![](images/aksdeploytask2.png)
    
    * **Create Deployments & Services in AKS** タスクは **mhc-aks.yaml** ファイルで指定された構成に従い、AKS に deployments と services を作成します。最初のポッドは最新の Docker イメージがプルされて作成されます。
    * **Update image in AKS** は、指定されたリポジトリから BuildID に対応する適切なイメージをプルアップし、AKSで実行されている**mhc-front pod** に Docker イメージをデプロイします。
    * **mysecretkey** という secret が、コマンド `kubectl create secret` を使用して、Azure DevOps を介してバックグラウンドで AKS クラスターに作成されます。この secret は、Azure コンテナー レジストリ (ACR) から `myhealth.web` イメージを取得する場合の認証に使用されます。

2. リリース定義の **Variables** セクション タブを選択し、**Pipeline Variables** にある **ACR** と **SQLserver** の値を、環境のセットアップの章でサービスを作成した時に指定した名前で更新し、**Save** ボタンをクリックします。
   
   > **Database Name** は **mhcdb** に、**Server Admin Login** は **sqladmin** に、**Password** は **P2ssw0rd1234** に設定されています。もし、Azure SQL Database サーバーの作成中に異なる詳細を入力した場合は、それに応じて値を更新します。

   ![releasevariables](images/releasevariables.png)

## 演習 2: Trigger a Build and deploy application

In this exercise, let us trigger a build manually and upon completion, an automatic deployment of the application will be triggered. Our application is designed to be deployed in the pod with the **load balancer** in the front-end and **Redis cache** in the back-end.

1. Select **MyHealth.AKS.build** pipeline. Click on **Run pipeline**

    ![manualbuild](images/runpipeline.png)

2. Once the build process starts, select the build job to see the build in progress.
    
    ![clickbuild](images/buildprogress.gif)
    

3. The build will generate and push the docker image to ACR. After the build is completed, you will see the build summary. To view the generated images navigate to the Azure Portal, select the **Azure Container Registry** and navigate to the **Repositories**.

    ![imagesinrepo](images/imagesinrepo.png)

4. Switch back to the Azure DevOps portal. Select the **Releases** tab in the **Pipelines** section and double-click on the latest release. Select **In progress** link to see the live logs and release summary.

    ![releaseinprog](images/releaseinprog.png)

    ![release_summary1](images/release_summary1.png)

5. Once the release is complete, launch the [Azure Cloud Shell](https://docs.microsoft.com/en-in/azure/cloud-shell/overview) and run the below commands to see the pods running in AKS:

    1. Type **`az aks get-credentials --resource-group yourResourceGroup --name yourAKSname`** in the command prompt to get the access credentials for the Kubernetes cluster. Replace the variables `yourResourceGroup` and `yourAKSname` with the actual values.

         ![Kubernetes Service Endpoint](images/getkubeconfig.png)

    2. **`kubectl get pods`**

        ![getpods](images/getpods.png)

        The deployed web application is running in the displayed pods.

6. To access the application, run the below command. If you see that **External-IP** is pending, wait for sometime until an IP is assigned.

    **`kubectl get service mhc-front --watch`**

    ![watchfront](images/watchfront.png)

7. Copy the **External-IP** and paste it in the browser and press the Enter button to launch the application.

    ![finalresult](images/finalresult.png)

### Access the Kubernetes web dashboard in Azure Kubernetes Service (AKS)

Kubernetes includes a web dashboard that can be used for basic management operations. This dashboard lets you view basic health status and metrics for your applications, create and deploy services, and edit existing applications. Follow [these instructions](https://docs.microsoft.com/en-us/azure/aks/kubernetes-dashboard) to access the Kubernetes web dashboard in Azure Kubernetes Service (AKS).

![finalresult](images/aksdashboard.png)

## Summary

[**Azure Kubernetes Service (AKS)**](https://azure.microsoft.com/en-us/services/container-service/){:target="_blank"}  reduces the complexity and operational overhead of managing a Kubernetes cluster by offloading much of that responsibility to the Azure. With **Azure DevOps** and **Azure Container Services (AKS)**, we can build DevOps for dockerized applications by leveraging docker capabilities enabled on Azure DevOps Hosted Agents.


## Reference

Thanks to **Mohamed Radwan** for making a video on this lab. You can watch the following video that walks you through all the steps explained in this lab

<figure class="video_container">
  <iframe width="560" height="315" src="https://www.youtube.com/embed/4DUhc0MjdUc" frameborder="0" allowfullscreen="true"> </iframe>
</figure>
