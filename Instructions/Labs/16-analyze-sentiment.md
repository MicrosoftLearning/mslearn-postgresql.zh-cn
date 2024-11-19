---
lab:
  title: 分析情绪
  module: Perform Sentiment Analysis and Opinion Mining using Azure Database for PostgreSQL
---

# 分析情绪

作为你为 Margie's Travel 构建的 AI 支持应用的一部分，你希望为用户提供有关单个评论的情绪信息，以及给定租赁物业的所有评论的整体情绪信息。 为此，请使用 Azure Database for PostgreSQL 灵活服务器中的 `azure_ai` 扩展将情绪分析功能集成到数据库中。

## 开始之前

你需要一个具有管理权限的 [Azure 订阅](https://azure.microsoft.com/free)，并且必须在该订阅中获得 Azure OpenAI 访问许可。 如果需要 Azure OpenAI 访问权限，请在 [Azure OpenAI 受限访问](https://learn.microsoft.com/legal/cognitive-services/openai/limited-access)页进行申请。

### 在你的 Azure 订阅上部署资源

此步骤指导你使用 Azure Cloud Shell 中的 Azure CLI 命令创建资源组并运行 Bicep 脚本，以将完成此练习所需的 Azure 服务部署到你的 Azure 订阅中。

1. 打开 web 浏览器，导航到 [Azure 门户](https://portal.azure.com/)。

2. 选择 Azure 门户工具栏中的“ **Cloud Shell** ”图标，以打开浏览器窗口底部的新“ [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) ”窗格。

    ![Azure 工具栏屏幕截图，Cloud Shell 图标用红框突出显示。](media/11-portal-toolbar-cloud-shell.png)

    如果出现提示，请选择打开 *Bash* shell 所需的选项。 如果以前使用过 *PowerShell* 控制台，请将其切换到 *Bash* shell。

3. 在 Cloud Shell 提示符下，输入以下内容以克隆包含练习资源的 GitHub 存储库：

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

4. 接下来，运行三个命令来定义变量，以在使用 Azure CLI 命令创建 Azure 资源时减少冗余键入。 变量表示要分配给资源组的名称（`RG_NAME`）、要将资源部署到的 Azure 区域（`REGION`）和随机生成的 PostgreSQL 管理员登录密码（`ADMIN_PASSWORD`）。

    在第一个命令中，分配给相应变量的区域是 `eastus`，但你也可以将其替换为首选位置。 但是，如果替换默认值，则必须选择另一个 [支持抽象摘要的 Azure 区域](https://learn.microsoft.com/azure/ai-services/language-service/summarization/region-support) ，以确保可以完成此学习路径中模块中的所有任务。

    ```bash
    REGION=eastus
    ```

    以下命令分配要用于资源组的名称，该资源组将容纳本练习中使用的所有资源。 分配给相应变量的资源组名称是 `rg-learn-postgresql-ai-$REGION`，其中 `$REGION` 是上文指定的位置。 但是，你可以将它更改为符合偏好的任何其他资源组名称。

    ```bash
    RG_NAME=rg-learn-postgresql-ai-$REGION
    ```

    最后一个命令随机生成 PostgreSQL 管理员登录的密码。 **请确保将其复制** 到安全位置，以便稍后连接到 PostgreSQL 灵活服务器。

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
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy.bicep" --parameters restore=false adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Bicep 部署脚本将完成此练习所需的 Azure 服务预配到你的资源组中。 部署的资源包括 Azure Database for PostgreSQL 灵活服务器、Azure OpenAI 和 Azure AI 语言服务。 Bicep 脚本还执行一些配置步骤，例如将 `azure_ai` 和 `vector` 扩展添加到 PostgreSQL 服务器的_允许列表_（通过 azure.extensions 服务器参数）、在服务器上创建名为 `rentals` 的数据库，以及使用 `text-embedding-ada-002` 模型将名为 `embedding` 的部署添加到 Azure OpenAI 服务。 请注意，Bicep 文件由此学习路径中的所有模块共享，因此在某些练习中只能使用某些已部署的资源。

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

## 在 Azure Cloud Shell 中使用 psql 连接到数据库

在此任务中，你将使用 [psql 命令行实用工具](https://www.postgresql.org/docs/current/app-psql.html)从 [Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 连接到 Azure Database for PostgreSQL 服务器上的 `rentals` 数据库。

1. 在 [Azure 门户](https://portal.azure.com/)中，导航到新创建的 Azure Database for PostgreSQL 灵活服务器实例。

2. 在资源菜单中的“**设置**”下，选择“**数据库**”为 `rentals` 数据库选择“**连接**”。

    ![Azure Database for PostgreSQL 数据库页的屏幕截图。 用于租赁数据库的数据库和连接以红色框突出显示。](media/17-postgresql-rentals-database-connect.png)

3. 在 Cloud Shell 中的“用户 pgAdmin 密码”提示符下，输入随机生成的 **pgAdmin** 登录密码。

    登录后，将显示 `rentals` 数据库的 `psql` 提示。

## 在数据库中填充示例数据

在使用 `azure_ai` 扩展分析租赁属性评审的情绪之前，必须将示例数据添加到数据库。 向 `rentals` 数据库添加一个表，并使用客户评论填充该表，以便你拥有要对其执行情绪分析的数据。

1. 运行以下命令创建一个名为 `reviews` 的表，用于存储客户提交的属性评审：

    ```sql
    DROP TABLE IF EXISTS reviews;

    CREATE TABLE reviews (
        id int,
        listing_id int, 
        date date,
        comments text
    );
    ```

2. 接下来，用 `COPY` 命令使用 CSV 文件中的数据填充表。 执行以下命令，将客户评审加载到 `reviews` 表中：

    ```sql
    \COPY reviews FROM 'mslearn-postgresql/Allfiles/Labs/Shared/reviews.csv' CSV HEADER
    ```

    命令输出应为 `COPY 354`，指示从 CSV 文件写入表中的 354 行。

## 安装和配置 `azure_ai` 扩展

在使用 `azure_ai` 扩展之前，必须先将其安装到数据库中，并将其配置为连接到 Azure AI 服务资源。 `azure_ai` 扩展让你能够将 Azure OpenAI 和 Azure AI 语言服务集成到数据库中。 要在数据库中启用该扩展，请执行以下步骤：

1. 在 `psql` 提示符处执行以下命令，验证设置环境时运行的 Bicep 部署脚本是否已成功将 `azure_ai` 扩展和 `vector` 扩展添加到服务器的“_允许列表中_”：

    ```sql
    SHOW azure.extensions;
    ```

    该命令显示服务器“_允许列表_”上的扩展列表。 如果所有内容都正确安装，则输出必须包含 `azure_ai` 和 `vector`，如下所示：

    ```sql
     azure.extensions 
    ------------------
     azure_ai,vector
    ```

    在 Azure Database for PostgreSQL 灵活服务器数据库中安装和使用扩展之前，必须将其添加到服务器的“_允许列表_”中，如[如何使用 PostgreSQL 扩展](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions)中所述。

2. 现在，你已准备好使用 [CREATE EXTENSION](https://www.postgresql.org/docs/current/sql-createextension.html) 命令安装 `azure_ai` 扩展。

    ```sql
    CREATE EXTENSION IF NOT EXISTS azure_ai;
    ```

    `CREATE EXTENSION` 通过运行其脚本文件将新扩展加载到数据库中。 此脚本通常会创建新的 SQL 对象，例如函数、数据类型和架构。 如果已存在同名的扩展，则会引发错误。 添加 `IF NOT EXISTS` 允许命令在已经安装的情况下执行，而不会引发错误。

## 连接 Azure AI 服务帐户

`azure_ai` 扩展的 `azure_cognitive` 架构中包含的 Azure AI 服务集成提供了一套丰富的 AI 语言功能，可直接从数据库使用。 情绪分析功能通过 [Azure AI 语言服务](https://learn.microsoft.com/azure/ai-services/language-service/overview)启用。

1. 若要使用 `azure_ai` 成功调用 Azure AI 语言服务，必须向扩展提供其终结点和密钥。 使用打开 Cloud Shell 的同一浏览器选项卡，导航到 [Azure 门户](https://portal.azure.com/)中的语言服务资源，并从左侧导航菜单中选择“**资源管理**”下的“**密钥和终结点**”项。

    ![显示 Azure 语言服务的“密钥和终结点”页的屏幕截图，其中“密钥 1”和“终结点复制”按钮用红色框突出显示。](media/16-azure-language-service-keys-endpoints.png)

    > [!Note]
    >
    > 如果在安装上述 `azure_ai` 扩展时收到消息 `NOTICE: extension "azure_ai" already exists, skipping CREATE EXTENSION`，并且以前使用语言服务终结点和密钥配置了该扩展，则可以使用 `azure_ai.get_setting()` 函数来确认这些设置是否正确，如果正确就跳过步骤 2。

2. 复制终结点并访问密钥值，然后在以下命令中，将 `{endpoint}` 和 `{api-key}` 令牌替换为从 Azure 门户复制的值。 从 Cloud Shell 中的 `psql` 命令提示符运行命令，将值添加到 `azure_ai.settings` 表中。

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '{api-key}');
    ```

## 查看扩展的分析情绪功能

在此任务中，你将使用 `azure_cognitive.analyze_sentiment()` 函数来评估出租属性列表的评价。

1. 在本练习的其余部分，你将专门在 Cloud Shell 中工作，因此通过选择 Cloud Shell 窗格右上角的“**最大化**”按钮来展开浏览器窗口中的窗格可能会很有帮助。

    ![Azure Cloud Shell 窗格的屏幕截图，“最大化”按钮用红框突出显示。](media/16-azure-cloud-shell-pane-maximize.png)

2. 在 Cloud Shell 中使用 `psql` 时，为查询结果启用扩展显示可能会有所帮助，因为它提高了后续命令输出的可读性。 执行以下命令以允许自动应用扩展显示。

    ```sql
    \x auto
    ```

3. `azure_ai` 扩展的情绪分析功能可在 `azure_cognitive` 架构中找到。 使用 `analyze_sentiment()` 函数。 使用 [`\df`meta-command](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) 来检查函数，运行：

    ```sql
    \df azure_cognitive.analyze_sentiment
    ```

    元命令输出显示函数的架构、名称、结果数据类型和参数。 此信息可帮助你了解如何与查询中的函数交互。

    输出显示 `analyze_sentiment()` 函数的三个重载，使你能够查看其差异。 输出中的 `Argument data types` 属性显示了三个函数重载所期望的参数列表：

    | 参数 | 类型 | 默认 | 说明 |
    | -------- | ---- | ------- | ----------- |
    | text | `text` 或 `text[]` || 应分析其情绪的文本。 |
    | language_text | `text` 或 `text[]` || 语言代码（或语言代码数组）代表要进行情绪分析的文本的语言。 查看[支持的语言列表](https://learn.microsoft.com/azure/ai-services/language-service/sentiment-opinion-mining/language-support)以检索所需的语言代码。 |
    | batch_size | `integer` | 10 | 仅适用于需要 `text[]` 输入的两个重载。 指定一次要处理的记录数。 |
    | disable_service_logs | `boolean` | false | 用于指示是否关闭服务日志的标志。 |
    | timeout_ms | `integer` | Null | 超时的毫秒数，超过该时间后操作将停止。 |
    | throw_on_error | `boolean` | 是 | 指示函数是否应在出错时引发异常，从而导致包装事务回滚的标志。 |
    | max_attempts | `integer` | 1 | 在发生故障时尝试重新调用 Azure AI 服务的次数。 |
    | retry_delay_ms | `integer` | 1000 | 尝试重新调用 Azure AI 服务终结点之前等待的时间（以毫秒为单位）。 |

4. 还必须了解函数返回的数据类型的结构，以便正确处理查询中的输出。 运行以下命令以检查 `sentiment_analysis_result` 类型：

    ```sql
    \dT+ azure_cognitive.sentiment_analysis_result
    ```

5. 上述命令的输出显示 `sentiment_analysis_result` 类型是 `tuple`。 可以通过运行以下命令来查看 `sentiment_analysis_result` 类型中包含的列，从而进一步了解 `tuple` 的结构：

    ```sql
    \d+ azure_cognitive.sentiment_analysis_result
    ```

    该命令的输出应如下所示：

    ```sql
                     Composite type "azure_cognitive.sentiment_analysis_result"
         Column     |     Type         | Collation | Nullable | Default | Storage  | Description 
    ----------------+------------------+-----------+----------+---------+----------+-------------
     sentiment      | text             |           |          |         | extended | 
     positive_score | double precision |           |          |         | plain    | 
     neutral_score  | double precision |           |          |         | plain    | 
     negative_score | double precision |           |          |         | plain    |
    ```

    `azure_cognitive.sentiment_analysis_result` 是一个复合类型，包含输入文本的情绪预测。 它包括情绪（可以是积极、消极、中立或混合）以及文本中发现的积极、中立和消极方面的分数。 分数表示为介于 0 和 1 之间的实数。 例如，在（中性，0.26、0.64、0.09）中，情绪是中性的，其中正面分数为 0.26，中性为 0.64，负面分数 为0.09。

## 分析评论的情绪

1. 在查看了 `analyze_sentiment()` 函数及其返回的 `sentiment_analysis_result`之后，让我们来使用该函数。 执行以下简单查询，对 `reviews` 表中的一些注释进行情绪分析：

    ```sql
    SELECT
        id,
        azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment
    FROM reviews
    WHERE id <= 10
    ORDER BY id;
    ```

    在分析的两条记录中，注意输出中的 `sentiment` 值，`(mixed,0.71,0.09,0.2)` 和 `(positive,0.99,0.01,0)`。 这些代表上述查询中 `sentiment_analysis_result` 函数返回的 `analyze_sentiment()`。 分析是在 `reviews` 表中的 `comments` 字段上执行的。

    > [!Note]
    >
    > 使用 `analyze_sentiment()` 内联函数可以快速分析查询中的文本情绪。 虽然这对少量记录很有效，但它可能不适合分析大量记录的情绪或更新可能包含数万条评论或更多评论的表中的所有记录。

2. 对于较长的评论，另一种有用的方法是分析其中每个句子的情绪。 为此，请使用 `analyze_sentiment()` 函数的重载，它可以接受文本数组。

    ```sql
    SELECT
        azure_cognitive.analyze_sentiment(ARRAY_REMOVE(STRING_TO_ARRAY(comments, '.'), ''), 'en') AS sentence_sentiments
    FROM reviews
    WHERE id = 1;
    ```

    在上述查询中，使用了 PostgreSQL 的 `STRING_TO_ARRAY` 函数。 此外，`ARRAY_REMOVE` 函数用于删除任何为空字符串的数组元素，因为这些元素会导致 `analyze_sentiment()` 函数出错。

    通过查询的输出，可以更好地了解分配给整体评论的 `mixed` 情绪。 这些句子混合了积极、中性和消极的情绪。

3. 前两个查询直接从查询中返回 `sentiment_analysis_result`。 不过，你可能希望检索 `sentiment_analysis_result``tuple` 内部的基础值。 执行以下查询，查找压倒性的积极评论并将情绪成分提取到单个字段中：

    ```sql
    WITH cte AS (
        SELECT id, comments, azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment FROM reviews
    )
    SELECT
        id,
        (sentiment).sentiment,
        (sentiment).positive_score,
        (sentiment).neutral_score,
        (sentiment).negative_score,
        comments
    FROM cte
    WHERE (sentiment).positive_score > 0.98
    LIMIT 5;
    ```

    上述查询使用通用表表达式或 CTE 来获取 `reviews` 表中所有记录的情绪分数。 然后从 CTE 返回的 `sentiment_analysis_result` 中选择 `sentiment` 复合类型列，以便从 `tuple.` 中提取单个值

## 在“评论表”中存储情绪

对于你为 Margie's Travel 构建的租赁物业推荐系统，你希望将情绪评级存储在数据库中，这样每次请求情绪评估时就无需进行调用，也不会产生费用。 对于少量记录或近乎实时的数据分析来说，执行即时情绪分析非常有用。 不过，将情绪数据添加到数据库中以供应用程序使用，对于存储的评论来说是很有意义的。 为此，你需要更改 `reviews` 表以添加用于存储情绪评估的列以及积极、中立和消极分数。

1. 运行以下查询来更新 `reviews` 表，以便它可以存储情绪详细信息：

    ```sql
    ALTER TABLE reviews
    ADD COLUMN sentiment varchar(10),
    ADD COLUMN positive_score numeric,
    ADD COLUMN neutral_score numeric,
    ADD COLUMN negative_score numeric;
    ```

2. 接下来，你希望使用情绪值和相关分数更新 `reviews` 表中的现有记录。

    ```sql
    WITH cte AS (
        SELECT id, azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment FROM reviews
    )
    UPDATE reviews AS r
    SET
        sentiment = (cte.sentiment).sentiment,
        positive_score = (cte.sentiment).positive_score,
        neutral_score = (cte.sentiment).neutral_score,
        negative_score = (cte.sentiment).negative_score
    FROM cte
    WHERE r.id = cte.id;
    ```

    执行此查询需要很长时间，因为表中每条评论的注释都会单独发送到语言服务的终结点进行分析。 处理大量记录时，批量发送记录更高效。

3. 让我们运行以下查询来执行相同的更新操作，但这次是以 10 个批次（这是允许的最大批次大小）发送 `reviews` 表中的注释，并评估性能差异。

    ```sql
    WITH cte AS (
        SELECT azure_cognitive.analyze_sentiment(ARRAY(SELECT comments FROM reviews ORDER BY id), 'en', batch_size => 10) as sentiments
    ),
    sentiment_cte AS (
        SELECT
            ROW_NUMBER() OVER () AS id,
            sentiments AS sentiment
        FROM cte
    )
    UPDATE reviews AS r
    SET
        sentiment = (sentiment_cte.sentiment).sentiment,
        positive_score = (sentiment_cte.sentiment).positive_score,
        neutral_score = (sentiment_cte.sentiment).neutral_score,
        negative_score = (sentiment_cte.sentiment).negative_score
    FROM sentiment_cte
    WHERE r.id = sentiment_cte.id;
    ```

    虽然此查询比较复杂，但使用两个 CTE，性能更高。 在此查询中，第一个 CTE 将分析审阅注释批的情绪，第二个 CTE 根据每一行的序号位置和 “sentiment_analysis_result”，将生成的 `sentiment_analysis_results` 表提取到包含 `id` 的新表中。 然后，可以在更新语句中使用第二个 CTE 将值写入数据库中。

4. 接下来，运行一个查询来观察更新结果，搜索具有**负面**情绪的评论，从最负面的开始。

    ```sql
    SELECT
        id,
        negative_score,
        comments
    FROM reviews
    WHERE sentiment = 'negative'
    ORDER BY negative_score DESC;
    ```

## 清理

完成本练习后，请删除创建的 Azure 资源。 你需要为配置的容量（而不是数据库的使用量）付费。 按照这些说明删除资源组和为此实验室创建的所有资源。

1. 打开 Web 浏览器并导航到 [Azure 门户](https://portal.azure.com/)，然后在主页上选择 Azure 服务下的**资源组**。

    ![Azure 门户中 Azure 服务下红框突出显示的资源组的屏幕截图。](media/16-azure-portal-home-azure-services-resource-groups.png)

2. 在任何字段搜索框的筛选器中，输入为此实验室创建的资源组的名称，然后从列表中选择你的资源组。

3. 在资源组的“概述”页面中，选择“删除资源组” 。

    ![资源组的“概述”边栏选项卡的屏幕截图，其中“删除资源组”按钮以红框突出显示。](media/16-resource-group-delete.png)

4. 在确认对话框中，输入要删除的资源组名称进行确认，然后选择“**删除**”。
