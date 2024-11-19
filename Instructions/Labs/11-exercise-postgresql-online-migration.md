---
lab:
  title: PostgreSQL 数据库联机迁移
  module: Migrate to Azure Database for PostgreSQL Flexible Server
---

## PostgreSQL 数据库联机迁移

在本练习中，你将在源 PostgreSQL 服务器和 Azure Database for PostgreSQL 灵活服务器之间配置逻辑复制，以允许进行联机迁移活动。

## 开始之前

需要有自己的 Azure 订阅才能完成本练习。 如果还没有 Azure 订阅，可以创建一个 [Azure 免费试用版](https://azure.microsoft.com/free)。

> **注意：**：本练习要求用作迁移源的服务器可供 Azure Database for PostgreSQL 灵活服务器访问，以便它可以连接和迁移数据库。 这要求源服务器可通过公共 IP 地址和端口进行访问。 > Azure 区域 IP 地址列表可从 [Azure IP 范围和服务标签 - 公有云](https://www.microsoft.com/en-gb/download/details.aspx?id=56519)下载，以帮助根据所使用的 Azure 区域最大限度地减少防火墙规则中允许的 IP 地址范围。

打开服务器防火墙，允许 Azure Database for PostgreSQL 灵活服务器中的迁移功能访问源 PostgreSQL 服务器（默认情况下为 TCP 端口 5432）。
>
在源数据库的前面使用防火墙设备时，可能需要添加防火墙规则，以允许 Azure Database for PostgreSQL 灵活服务器中的迁移功能访问源数据库进行迁移。
>
> 用于迁移的 PostgreSQL 支持的最大版本是版本 16。

### 先决条件

> **注意**：在开始本练习之前，需要完成上一练习，使源数据库和目标数据库到位，以便配置逻辑复制，因为本练习基于该练习中的活动。

## 创建发布 - 源服务器

1. 打开 PGAdmin 并连接到源服务器，其中包含将充当数据同步到 Azure Database for PostgreSQL 灵活服务器的源的数据库。
1. 打开连接到源数据库的新查询窗口，其中包含要同步的数据。
1. 将源服务器 wal_level 配置为**逻辑**，以允许发布数据。
    1. 找到并打开 PostgreSQL 安装目录下 bin 目录中的 **postgresql.conf** 文件。
    1. 找到配置设置 **wal_level** 的行。
    1. 确保行未注释，并将值设置为**逻辑**。
    1. 保存并关闭该文件。
    1. 然后重启 PostgreSQL 服务。
1. 现在配置一个发布，该发布将包含数据库中的所有表。

    ```SQL
    CREATE PUBLICATION migration1 FOR ALL TABLES;
    ```

## 创建订阅 - 目标服务器

1. 打开 PGAdmin 并连接到 Azure Database for PostgreSQL 灵活服务器，其中包含将充当源服务器中的数据同步的目标的数据库。
1. 打开连接到源数据库的新查询窗口，其中包含要同步的数据。
1. 创建源服务器的订阅。

    ```sql
    CREATE SUBSCRIPTION migration1
    CONNECTION 'host=<source server name> port=<server port> dbname=adventureworks application_name=migration1 user=<username> password=<password>'
    PUBLICATION migration1
    WITH(copy_data = false)
    ;    
    ```

1. 检查表复制的状态。

    ```SQL
    SELECT PT.schemaname, PT.tablename,
        CASE PS.srsubstate
            WHEN 'i' THEN 'initialize'
            WHEN 'd' THEN 'data is being copied'
            WHEN 'f' THEN 'finished table copy'
            WHEN 's' THEN 'synchronized'
            WHEN 'r' THEN ' ready (normal replication)'
            ELSE 'unknown'
        END AS replicationState
    FROM pg_publication_tables PT,
            pg_subscription_rel PS
            JOIN pg_class C ON (C.oid = PS.srrelid)
            JOIN pg_namespace N ON (N.oid = C.relnamespace)
    ;
    ```

## 测试数据复制

1. 在源服务器上检查工单表的行计数。

    ```SQL
    SELECT COUNT(*) FROM production.workorder;
    ```

1. 在目标服务器上，检查工单表的行计数。

    ```SQL
    SELECT COUNT(*) FROM production.workorder;
    ```

1. 检查行计数值是否匹配。
1. 现在，将 Lab11_workorder.csv 文件从存储库[此处](https://github.com/MicrosoftLearning/mslearn-postgresql/tree/main/Allfiles/Labs/11)下载到 C:\
1. 使用以下命令从 CSV 将新数据加载到源服务器上的工单表中。

    ```Bash
    psql --host=localhost --port=5432 --username=postgres --dbname=adventureworks --command="\COPY production.workorder FROM 'C:\Lab11_workorder.csv' CSV HEADER"
    ```

命令输出应为 `COPY 490`，指示从 CSV 文件写入表中的 490 行。

1. 检查源（72591 行）和目标中工作单表的行数是否匹配，以验证数据复制是否正常工作。

## 练习清理

我们在本练习中部署的 Azure Database for PostgreSQL 将产生费用，你可以在本练习后删除服务器。 或者，你可以删除 **rg-learn-work-with-postgresql-eastus** 资源组，以移除在本练习中部署的所有资源。
