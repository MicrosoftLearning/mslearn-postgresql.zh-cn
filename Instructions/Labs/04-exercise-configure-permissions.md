---
lab:
  title: 在 Azure Database for PostgreSQL 中配置权限
  module: Secure Azure Database for PostgreSQL
---

# 在 Azure Database for PostgreSQL 中配置权限

在本实验室练习中，你将分配 RBAC 角色来控制对 Azure Database for PostgreSQL 资源和 PostgreSQL GRANTS 的访问，以控制对数据库操作的访问。

## 开始之前

需要有自己的 Azure 订阅才能完成本练习。 如果还没有 Azure 订阅，请创建一个 [Azure 免费试用版](https://azure.microsoft.com/free)。

若要完成这些练习，需要安装连接到 Microsoft Entra ID（前身是 Azure Active Directory）的 PostgreSQL 服务器

### 创建资源组。

1. 在 Web 浏览器中，导航到 [Azure 门户](https://portal.azure.com)。 使用所有者或参与者帐户登录。
2. 在 Azure 服务下，选择“资源组”，然后选择“+ 创建”。
3. 检查是否显示了正确的订阅，然后输入资源组名称 **rg-PostgreSQL_Entra**。 选择一个**区域**。
4. 选择“查看 + 创建”。 然后选择“创建”。

## 创建 Azure Database for PostgreSQL 灵活服务器

1. 在“Azure 服务”下，选择“+ 创建资源”。
    1. 在“**搜索市场**”中，键入**`azure database for postgresql flexible server`**，选择“**Azure Database for PostgreSQL 灵活服务器**”，然后单击“**创建**”。
1. 在灵活服务器“基本信息”选项卡上，输入每个字段，如下所示：
    1. 订阅 - 你的订阅。
    1. 资源组 - **rg-PostgreSQL_Entra**。
    1. 服务器名称 - **psql-postgresql-fx7777**（服务器名称必须是全局唯一的，因此请将 7777 替换为四个随机数）。
    1. 区域 - 选择与资源组相同的区域。
    1. PostgreSQL 版本 - 选择“16”。
    1. 工作负载类型 - 开发。
    1. 计算 + 存储 - **可突发，B1ms**。
    1. 可用性区域 - 无首选项。
    1. 高可用性 - 保持未选中状态。
    1. 身份验证方法 - 选择 **PostgreSQL 和 Microsoft Entra 身份验证**。
    1. 设置 Microsoft Entra 管理员，选择“**设置管理员**”。
        1. 在“**选择 Microsoft Entra 管理员**”中搜索并选择你的帐户，然后单击“**选择**”
    1. 在**管理员用户名**中，输入**`demo`**。
    1. 在**密码**中，输入合适的复杂密码。
    1. 选择“下一步: 网络 >”****。
1. 在灵活服务器“网络”选项卡上，输入每个字段，如下所示：
    1. 连接方法：(o) 公共访问（允许的 IP 地址）和专用终结点
    1. 公共访问，选择 **允许公众使用公共 IP 地址通过互联网访问此资源**
    1. 在“防火墙规则”下，选择“+ 添加当前客户端 IP 地址”，以将当前 IP 地址添加为防火墙规则。 可以选择为此防火墙规则指定一个有意义的名称。 此外，选择“**添加0.0.0.0 - 255.255.255.255**”，然后单击“**继续**”
1. 选择“查看 + 创建”。 查看设置，然后选择“**创建**”以创建 Azure Database for PostgreSQL 灵活服务器。 部署完成后，选择“转到资源”以准备下一步。

## 安装 Azure Data Studio

安装 Azure Data Studio 以用于 Azure Database for PostgreSQL：

1. 在浏览器中，导航到[下载并安装 Azure Data Studio](/sql/azure-data-studio/download-azure-data-studio)，然后在 Windows 平台下，选择“用户安装程序(推荐)”。 可执行文件将下载到“下载”文件夹。
1. 选择“打开文件”。
1. 此时将显示“许可协议”。 阅读并接受协议，然后选择“下一步”。
1. 在“选择其他任务”中，选择“添加到 PATH”以及所需的任何其他添加内容。 选择“**下一步**”。
1. 此时将显示“准备安装”对话框。 复查你的设置。 选择“返回”进行更改，或选择“安装”。
1. 此时将显示“正在完成 Azure Data Studio 安装向导”对话框。 选择“完成”。 Azure Data Studio 启动。

### 安装 PostgreSQL 扩展

1. 如果 Azure Data Studio 尚未打开，请将其打开。
1. 在左侧菜单中，选择“扩展”以显示“扩展”面板。
1. 在搜索栏中，输入“PostgreSQL”。 将显示 Azure Data Studio 的 PostgreSQL 扩展图标。
1. 选择“安装”  。 扩展安装。

### 连接到 Azure Database for PostgreSQL 灵活服务器

1. 如果 Azure Data Studio 尚未打开，请将其打开。
1. 从左侧菜单中，选择“连接”。
1. 选择“新建连接”。
1. 在“连接详细信息”下的“连接类型”中，从下拉列表中选择“PostgreSQL”。
1. 在“服务器名称”中，输入 Azure 门户上显示的完整服务器名称。
1. 在“身份验证类型”中，保留密码。
1. 在“用户名和密码”中，输入用户名 **demo** 和上面创建的复杂密码
1. 选择 [ x ] 记住密码。
1. 其余字段是可选的。
1. 选择“连接” 。 你已连接到 Azure Database for PostgreSQL 服务器。
1. 将显示服务器数据库的列表。 这包括系统数据库和用户数据库。

### 创建 zoo 数据库

1. 导航到包含练习脚本文件的文件夹，或从 [MSLearn PostgreSQL 实验室](https://github.com/MicrosoftLearning/mslearn-postgresql/blob/main/Allfiles/Labs/02)下载 **Lab2_ZooDb.sql**。
1. 如果 Azure Data Studio 尚未打开，请将其打开。
1. 依次选择“文件”、“打开文件”，并导航到保存脚本的文件夹。 选择 **../Allfiles/Labs/02/Lab2_ZooDb.sql** 和“**打开**”。 如果显示信任警告，请选择“打开”。
1. 运行该脚本。 创建 zoodb 数据库。

## 在 Microsoft Entra ID 中创建新的用户帐户

> [!NOTE]
> 在大多数生产或开发环境中，你很可能没有在 Microsoft Entra ID 服务上创建帐户的订阅帐户权限。  在这种情况下，如果组织允许，请尝试请求 Microsoft Entra ID 管理员为你创建一个测试帐户。 如果无法获取测试 Entra 帐户，请跳过本部分并继续访问**授予 Azure Database for PostgreSQL 的访问权限**部分。 

1. 在 [Azure 门户](https://portal.azure.com)中，使用所有者帐户登录并导航到 Microsoft Entra ID。
1. 在“管理”下，选择“用户” 。
1. 在左上角，选择“新建用户”，然后选择“创建新用户”。********
1. 在“新建用户”页中，输入以下详细信息，然后选择“创建”********：
    - **用户主体名称：** 选择主体名称
    - **显示名称：** 选择显示名称
    - **密码：** 取消选中“**自动生成密码**”，然后输入强密码。 记下主体名称和密码。
    - 单击“查看 + 创建”

    > [!TIP]
    > 创建用户后，请记下完整的“用户主体名称”****，以便稍后可以使用它登录。

### 分配读者角色

1. 在 Azure 门户中，选择“所有资源”****，然后选择 Azure Database for PostgreSQL 资源。
1. 选择“访问控制(IAM)”，然后选择“角色分配”。******** 新帐户未出现在列表中。
1. 选择“+ 添加”，然后选择“添加角色分配”。********
1. 选择“读者”角色，然后选择“下一步”。********
1. 选择 **“+ 选择成员**”，将上一步骤中添加的新帐户添加到成员列表中，然后选择“**下一步**”。
1. 选择“查看 + 分配”。

### 测试读者角色

1. 在 Azure 门户的右上方，选择用户帐户，然后选择“注销”****。
1. 使用你记下的用户主体名称和密码以新用户身份登录。 如果系统提示输入并记下新密码，请替换默认密码。
1. 如果系统提示进行多重身份验证，请选择“**稍后询问我**”
1. 在门户主页中，选择“所有资源”****，然后选择 Azure Database for PostgreSQL 资源。
1. 选择“停止”  。 显示错误，因为“读者”角色使你能够查看资源，但不能更改它。

### 分配参与者角色

1. 在 Azure 门户的右上角，选择新帐户的用户帐户，然后选择“**注销**”。
1. 使用原始所有者帐户登录。
1. 导航到 Azure Database for PostgreSQL 资源，然后选择“访问控制(IAM)”。****
1. 选择“+ 添加”，然后选择“添加角色分配”。********
1. 选择“**特权管理员角色**”
1. 选择“参与者”角色，然后选择“下一步”。********
1. 添加之前添加到成员列表中的新帐户，然后选择“**下一步**”。
1. 选择“查看 + 分配”。
1. 选择“角色分配”。**** 新帐户现在同时分配了“读者”和“参与者”角色。

## 测试参与者角色

1. 在 Azure 门户的右上方，选择用户帐户，然后选择“注销”****。
1. 使用你记下的用户主体名称和密码以新帐户身份登录。
1. 在门户主页中，选择“所有资源”****，然后选择 Azure Database for MySQL 资源。
1. 选择“停止”，然后选择“是”。******** 这一次，服务器停止运行时没有出现错误，因为新帐户已分配了必要的角色。
1. 选择“**开始**”以确保 PostgreSQL 服务器已为后续步骤做好准备。
1. 在 Azure 门户的右上角，选择新帐户的用户帐户，然后选择“**注销**”。
1. 使用原始所有者帐户登录。

## 授予对 Azure Database for PostgreSQL 的访问权限

1. 打开 Azure Data Studio，并使用上面设置为管理员的 **demo** 用户连接到 Azure Database for PostgreSQL 服务器。
1. 在查询窗格中，针对 Postgres 数据库执行此代码。 应返回 12 个用户角色，包括用于连接的演示角色：****

    ```SQL
    SELECT rolname FROM pg_catalog.pg_roles;
    ```

1. 若要创建新角色，请执行此代码

    ```SQL
    CREATE ROLE dbuser WITH LOGIN NOSUPERUSER INHERIT CREATEDB NOCREATEROLE NOREPLICATION PASSWORD 'R3placeWithAComplexPW!';
    GRANT CONNECT ON DATABASE zoodb TO dbuser;
    ```
    > [!NOTE]
    > 请确保将上述脚本中的密码替换为复杂密码。

1. 若要列出新角色，请再次在 **pg_catalog.pg_roles** 中执行上述 SELECT 查询。 应看到列出 dbuser 角色。****
1. 若要使新角色能够查询和修改 **zoodb** 数据库中 **animal** 表中的数据，请针对 zoodb 数据库执行以下代码：

    ```SQL
    GRANT SELECT, INSERT, UPDATE, DELETE ON animal TO dbuser;
    ```

## 测试新角色

1. 在 Azure Data Studio 中，在 CONNECTIONS 列表中选择新的连接按钮。****
1. 在“连接类型”列表中，选择“PostgreSQL”********。
1. 在“服务器名称”文本框中，键入 Azure Database for PostgreSQL 资源的完全限定的服务器名称。**** 可以从 Azure 门户中复制它。
1. 在“验证类型”列表中，选择“密码”********。
1. 在“**用户名**”文本框中，键入 **dbuser**，并在“**密码**”文本框中键入创建帐户时使用的复杂密码。
1. 选中“记住密码”复选框，然后选择“连接”********。
1. 选择“新建查询”，然后执行以下代码：****

    ```SQL
    SELECT * FROM animal;
    ```

1. 若要测试是否具有 UPDATE 权限，请执行以下代码：

    ```SQL
    UPDATE animal SET name = 'Linda Lioness' WHERE ani_id = 7;
    SELECT * FROM animal;
    ```

1. 若要测试是否具有 DROP 权限，请执行以下代码。 如果出现错误，请检查错误代码：

    ```SQL
    DROP TABLE animal;
    ```

1. 若要测试是否具有 GRANT 权限，请执行以下代码：

    ```SQL
    GRANT ALL PRIVILEGES ON animal TO dbuser;
    ```

这些测试表明，新用户可以执行数据操作语言 (DML) 命令来查询和修改数据，但无法使用数据定义语言 (DDL) 命令更改架构。 此外，新用户无法授予任何规避权限的新特权。

## 清理

你将不再使用此 PostgreSQL 服务器，因此请删除创建的资源组，这样就能移除服务器。
