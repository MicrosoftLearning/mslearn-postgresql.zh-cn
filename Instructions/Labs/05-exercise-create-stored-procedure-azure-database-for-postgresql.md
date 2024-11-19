---
lab:
  title: 在 Azure Database for PostgreSQL 中创建存储过程
  module: Procedures and functions in PostgreSQL
---

# 在 Azure Database for PostgreSQL 中创建存储过程

在本练习中，你将创建一个存储过程。

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

2. 选择 Azure 门户工具栏中的“**Cloud Shell**”图标，以打开浏览器窗口底部的新“[Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview)”窗格。

    ![Azure 工具栏屏幕截图，Cloud Shell 图标用红框突出显示。](media/05-portal-toolbar-cloud-shell.png)

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
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- 如果脚本由于接受负责任的 AI 协议的要求而无法创建 AI 资源，则可能会遇到以下错误：在这种情况下，使用 Azure 门户用户界面创建 Azure AI 服务资源，然后重新运行部署脚本。

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## 本地克隆 GitHub 存储库

确保已从 [PostgreSQL 实验室](https://github.com/MicrosoftLearning/mslearn-postgresql.git)克隆实验室脚本。 如果尚未执行此操作，请在本地克隆存储库：

1. 打开命令行/终端。
1. 运行以下命令：
    ```bash
    md .\DP3021Lab
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git .\DP3021Lab
    ```
    > 注意
    > 
    > 如果未安装 **git** ， [请下载并安装 ***git*** 应用](https://git-scm.com/download) ，然后再次尝试运行上述命令。

## 安装 Azure Data Studio

如果你没有安装 Azure Data Studio：

1. 在浏览器中，导航到[下载并安装 Azure Data Studio](/sql/azure-data-studio/download-azure-data-studio)，然后在 Windows 平台下，选择“用户安装程序(推荐)”。 可执行文件将下载到“下载”文件夹。
1. 选择“打开文件”。
1. 此时将显示“许可协议”。 阅读并接受协议，然后选择“下一步”。
1. 在“选择其他任务”中，选择“添加到 PATH”以及所需的任何其他添加内容。 选择**下一步**。
1. 此时将显示“准备安装”对话框。 复查你的设置。 选择“返回”进行更改，或选择“安装”。
1. 此时将显示“正在完成 Azure Data Studio 安装向导”对话框。 选择“完成”。 Azure Data Studio 启动。

## 安装 PostgreSQL 扩展

如果你没有在 Azure Data Studio 中安装 PostgreSQL 扩展：

1. 如果 Azure Data Studio 尚未打开，请将其打开。
1. 在左侧菜单中，选择“扩展”以显示“扩展”面板。
1. 在搜索栏中，输入“PostgreSQL”。 将显示 Azure Data Studio 的 PostgreSQL 扩展图标。
1. 选择“安装”  。 扩展安装。

## 连接到 Azure Database for PostgreSQL 灵活服务器

1. 如果 Azure Data Studio 尚未打开，请将其打开。
1. 从左侧菜单中，选择“连接”。
1. 选择“新建连接”。
1. 在“连接详细信息”下的“连接类型”中，从下拉列表中选择“PostgreSQL”。
1. 在“服务器名称”中，输入 Azure 门户上显示的完整服务器名称。
1. 在“身份验证类型”中，保留密码。
1. 在“用户名和密码”中，输入用户名 **pgAdmin** 和上面创建的**随机管理员密码**。
1. 选择 [ x ] 记住密码。
1. 其余字段是可选的。
1. 选择“连接” 。 你已连接到 Azure Database for PostgreSQL 服务器。
1. 将显示服务器数据库的列表。 这包括系统数据库和用户数据库。
1. 如果尚未创建 zoodb 数据库，请选择“**文件**”、“**打开文件**”，然后导航到保存脚本的文件夹。 选择 **../Allfiles/Labs/02/Lab2_ZooDb.sql** 和“**打开**”。
   1. 突出显示 **DROP** 和 **CREATE** 语句并运行它们。
   1. 在屏幕顶部，使用下拉箭头显示服务器上的数据库，包括 zoodb 和系统数据库。 选择 **zoodb** 数据库。
   1. 突出显示“**Create tables**”、“**Create foreign keys**”和“**Populate tables**”部分并运行它们。
   1. 突出显示脚本末尾的 3 个 **SELECT** 语句，并运行它们以验证表是否已创建和填充。

## 创建存储过程 repopulate_zoo()

1. 在屏幕顶部，使用下拉箭头将 zoodb 设置为当前数据库。
1. 在 Azure Data Studio 中，选择“文件”、“打开文件”，然后导航到实验室脚本。******** 选择 **../Allfiles/Labs/03/Lab3_RepopulateZoo.sql** ，然后选择“ **打开**”。 如有必要，请重新连接到服务器。
1. 突出显示“**创建存储过程**”下从“**删除过程**到 **END $$**”的部分。 运行突出显示的文本。
1. 保持 Azure Data Studio 打开并打开文件，为下一个练习做好准备。

## 创建 new_exhibit() 存储过程

1. 在屏幕顶部，使用下拉箭头将 zoodb 设置为当前数据库。
1. 在 Azure Data Studio 中，选择“文件”、“打开文件”，然后导航到实验室脚本。******** 选择 **../Allfiles/Labs/05/Lab5_StoredProcedure.sql** 然后选择 **打开**。 如有必要，请重新连接到服务器。
1. 突出显示“**创建存储过程**”下从“**删除过程**到 **END $$**”的部分。 运行突出显示的文本。 通读过程。 你将看到它声明一些输入参数，并使用它们将行插入 enclosure 表和 animal 表。
1. 保持 Azure Data Studio 打开并打开文件，为下一个练习做好准备。

## 调用存储过程

1. 突出显示“调用存储过程”下的部分。**** 运行突出显示的文本。 这会通过将值传递给输入参数来调用存储过程。
1. 突出显示并运行两个 SELECT 语句。**** 运行突出显示的文本。 可以看到，一个新行已插入到 enclosure 中，五个新行插入到了 animal 中。

## 创建和调用表值函数

1. 在 Azure Data Studio 中，选择“文件”、“打开文件”，然后导航到实验室脚本。******** 选择 **../Allfiles/Labs/05/Lab5_Table_Function.sql**，然后选择 **打开**。
1. 突出显示并运行第一个 SELECT 语句，检查是否选择了 zoodb 数据库。****
1. 突出显示并运行 repopulate_zoo() 存储过程以从干净数据开始。****
1. 突出显示并运行“创建表值函数”下的部分。**** 此函数返回名为 enclosure_summary 的表。**** 通读函数代码，了解表的填充方式。
1. 突出显示并运行两个 select 语句，每次传入不同的机箱 ID。
1. 突出显示并运行“如何使用具有 LATERAL 联接的表值函数”下的部分。**** 这将显示用于代替联接中的表名的表值函数。

## 可选练习 - 内置函数

1. 在 Azure Data Studio 中，选择“文件”、“打开文件”，然后导航到实验室脚本。******** 选择 **../Allfiles/Labs/05/Lab5_SimpleFunctions.sql** ，然后选择“**打开**”。
1. 突出显示并运行每个函数以查看其工作原理。 有关每个函数的详细信息，请参考[联机文档](https://www.postgresql.org/docs/current/functions.html)。
1. 关闭 Azure Data Studio 而不保存脚本。
1. 停止 Azure Database for PostgreSQL 服务器，以便在不使用服务器时不收取费用。

## 清理

1. 删除在本练习中创建的资源组，以避免产生不必要的 Azure 成本。
1. 如果需要，请删除 .\DP3021Lab 文件夹。

