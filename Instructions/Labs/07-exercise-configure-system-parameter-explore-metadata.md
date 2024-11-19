---
lab:
  title: 配置系统参数并使用系统目录和视图探索元数据
  module: Configure and manage Azure Database for PostgreSQL
---

# 配置系统参数并使用系统目录和视图探索元数据

在本练习中，你将查看 PostgreSQL 中的系统参数和元数据。

## 开始之前

> [!IMPORTANT]
> 你需要自己的 Azure 订阅才能完成此模块中的练习。 如果没有 Azure 订阅，可以在[使用 Azure 免费帐户在云中构建](https://azure.microsoft.com/free/)设置免费试用帐户。

## 创建练习环境

### 在你的 Azure 订阅上部署资源

此步骤指导你使用 Azure Cloud Shell 中的 Azure CLI 命令创建资源组并运行 Bicep 脚本，以将完成此练习所需的 Azure 服务部署到你的 Azure 订阅中。

> 注意
>
> 如果你要在此学习路径中执行多个模块，则可以在它们之间共享 Azure 环境。 在这种情况下，你只需完成此资源部署步骤一次。

1. 打开 web 浏览器，导航到 [Azure 门户](https://portal.azure.com/)。

2. 选择 Azure 门户工具栏中的“**Cloud Shell**”图标，以打开浏览器窗口底部的新“[Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview)”窗格。

    ![Azure 工具栏屏幕截图，Cloud Shell 图标用红框突出显示。](media/07-portal-toolbar-cloud-shell.png)

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

### 使用 Azure Data Studio 连接到数据库

1. 如果你还没有这样做，请在本地克隆 [PostgreSQL 实验室](https://github.com/MicrosoftLearning/mslearn-postgresql.git) GitHub 存储库中的实验室脚本：
    1. 打开命令行/终端。
    1. 运行以下命令：
       ```bash
       md .\DP3021Lab
       git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git .\DP3021Lab
       ```
       > 注意
       > 
       > 如果未安装 **git** ， [请下载并安装 ***git*** 应用](https://git-scm.com/download) ，然后再次尝试运行上述命令。
1. 如果尚未安装 Azure Data Studio， [下载并安装 ***Azure Data Studio***](https://go.microsoft.com/fwlink/?linkid=2282284)。
1. 如果尚未在 Azure Data Studio 中安装 **PostgreSQL** 扩展，请立即安装。
1. 打开 Azure Data Studio。
1. 选择**连接**。
1. 选择“服务器”，然后选择“新建连接”。
1. 在“连接类型”中，选择“PostgreSQL”。
1. 在“服务器名称”中，键入部署服务器时指定的值。
1. 在“**用户名**”中，键入 **pgAdmin**。
1. 在“**密码**”中，输入随机生成的 **pgAdmin** 登录密码
1. 选择“记住密码”。
1. 单击“连接”
1. 如果尚未创建 zoodb 数据库，请选择“**文件**”、“**打开文件**”，然后导航到保存脚本的文件夹。 选择 **../Allfiles/Labs/02/Lab2_ZooDb.sql** 和“**打开**”。
   1. 突出显示 **DROP** 和 **CREATE** 语句并运行它们。
   1. 在屏幕顶部，使用下拉箭头显示服务器上的数据库，包括 zoodb 和系统数据库。 选择 **zoodb** 数据库。
   1. 突出显示“**Create tables**”、“**Create foreign keys**”和“**Populate tables**”部分并运行它们。
   1. 突出显示脚本末尾的 3 个 **SELECT** 语句，并运行它们以验证表是否已创建和填充。

## 任务 1：探索 PostgreSQL 中的真空过程

1. 如果尚未打开，请打开 Azure Data Studio。
1. 在 Azure Data Studio 中，选择“文件”、“打开文件”，然后导航到实验室脚本。******** 选择 **../Allfiles/Labs/07/Lab7_vacuum.sql**，然后选择“**打开**”。 如有必要，请重新连接到服务器。
1. 从数据库下拉菜单中选择“ **zoodb** ”数据库。
1. 突出显示并运行“检查 zoodb 数据库是否已选中”部分****。 如有必要，请使用下拉列表将 zoodb 设置为当前数据库。
1. 突出显示并运行“显示死元组”部分****。 此查询会显示数据库中的死元组和活元组数目。 记下死元组的数目。
1. 突出显示并连续运行 **Change weight** 部分 10 次。 此查询会更新所有动物的权重列。
1. 再次运行“显示死元组”下的部分****。 记下更新完成后的死元组数目。
1. 运行“手动运行 VACUUM”下的部分以运行 vacuum 过程****。
1. 再次运行“显示死元组”下的部分****。 记下 vacuum 过程运行后死元组的数量。

## 任务 2：配置 autovacuum 服务器参数

1. 在 Azure 门户中，导航到 Azure Database for PostgreSQL 灵活服务器。
1. 在“设置”下，选择“服务器参数”。
1. 在搜索栏中，输入 **`vacuum`**。 查找以下参数，并按如下所示更改值：
    1. autovacuum = ON（默认为 ON）
    1. autovacuum_vacuum_scale_factor = 0.1
    1. autovacuum_vacuum_threshold = 50

    这相当于在表中有 10% 的行被标记为要删除时运行 autovacuum 过程，或者在任意一个表中更新或删除了 50 行时运行 autovacuum 过程。

1. 选择“保存”。 服务器已重启。

## 任务 3：在 Azure 门户中查看 PostgreSQL 元数据

1. 导航到 [Azure 门户](https://portal.azure.com)并登录。
1. 搜索 **Azure Database for PostgreSQL** 并选择它。
1. 选择为此练习创建的 Azure Database for PostgreSQL 灵活服务器。
1. 在“监视”中，选择“指标”********。
1. 选择“指标”并选择“CPU 百分比”。********
1. 请注意，可以查看有关数据库的各种指标。

## 任务 4：查看系统目录表中的数据

1. 切换到 Azure Data Studio。
1. 在“服务”中，选择 PostgreSQL 服务器，等待连接建立，并在服务器上显示绿色圆圈。****
1. 右键单击服务器并选择“新建查询”。****
1. 键入以下 SQL 并选择“运行”：****

    ```sql
    SELECT datname, xact_commit, xact_rollback FROM pg_stat_database;
    ```

1. 请注意，可以查看每个数据库的提交和回滚。

## 使用系统视图查看复杂元数据查询

1. 右键单击服务器并选择“新建查询”。****
1. 键入以下 SQL 并选择“运行”：****

    ```sql
    SELECT *
    FROM pg_catalog.pg_stats;
    ```

1. 请注意，可以查看大量的统计信息。
1. 通过使用系统视图，可以减少需要写入的 SQL 的复杂性。 如果不使用 pg_stats 视图，则上一个查询需要以下代码：****

    ```sql
    SELECT n.nspname AS schemaname,
    c.relname AS tablename,
    a.attname,
    s.stainherit AS inherited,
    s.stanullfrac AS null_frac,
    s.stawidth AS avg_width,
    s.stadistinct AS n_distinct,
        CASE
            WHEN s.stakind1 = 1 THEN s.stavalues1
            WHEN s.stakind2 = 1 THEN s.stavalues2
            WHEN s.stakind3 = 1 THEN s.stavalues3
            WHEN s.stakind4 = 1 THEN s.stavalues4
            WHEN s.stakind5 = 1 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS most_common_vals,
        CASE
            WHEN s.stakind1 = 1 THEN s.stanumbers1
            WHEN s.stakind2 = 1 THEN s.stanumbers2
            WHEN s.stakind3 = 1 THEN s.stanumbers3
            WHEN s.stakind4 = 1 THEN s.stanumbers4
            WHEN s.stakind5 = 1 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS most_common_freqs,
        CASE
            WHEN s.stakind1 = 2 THEN s.stavalues1
            WHEN s.stakind2 = 2 THEN s.stavalues2
            WHEN s.stakind3 = 2 THEN s.stavalues3
            WHEN s.stakind4 = 2 THEN s.stavalues4
            WHEN s.stakind5 = 2 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS histogram_bounds,
        CASE
            WHEN s.stakind1 = 3 THEN s.stanumbers1[1]
            WHEN s.stakind2 = 3 THEN s.stanumbers2[1]
            WHEN s.stakind3 = 3 THEN s.stanumbers3[1]
            WHEN s.stakind4 = 3 THEN s.stanumbers4[1]
            WHEN s.stakind5 = 3 THEN s.stanumbers5[1]
            ELSE NULL::real
        END AS correlation,
        CASE
            WHEN s.stakind1 = 4 THEN s.stavalues1
            WHEN s.stakind2 = 4 THEN s.stavalues2
            WHEN s.stakind3 = 4 THEN s.stavalues3
            WHEN s.stakind4 = 4 THEN s.stavalues4
            WHEN s.stakind5 = 4 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS most_common_elems,
        CASE
            WHEN s.stakind1 = 4 THEN s.stanumbers1
            WHEN s.stakind2 = 4 THEN s.stanumbers2
            WHEN s.stakind3 = 4 THEN s.stanumbers3
            WHEN s.stakind4 = 4 THEN s.stanumbers4
            WHEN s.stakind5 = 4 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS most_common_elem_freqs,
        CASE
            WHEN s.stakind1 = 5 THEN s.stanumbers1
            WHEN s.stakind2 = 5 THEN s.stanumbers2
            WHEN s.stakind3 = 5 THEN s.stanumbers3
            WHEN s.stakind4 = 5 THEN s.stanumbers4
            WHEN s.stakind5 = 5 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS elem_count_histogram
    FROM pg_statistic s
     JOIN pg_class c ON c.oid = s.starelid
     JOIN pg_attribute a ON c.oid = a.attrelid AND a.attnum = s.staattnum
     LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
    WHERE NOT a.attisdropped AND has_column_privilege(c.oid, a.attnum, 'select'::text) AND (c.relrowsecurity = false OR NOT row_security_active(c.oid));
    ```

## 练习清理

1. 我们在本练习中部署的 Azure Database for PostgreSQL 将产生费用，你可以在本练习后删除服务器。 或者，你可以删除 **rg-learn-work-with-postgresql-eastus** 资源组，以移除在本练习中部署的所有资源。
1. 如果需要，请删除 .\DP3021Lab 文件夹。
