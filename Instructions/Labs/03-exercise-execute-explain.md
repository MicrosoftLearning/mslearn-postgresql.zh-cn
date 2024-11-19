---
lab:
  title: 执行 EXPLAIN 语句
  module: Understand PostgreSQL query processing
---

# 执行 EXPLAIN 语句

在本练习中，你将了解 EXPLAIN 函数，以及它如何显示 PostgreSQL 规划器为提供的语句生成的执行计划。

## 开始之前

需要有自己的 Azure 订阅才能完成本练习。 如果还没有 Azure 订阅，可以创建一个 [Azure 免费试用版](https://azure.microsoft.com/free)。

## 创建练习环境

在本练习和以后的所有练习中，你将在 Azure Cloud Shell 中使用 Bicep 来部署 PostgreSQL 服务器。
如果已安装这些资源，请跳过部署资源和安装 Azure Data Studio。

### 在你的 Azure 订阅上部署资源

此步骤指导你使用 Azure Cloud Shell 中的 Azure CLI 命令创建资源组并运行 Bicep 脚本，以将完成此练习所需的 Azure 服务部署到你的 Azure 订阅中。

> 注意
>
> 如果你要在此学习路径中执行多个模块，则可以在它们之间共享 Azure 环境。 在这种情况下，你只需完成此资源部署步骤一次。

1. 打开 web 浏览器，导航到 [Azure 门户](https://portal.azure.com/)。

2. 选择 Azure 门户工具栏中的“ **Cloud Shell** ”图标，以打开浏览器窗口底部的新“ [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) ”窗格。

    ![Azure 工具栏屏幕截图，Cloud Shell 图标用红框突出显示。](media/03-portal-toolbar-cloud-shell.png)

    如果出现提示，请选择打开 *Bash* shell 所需的选项。 如果以前使用过 *PowerShell* 控制台，请将其切换到 *Bash* shell。

3. 在 Cloud Shell 提示符下，输入以下内容以克隆包含练习资源的 GitHub 存储库：

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

4. 接下来，运行三个命令来定义变量，以在使用 Azure CLI 命令创建 Azure 资源时减少冗余键入。 变量表示要分配给资源组的名称（`RG_NAME`）、要将资源部署到的 Azure 区域（`REGION`）和随机生成的 PostgreSQL 管理员登录密码（`ADMIN_PASSWORD`）。

    在第一个命令中，分配给相应变量的区域是 `eastus`，但你也可以将其替换为首选位置。

    ```bash
    REGION=eastus
    ```

    以下命令分配要用于资源组的名称，该资源组将容纳本练习中使用的所有资源。 分配给相应变量的资源组名称是 `rg-learn-work-with-postgresql-$REGION`，其中 `$REGION` 是上文指定的位置。 但是，你可以将它更改为符合偏好的任何其他资源组名称。

    ```bash
    RG_NAME=rg-learn-work-with-postgresql-$REGION
    ```

    最后一个命令随机生成 PostgreSQL 管理员登录的密码。 请确保将其复制到安全位置，以便稍后可以使用它连接到 PostgreSQL 灵活服务器。

    ```bash
    a=()
    for i in {a..z} {A..Z} {0..9}; 
       do
       a[$RANDOM]=$i
    done
    ADMIN_PASSWORD=$(IFS=; echo "${a[*]::18}")
    echo "Your randomly generated PostgreSQL admin user's password is:"
    echo $ADMIN_PASSWORD
    ```

5. 如果有权访问多个 Azure 订阅，并且默认订阅不是要为此练习创建资源组和其他资源的订阅，请运行此命令来设置相应的订阅，将 `<subscriptionName|subscriptionId>` 令牌替换为要使用的订阅的名称或 ID：

    ```azurecli
    az account set --subscription <subscriptionName|subscriptionId>
    ```

6. 运行以下 Azure CLI 命令创建资源组：

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

7. 最后，使用 Azure CLI 执行 Bicep 部署脚本，在资源组中预配 Azure 资源：

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Bicep 部署脚本将完成此练习所需的 Azure 服务预配到你的资源组中。 部署的资源是 Azure Database for PostgreSQL - 灵活服务器。 bicep 脚本还会创建一个数据库 - 该数据库可在命令行上配置为参数。

    部署需要数分钟才能完成。 你可以从 Cloud Shell 监视它，也可以导航到上述创建的资源组的“**部署**”页面，在那里观察部署进度。

8. 完成资源部署后，关闭 Cloud Shell 窗格。

### 排查部署错误

运行 Bicep 部署脚本时可能会遇到一些错误。 最常见的消息和解决它们的步骤包括：

- 如果你以前为此学习路径运行过 Bicep 部署脚本并随后删除了资源，如果在删除资源后 48 小时内尝试重新运行该脚本，可能会收到如下所示的错误消息：

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is '4e87a33d-a0ac-4aec-88d8-177b04c1d752'. See inner errors for details."}
    
    Inner Errors:
    {"code": "FlagMustBeSetForRestore", "message": "An existing resource with ID '/subscriptions/{subscriptionId}/resourceGroups/rg-learn-postgresql-ai-eastus/providers/Microsoft.CognitiveServices/accounts/{accountName}' has been soft-deleted. To restore the resource, you must specify 'restore' to be 'true' in the property. If you don't want to restore existing resource, please purge it first."}
    ```

    如果收到此消息，请修改上述 `azure deployment group create` 命令，将 `restore` 参数设置为 `true`，然后重新运行。

- 如果所选区域受限于预配特定资源，则必须将 `REGION` 变量设置为其他位置，然后重新运行命令以创建资源组并运行 Bicep 部署脚本。

    ```bash
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- 如果脚本由于接受负责任的 AI 协议的要求而无法创建 AI 资源，则可能会遇到以下错误：在这种情况下，使用 Azure 门户用户界面创建 Azure AI 服务资源，然后重新运行部署脚本。

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## 在继续之前

确保已完成以下操作：

1. 你已安装并启动 Azure Database for PostgreSQL 灵活服务器。 这应该由以前的 Bicep 脚本安装。
1. 你已经从 [PostgreSQL 实验室](https://github.com/MicrosoftLearning/mslearn-postgresql.git)克隆了实验室脚本。 如果尚未执行此操作，请在本地克隆存储库：
    1. 打开命令行/终端。
    1. 运行以下命令：
       ```bash
       md .\DP3021Lab
       git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git .\DP3021Lab
       ```
       > 注意
       > 
       > 如果未安装 **git** ， [请下载并安装 ***git*** 应用](https://git-scm.com/download) ，然后再次尝试运行上述命令。
1. 你已安装 Azure Data Studio。 如果尚未 [下载并安装 ***Azure Data Studio***](https://go.microsoft.com/fwlink/?linkid=2282284)。
1. 在 Azure Data Studio 中安装 **PostgreSQL** 扩展。
1. 打开 Azure Data Studio 并连接到 Bicep 脚本创建的 Azure Database for PostgreSQL 灵活服务器。 输入用户名 **pgAdmin** 和之前创建的**随机管理员密码**。
1. 如果尚未创建 zoodb 数据库，请选择“**文件**”、“**打开文件**”，然后导航到保存脚本的文件夹。 选择 **../Allfiles/Labs/02/Lab2_ZooDb.sql** 和“**打开**”。
   1. 突出显示 **DROP** 和 **CREATE** 语句并运行它们。
   1. 在屏幕顶部，使用下拉箭头显示服务器上的数据库，包括 zoodb 和系统数据库。 选择“ **zoodb** ”数据库。
   1. 突出显示 **Create tables**、**Create foreign keys** 和 **Populate tables** 部分并运行它们。
   1. 突出显示脚本末尾的 3 个 **SELECT** 语句，并运行它们以验证表是否已创建和填充。

## 练习 EXPLAIN ANALYZE

1. 在 [Azure 门户](https://portal.azure.com)中，导航到 Azure Database for PostgreSQL 灵活服务器。 检查服务器是否已启动或在必要时重启服务器。
1. 打开 Azure Data Studio 并连接到 Azure Database for PostgreSQL 灵活服务器。
1. 选择“文件”、“打开文件”，并导航到保存脚本的文件夹。 打开 **../Allfiles/Labs/03/Lab3_RepopulateZoo.sql**。 如有必要，请重新连接到服务器。
1. 选择“运行”以执行查询。 这会重新填充 zoodb 数据库。
1. 选择“文件”，“**打开文件**”，然后选择 **../Allfiles/Labs/03/Lab3_explain.sql**。
1. 在实验室文件的 **1. Investigate EXPLAIN ANALYZE** 部分，突出显示并分别运行语句 A 和语句 B。
    1. 哪个语句更新了数据库，为什么？
    1. 计划语句 A 需要多少毫秒？
    1. 语句 B 的执行时间是多少？

## 练习 EXPLAIN

1. 在实验室文件的 **2.Investigate EXPLAIN** 部分，突出显示并运行该语句。
    1. 使用了哪种排序键，为什么？
1. 在实验室文件的“**3.调查 EXPLAIN 选项**”部分，突出显示并分别运行每个语句。 比较每个选项的查询计划统计信息。

## 清理

1. 删除在本练习中创建的资源组，以避免产生不必要的 Azure 成本。
1. 如果需要，请删除 **.\DP3021Lab** 文件夹。

