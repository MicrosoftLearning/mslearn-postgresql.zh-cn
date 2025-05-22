---
lab:
  title: 执行 EXPLAIN 语句
  module: Understand PostgreSQL query processing
---

# 执行 EXPLAIN 语句

在本练习中，你将了解 EXPLAIN 函数，以及它如何显示 PostgreSQL 规划器为提供的语句生成的执行计划。

## 开始之前

需要有自己的 Azure 订阅才能完成本练习。 如果还没有 Azure 订阅，可以创建一个 [Azure 免费试用版](https://azure.microsoft.com/free)。

此外，需要在计算机上安装以下软件：

- Visual Studio Code。
- Microsoft Postgres Visual Studio Code 扩展。
- Azure CLI。
- Git。

## 创建练习环境

在本练习及以后的练习中，将使用 Bicep 脚本将 Azure Database for PostgreSQL - 灵活服务器和其他资源部署到 Azure 订阅中。 Bicep 脚本位于之前克隆的 GitHub 存储库的 `/Allfiles/Labs/Shared` 文件夹中。

### 下载并安装 Visual Studio Code 和 PostgreSQL 扩展

如果未安装 Visual Studio Code：

1. 在浏览器中，导航到“[下载 Visual Studio Code](https://code.visualstudio.com/download)”，并选择适合操作系统的版本。

1. 按照操作系统的安装说明进行操作。

1. 打开 Visual Studio Code。

1. 在左侧菜单中，选择“扩展”以显示“扩展”面板。

1. 在搜索栏中，输入“PostgreSQL”。 将显示“Visual Studio Code 的 PostgreSQL 扩展”图标。 请确保选择 Microsoft 的产品。

1. 选择“安装”  。 扩展安装。

### 下载并安装 Azure CLI 和 Git

如果没有安装 Azure CLI 或 Git：

1. 在浏览器中，导航到“[安装 Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)”，并按照操作系统的说明进行操作。

1. 在浏览器中，导航到“[下载并安装 Git](https://git-scm.com/downloads)”，并按照操作系统的说明进行操作。

### 下载练习文件

如果已克隆包含练习文件的 GitHub 存储库，*请跳过下载练习文件*。

要下载练习文件，请将包含练习文件的 GitHub 存储库克隆到本地计算机。 存储库包含完成本练习所需的所有脚本和资源。

1. 打开 Visual Studio Code（如果尚未打开）。

1. 选择“**显示所有命令**” (Ctrl+Shift+P)，以打开命令面板。

1. 在命令面板中，搜索并选择“**Git：克隆**”。

1. 在命令面板中，输入以下内容以克隆包含练习资源的 GitHub 存储库，然后按 **Enter**：

    ```bash
    https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

1. 按照提示选择要克隆存储库的文件夹。 存储库克隆到所选位置中命名为 `mslearn-postgresql` 的文件夹。

1. 当系统询问你是否要打开克隆的存储库时，选择“打开”。 在 Visual Studio Code 中打开存储库。

### 在你的 Azure 订阅上部署资源

如果已安装 Azure 资源，*请跳过部署资源*。

此步骤指导你使用 Visual Studio Code 中的 Azure CLI 命令创建资源组并运行 Bicep 脚本，以将完成此练习所需的 Azure 服务部署到 Azure 订阅中。

> &#128221; 如果要在此学习路径中执行多个模块，则可以在它们之间共享 Azure 环境。 在这种情况下，你只需完成此资源部署步骤一次。

1. 打开 Visual Studio Code 并打开克隆存储库的文件夹（如果尚未打开）。

1. 展开资源管理器窗格中的 **mslearn-postgresql** 文件夹。

1. 展开 **Allfiles/Labs/Shared** 文件夹。

1. 右键单击 **Allfiles/Labs/Shared** 文件夹，然后选择“**在集成终端中打开**”。 该选择会在 “Visual Studio Code” 窗口中打开一个终端窗口。

1. 默认情况下，终端可能会打开 “**Powershell**” 窗口。 在本部分实验中，请使用 **bash shell**。 **+** 图标旁边还有一个下拉箭头。 选择它后，从可用配置文件列表中选择 **Git Bash** 或 **Bash**。 该选择会打开一个新的终端窗口，使用 **bash shell**。

    > &#128221; 如有需要，你可以关闭 “**PowerShell**” 终端窗口，但这并非必要。 你可以同时打开多个终端窗口。

1. 在终端窗口中运行以下命令，登录你的 Azure 帐户：

    ```bash
    az login
    ```

    运行该命令后将打开一个新的浏览器窗口，提示你登录 Azure 帐户。 完成登录后，返回终端窗口。

1. 接下来，运行三个命令来定义变量，以在使用 Azure CLI 命令创建 Azure 资源时减少冗余键入。 这些变量分别表示要分配给资源组的名称（`RG_NAME`）、资源部署所在的 Azure 区域（`REGION`），以及 PostgreSQL 管理员登录所用的随机生成密码（`ADMIN_PASSWORD`）。

    在第一个命令中，分配给相应变量的区域是 `eastus`，但你也可以将其替换为首选位置。

    ```bash
    REGION=eastus
    ```

    以下命令用于分配资源组的名称，该资源组将包含本练习所用的所有资源。 资源组名称已分配至对应变量 `rg-learn-work-with-postgresql-$REGION`，其中`$REGION`表示你先前指定的位置。 *你也可以更改为符合个人喜好或已有的其他资源组名称。*。

    ```bash
    RG_NAME=rg-learn-work-with-postgresql-$REGION
    ```

    最后一个命令会随机生成 PostgreSQL 管理员登录密码。 请确保将其复制到安全位置，以便稍后可以使用它连接到 PostgreSQL 灵活服务器。

    ```bash
    #!/bin/bash
    
    # Define array of allowed characters explicitly
    chars=( {a..z} {A..Z} {0..9} '!' '@' '#' '$' '%' '^' '&' '*' '(' ')' '_' '+' )
    
    a=()
    for ((i = 0; i < 100; i++)); do
        rand_char=${chars[$RANDOM % ${#chars[@]}]}
        a+=("$rand_char")
    done
    
    # Join first 18 characters without delimiter
    ADMIN_PASSWORD=$(IFS=; echo "${a[*]:0:18}")
    
    echo "Your randomly generated PostgreSQL admin user's password is:"
    echo "$ADMIN_PASSWORD"
    echo "Please copy it to a safe place, as you will need it later to connect to your PostgreSQL flexible server."
    ```

1. （如果使用默认订阅，请跳过此步骤。）如果你有多个 Azure 订阅权限，且默认订阅*不是*本次练习中用于创建资源组和其他资源的订阅，请运行此命令，替换`<subscriptionName|subscriptionId>`为你希望使用的订阅名称或 ID，以设置正确的订阅。

    ```azurecli
    az account set --subscription 16b3c013-d300-468d-ac64-7eda0820b6d3
    ```

1. （如使用已有资源组，请跳过此步骤）运行以下 Azure CLI 命令以创建资源组：

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

1. 最后，使用 Azure CLI 执行 Bicep 部署脚本，在资源组中预配 Azure 资源：

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Bicep 部署脚本将完成此练习所需的 Azure 服务预配到你的资源组中。 部署的资源是 Azure Database for PostgreSQL - 灵活服务器。 bicep 脚本还会创建一个数据库 - 该数据库可在命令行上配置为参数。

    部署需要数分钟才能完成。 你可以在 bash 终端监控部署进度，或者进入之前创建的资源组的“**部署**”页面查看。

1. 脚本为 PostgreSQL 服务器创建随机名称，你可运行以下命令查询服务器名称：

    ```azurecli
    az postgres flexible-server list --query "[].{Name:name, ResourceGroup:resourceGroup, Location:location}" --output table
    ```

    请记下服务器名称，稍后本练习中需要用它连接服务器。

    > &#128221; 服务器名称也可在 Azure 门户中查看。 在 Azure 门户中，导航到**资源组**，选择之前创建的资源组。 资源组中列出了 PostgreSQL 服务器。

### 排查部署错误

运行 Bicep 部署脚本时，可能会出现若干错误。 最常见的消息和解决它们的步骤包括：

- 如果你此前运行过该学习路径的 Bicep 部署脚本并删除了资源，且在资源删除后的 48 小时内重新执行该脚本，可能会收到类似以下的错误消息：

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is '4e87a33d-a0ac-4aec-88d8-177b04c1d752'. See inner errors for details."}
    
    Inner Errors:
    {"code": "FlagMustBeSetForRestore", "message": "An existing resource with ID '/subscriptions/{subscriptionId}/resourceGroups/rg-learn-postgresql-ai-eastus/providers/Microsoft.CognitiveServices/accounts/{accountName}' has been soft-deleted. To restore the resource, you must specify 'restore' to be 'true' in the property. If you don't want to restore existing resource, please purge it first."}
    ```

    如果收到此消息，请修改之前的`azure deployment group create`命令，将`restore`参数设置为`true`，然后重新执行。

- 如果所选区域受限于预配特定资源，则必须将 `REGION` 变量设置为其他位置，然后重新运行命令以创建资源组并运行 Bicep 部署脚本。

    ```bash
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- 如果实验室需要 AI 资源，可能会出现以下错误。 当脚本因需接受负责任 AI 协议而无法创建 AI 资源时，会出现此错误。 如果是这种情况，请使用 Azure 门户用户界面创建一个 Azure AI 服务资源，然后重新运行部署脚本。

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## 连接 Visual Studio Code 中的 PostgreSQL 扩展

本部分将使用 Visual Studio Code 的 PostgreSQL 扩展连接 PostgreSQL 服务器。 使用 PostgreSQL 扩展对 PostgreSQL 服务器执行 SQL 脚本。

1. 打开 Visual Studio Code（如未打开），并打开你克隆的 GitHub 存储库所在文件夹。

1. 在左侧菜单中选择 “**PostgreSQL**” 图标。

    > &#128221; 如果未看到 PostgreSQL 图标，请点击 “**扩展**” 图标并搜索 **PostgreSQL**。 选择 Microsoft 提供的 **PostgreSQL** 扩展，并点击“安装”****。

1. 如果你已连接 PostgreSQL 服务器，请跳过此步骤。 要创建新连接：

    1. 在 **PostgreSQL** 扩展中，选择“**+ 添加连接**”，以添加新连接。

    1. 在“**新建连接**”对话框中，输入以下信息：

        - **服务器名称**：`<your-server-name>`.postgres.database.azure.com
        - 身份验证类型：密码
        - **用户名**：pgAdmin
        - **密码**：之前生成的随机密码。
        - 选中“**保存密码**”复选框。
        - **连接名称**：`<your-server-name>`

    1. 选择“**测试连接**”以测试连接。 如果连接成功，请选择“**保存并连接**”以保存连接，否则请查看连接信息，然后重试。

1. 如果尚未连接，请选择 PostgreSQL 服务器的“**连接**”。 已连接到 Azure Database for PostgreSQL 服务器。

1. 展开服务器节点及其数据库。 列出了现有数据库。

1. 如果尚未创建 zoodb 数据库，请选择“**文件**”、“**打开文件**”，然后导航到保存脚本的文件夹。 选择 **../Allfiles/Labs/02/Lab2_ZooDb.sql** 和“**打开**”。

1. 在 Visual Studio Code 右下角，确保连接为绿色。 如果不是，则应显示“**PGSQL 已断开连接**”。 选择“**PGSQL 已断开连接**”文本，然后从命令面板的列表中选择 PostgreSQL 服务器连接。 如果请求密码，请输入之前生成的密码。

1. 是时候创建数据库了。

    1. 突出显示 **DROP** 和 **CREATE** 语句并运行它们。

    1. 如果仅突出显示 **SELECT current_database()** 语句并运行它，则会看到数据库当前设置为 `postgres`。 需要将其更改为 `zoodb`。

    1. 选择菜单栏中带有“*运行*”图标的省略号，然后选择“**更改 PostgreSQL 数据库**”。 从数据库列表中选择 `zoodb`。

        > &#128221; 还可以更改查询窗格上的数据库。 可以在查询选项卡本身下记下服务器名称和数据库名称。 选择数据库名称将显示数据库列表。 从列表中选择 `zoodb` 数据库。

    1. 再次运行 **SELECT current_database()** 语句以确认数据库现在设置为 `zoodb`。

    1. 突出显示“**Create tables**”、“**Create foreign keys**”和“**Populate tables**”部分并运行它们。

    1. 突出显示脚本末尾的 3 个 **SELECT** 语句，并运行它们以验证表是否已创建和填充。

## 练习 EXPLAIN ANALYZE

在本部分中，将运行 EXPLAIN 语句，分析查询的执行计划。 EXPLAIN 语句展示了 PostgreSQL 执行查询的方式，包括估算成本和实际执行时间。

1. 在 Visual Studio Code 窗口中，依次选择“文件”****、“打开文件”****，然后导航到实验室脚本。 选择 **../Allfiles/Labs/03/Lab3_RepopulateZoo.sql** ，然后选择“ **打开**”。 如有必要，选择显示为 “**PGSQL 断开连接**”文本，然后在命令面板列表中选择你的 PostgreSQL 服务器连接以重新连接服务器。 如果请求密码，请输入之前生成的密码。

1. 在 Visual Studio Code 右下角，确保连接为绿色。 如果不是，则应显示“**PGSQL 已断开连接**”。 选择“**PGSQL 已断开连接**”文本，然后从命令面板的列表中选择 PostgreSQL 服务器连接。 如果请求密码，请输入之前生成的密码。

1. 运行 **SELECT current_database()** 语句以检查当前数据库。 验证当前连接是否已设置为 **zoodb** 数据库。 如果不是，可以将数据库更改为 **zoodb**。 要更改数据库，请选择菜单栏中带有“*运行*”图标的省略号，然后选择“**更改 PostgreSQL 数据库**”。 从数据库列表中选择 `zoodb`。 通过运行 **SELECT current_database();** 语句，验证数据库现在是否设置为 `zoodb`。

1. 选择“**运行**”以执行查询。 此查询会重新填充 zoodb 数据库。

1. 在 Visual Studio Code 窗口中，依次选择“文件”****、“打开文件”****，然后导航到实验室脚本。 选择 **../Allfiles/Labs/03/Lab3_explain.sql**，然后选择“**打开**”。 如有必要，请重新连接到服务器，方法是选择“**PGSQL 已断开连接**”文本，然后从命令面板的列表中选择 PostgreSQL 服务器连接。 如果请求密码，请输入之前生成的密码。

1. 在 Visual Studio Code 右下角，确保连接为绿色。 如果不是，则应显示“**PGSQL 已断开连接**”。 选择“**PGSQL 已断开连接**”文本，然后从命令面板的列表中选择 PostgreSQL 服务器连接。 如果请求密码，请输入之前生成的密码。

1. 运行 **SELECT current_database()** 语句以检查当前数据库。 验证当前连接是否已设置为 **zoodb** 数据库。 如果不是，可以将数据库更改为 **zoodb**。 要更改数据库，请选择菜单栏中带有“*运行*”图标的省略号，然后选择“**更改 PostgreSQL 数据库**”。 从数据库列表中选择 `zoodb`。 通过运行 **SELECT current_database();** 语句，验证数据库现在是否设置为 `zoodb`。

1. 在实验室文件的 **1. Investigate EXPLAIN ANALYZE** 部分，突出显示并分别运行语句 A 和语句 B。

    1. 哪个语句更新了数据库，为什么？

    1. 计划语句 A 需要多少毫秒？

    1. 语句 B 的执行时间是多少？

## 练习 EXPLAIN

在本部分中，将运行 EXPLAIN 语句，分析查询的执行计划。 EXPLAIN 语句展示了 PostgreSQL 执行查询的方式，包括估算成本和实际执行时间。

1. 在实验室文件的 **2.Investigate EXPLAIN** 部分，突出显示并运行该语句。

    使用了哪种排序键，为什么？

1. 在实验室文件的“**3.调查 EXPLAIN 选项**”部分，突出显示并分别运行每个语句。 比较每个选项的查询计划统计信息。

## 清理

1. 如果不再需要此 PostgreSQL 服务器进行其他练习，若要避免产生不必要的 Azure 成本，请删除在本练习中创建的资源组。

1. 如果想让 PostgreSQL 服务器继续运行，可以保持其开启状态。 如果不想让它一直运行，可以在 bash 终端停止服务器，避免产生不必要的费用。 运行以下命令以停止服务器：

    ```azurecli
    az postgres flexible-server stop --name <your-server-name> --resource-group $RG_NAME
    ```

    将 `<your-server-name>` 替换为你的 PostgreSQL 服务器名称。

    > &#128221; 你也可以从 Azure 门户停止服务器。 在 Azure 门户中，导航到**资源组**，选择之前创建的资源组。 选择 PostgreSQL 服务器，然后从菜单中选择“停止”****。

1. 如果需要，请删除之前克隆的 Git 存储库。

你已成功完成此练习。 你学习了如何使用 EXPLAIN 语句分析查询的执行计划。 此外，你还学习了如何使用 EXPLAIN ANALYZE 语句结合实际执行统计信息分析查询计划。
