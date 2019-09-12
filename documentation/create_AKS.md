# Azure Kubernetes Service (AKS) クラスタの作成

## 1. リソースグループを作成

```bash
az group create --name akshandsonlab --location japaneast
```

## 2. AKS クラスタを作成

### 2-1. 最新の Kubernetes バージョンを bash 変数に取得

```bash
version=$(az aks get-versions -l japaneast --query 'orchestrators[-1].orchestratorVersion' -o tsv)
```

### 2-2. AKS クラスタの作成 (サービスプリンシパル作成権限のあるアカウントで実行する場合)

```bash
az aks create --resource-group akshandsonlab \
    --kubernetes-version $version \
    --location japaneast \
    --enable-addons monitoring \
    --generate-ssh-keys \
    --node-vm-size Standard_DS1_v2 \
    --name <unique-aks-cluster-name>
```

> `<unique-aks-cluster-name>` に一意の AKS クラスター名を入力します。AKS 名には、3 - 31 文字数で、文字、数字、およびハイフンのみを含めることができます。名前は文字で始まる必要があり、文字または数字で終わる必要があります。AKS の展開には 10 - 15 分かかる場合があります。

> 新規で SSH 鍵を作成するのではなく、もし既存の SSH 鍵があって、それを利用したい場合は、AKS クラスタ作成時に `--generate-ssh-keys` オプションを指定するのはなく `--ssh-key-value` オプションに自分の SSH 鍵を指定してください。

#### (補足) サービスプリンシパル作成でエラーになる場合の対処方法

上記の `az aks create` コマンドでサービスプリンシパルが作成できない等のエラー (`Operation failed with status: 'Bad Request'` など) が出た場合、次の方法を試してみてください。

1. 次のコマンドを実行して、`aksServicePricipal.json` ファイルが存在するか確認する
   
  ```bash
  cat ${HOME}/.azure/aksServicePrincipal.json | jq
  ```

2. ファイルが存在する場合は、次のような出力が得られるので、`<client-secret>` と `<service-principal-id>` の値をメモしておく

  ```json
  {
    "863ccd07-cee4-4bf7-8988-efb6f0e5c102": {
      "client_secret": <client-secret>,
      "service_principal": <service-principal-id>
    }
  }
```

3. 続いて、次のコマンドを実行して、実際に `<service-principal-id>` のサービスプリンシパルが作成されているか確認する

```bash
az ad sp show --id <service-principal-id>
```

4. サービスプリンシパルが作成されている場合は、次のような出力が得られる

```json
{
  "accountEnabled": "True",
  "addIns": [],
  "alternativeNames": [],
  "appDisplayName": <aks-cluster-name>,
  "appId": <service-principal-id>,
  ...
```

5. サービスプリンシパルが作成されていることが確認できれば、次のコマンドで `<client-secret>` と `<service-principal-id>` を直接指定して AKS クラスタを作成する

```bash
az aks create --resource-group akshandsonlab \
    --kubernetes-version $version \
    --location japaneast \
    --enable-addons monitoring \
    --generate-ssh-keys \
    --node-vm-size Standard_DS1_v2 \
    --name <unique-aks-cluster-name> \
    --service-principal <service-principal-id> \
    --client-secret <client-secret>
```
> これでもエラーになる場合や、1-4 の手順でサービスプリンシパルが作成されていない場合には、サービスプリンシパル作成権限のないアカウントで実行されている可能性があるため、次の 2-3 の手順をおこなってください。

### 2-3. AKS クラスタの作成 (2-2 の手順でエラーになる場合、またはサービスプリンシパル作成権限のないアカウントで実行する場合)

1. まず、次のコマンドを実行して、サービスプリンシパルが作成できるか確認する
```bash
# az ad sp create-for-rbac --skip-assignment -n <サービスプリンシパル名>
az ad sp create-for-rbac --skip-assignment -n my-aks-sp
```

> このコマンド実行でエラーになる場合、サービスプリンシパル作成権限（対象アカウントに `Owner/所有者` または `User Access Administrator/ユーザーアクセス管理者` ロールが割り当てられている）をもったアカウント保持者に依頼して AKS クラスタ作成のためのサービスプリンシパルを作成してもらう必要があります

> NOTE: サービスプリンシパルを作成するには、アプリケーションを Azure AD テナントに登録し、サブスクリプション内のロールにアプリケーションを割り当てるためのアクセス許可が必要です。対象アカウントに `Owner/所有者` または `User Access Administrator/ユーザーアクセス管理者` ロールが割り当てられていることをご確認ください。ロールに関する詳細は [こちら](https://docs.microsoft.com/ja-jp/azure/role-based-access-control/rbac-and-directory-admin-roles#azure-rbac-roles) を参照ください

2. 上記コマンドを実行すると次のような結果出力が得られるので、この出力結果をメモする（または、サービスプリンシパルの作成を管理者に依頼した場合には、その結果を入手する）

```json
{
  "appId": "72155cb8-d08d-4435-9052-4b8e8f7352a9",
  "displayName": "my-aks-sp",
  "name": "http://my-aks-sp",
  "password": "13420a1e-ec5f-4a83-b763-b6f599e88899",
  "tenant": "72f988bf-86f1-41af-91ab-2d7cd011db47"
}
```

3. 先の出力の `appId` と `password` の値を使って、次のコマンドで AKS クラスタを作成する
```bash
az aks create --resource-group akshandsonlab \
    --kubernetes-version $version \
    --location japaneast \
    --enable-addons monitoring \
    --generate-ssh-keys \
    --node-vm-size Standard_DS1_v2 \
    --name <unique-aks-cluster-name> \
    --service-principal <appIdの値>> \
    --client-secret <passwordの値>>
```

> `<unique-aks-cluster-name>` に一意の AKS クラスター名を入力します。AKS 名には、3 - 31 文字数で、文字、数字、およびハイフンのみを含めることができます。名前は文字で始まる必要があり、文字または数字で終わる必要があります。AKS の展開には 10 - 15 分かかる場合があります。

> 新規で SSH 鍵を作成するのではなく、もし既存の SSH 鍵があって、それを利用したい場合は、AKS クラスタ作成時に `--generate-ssh-keys` オプションを指定するのはなく `--ssh-key-value` オプションに自分の SSH 鍵を指定してください。

## 3. クラスタアクセス用資格情報取得

1. 次のコマンドを実行して Kubernetes クラスタアクセスのための資格情報を取得
```sh
az aks get-credentials -g user_akstest -n userakscluster
```

2. 次のコマンドでクラスタにアクセスしてノードを取得する
```sh
kubectl get nodes
```
3. 次のような出力が得られれば、ノードが作成されていることが確認できる
```
NAME                       STATUS    ROLES     AGE       VERSION
aks-nodepool1-40291275-0   Ready    agent   2m55s   v1.12.7
aks-nodepool1-40291275-1   Ready    agent   2m48s   v1.12.7
aks-nodepool1-40291275-2   Ready    agent   2m53s   v1.12.7
```

---
[Back](../readme.md)
