[合集 \- .NET云原生应用实践(3\)](https://github.com)[1\..NET云原生应用实践（二）：Sticker微服务RESTful API的实现10\-13](https://github.com/daxnet/p/18456878)[2\..NET云原生应用实践（一）：从搭建项目框架结构开始10\-09](https://github.com/daxnet/p/18172088)3\..NET云原生应用实践（三）：连接到PostgreSQL数据库10\-22收起
# 本章目标


1. 实现基于PostgreSQL的SDAC（简单数据访问层）
2. 将Stickers微服务切换到使用PostgreSQL SDAC


# 为什么选择PostgreSQL数据库？


其实并不一定要选择PostgreSQL数据库，这里主要出于几个方面考虑：


1. PostgreSQL免费易用，轻量效率高，能够满足目前的需求
2. PostgreSQL生态成熟，资源丰富，遇到问题容易排查
3. 后续Keycloak以及Hangfire/Quartz.NET集成PostgreSQL比较方便，减少一个数据库实例的部署，可以减少一部分成本开销，至少在我们的这个简单案例中是这样


基于文档的MongoDB也是一个不错的选择，但是出于上面第三点考虑，有些所需依赖的第三方解决方案对MongoDB的支持并不是那么完美，所以，在我们的案例中选择了PostgreSQL作为数据库。


访问PostgreSQL数据库可以使用ADO.NET体系，具体说是使用官方的[Npgsql](https://github.com)包，然后用ADO.NET的编程模型和编程范式来读写PostgreSQL数据库。由于在某些场景下使用ADO.NET还不是特别的方便，比如根据Id查询某个“贴纸”实体的时候，需要首先构建DbCommand对象，然后将Id作为参数传入，之后执行DbReader逐行读入数据，并根据读入的数据构建Sticker对象，最后关闭数据库连接。所以，在本案例中，我会选择一个轻量型的对象映射框架：[Dapper](https://github.com)来实现基于PostgreSQL的SDAC。


# 准备PostgreSQL数据库与开发工具


你可以选择两种方式来准备PostgreSQL数据库：


1. 本地机器直接安装PostgreSQL服务
2. 使用Docker


单从开发的角度，选择第一种方式更为简单，下载PostgreSQL服务器安装包后直接安装就能运行起来，但在本案例中，我选择使用Docker来运行PostgreSQL。首先，部署和分发更为简单，有一个Dockerfile的定义文件就可以编译出Docker镜像并在本地运行容器，Dockerfile文件可以方便地托管在代码仓库中进行版本控制；其次，今后的云端部署以Docker容器的方式也会更为简单方便，使用Azure提供的Web App for Containers托管服务，或者部署到Azure Kubernetes Service（AKS），都可以很方便地将容器化的应用程序直接部署到Azure上。



> 如果你仍然选择使用本地机器直接安装PostgreSQL服务的方式，请跳过接下来的Docker部分，直接进入数据库和数据表的创建部分。


## 在Docker中运行PostgreSQL数据库


你可以直接使用docker run命令来运行一个PostgreSQL数据库，不过，在这里我们稍微做一点调整：我们基于PostgreSQL的官方镜像，自己构建一个自定义的PostgreSQL镜像，这样做的目的到本文最后部分我会介绍。


首先，在`src`文件夹的同级文件夹中，创建一个名为`docker`的文件夹，然后再在`docker`文件夹下，创建一个名为`postgresql`的文件夹，然后在`postgresql`文件夹中，新建一个`Dockerfile`：



```


|  | $ mkdir -p docker/postgresql |
| --- | --- |
|  | $ cd docker/postgresql |
|  | $ echo "FROM postgres:17.0-alpine" >> Dockerfile |


```

这个`Dockerfile`目前只有一条指令，就是基于`postgres:17.0-alpine`这个镜像来构建新的镜像，你现在就可以使用docker build命令，基于这个Dockerfile来创建镜像，做法是，在postgresql文件夹下（在Dockerfile相同的目录下），执行docker build命令：



```


|  | $ docker build -t daxnet/stickers-pgsql:dev . |
| --- | --- |


```


> 注意：这里我使用daxnet/stickers\-pgsql作为镜像名称，因为后面我需要将这个镜像发布到Docker Hub中，所以镜像名称中带有我在Docker Hub中的Registry名称。请可以根据自己的情况来决定镜像名称。
> 
> 
> 注意：在选择Base Image（也就是这里的postgres镜像）时，通常我们会指定一个特定的tag，而不是使用latest tag，这是为了防止在持续集成的时候由于使用了一个更新版本的镜像而导致一些版本兼容性问题。


 我们还可以使用Docker Compose来构建镜像。在docker文件夹下，新建一个名为`docker-compose.dev.yaml`的文件，其内容为：



```


|  | volumes: |
| --- | --- |
|  | stickers_postgres_data: |
|  |  |
|  | services: |
|  | stickers-pgsql: |
|  | image: daxnet/stickers-pgsql:dev |
|  | build: |
|  | context: ./postgresql |
|  | dockerfile: Dockerfile |
|  | environment: |
|  | - POSTGRES_USER=postgres |
|  | - POSTGRES_PASSWORD=postgres |
|  | - POSTGRES_DB=stickersdb |
|  | volumes: |
|  | - stickers_postgres_data:/data:Z |
|  | ports: |
|  | - "5432:5432" |


```

使用Docker Compose的好处是，它可以批量构建多个镜像，并且你不需要在每次编译的时候，都去考虑每个镜像应该使用什么名称什么tag，不仅如此，同一个Docker Compose文件还可以用来定义容器启动的配置和参数，并将所定义的容器一并运行起来。



> 请注意这里的Docker Compose文件名（docker\-compose.dev.yaml）中有“dev”字样，这是因为，我打算在这个Docker Compose文件中仅包含支持开发环境的基础设施相关的容器，比如数据库、Keycloak、日志存储、缓存、Elasticsearch等等，而今后我们的案例应用程序本身的docker容器并不会包含在这个Compose文件中，这样做的好处是，只需要一条Docker Compose命令就将运行和调试我们的案例程序的所有基础设施服务全部启动起来，然后只需要在IDE中调试我们的程序即可。


准备好上面的docker\-compose.dev.yaml文件之后，即可使用下面两条命令来编译和运行PostgreSQL数据库服务了：



```


|  | $ docker compose -f docker-compose.dev.yaml build |
| --- | --- |
|  | $ docker compose -f docker-compose.dev.yaml up |


```

成功启动容器之后，应该可以看到类似下面的输出日志：



```


|  | stickers-pgsql-1  | 2024-10-17 12:55:55.024 UTC [1] LOG:  database system is ready to accept connections |
| --- | --- |


```

## 创建数据库与数据表


可以使用pgAdmin等客户端工具连接到数据库，先创建一个名为`stickersdb`的数据库，然后在这个数据库上，执行下面的SQL语句来创建数据表：



```


|  | CREATE TABLE public.stickers ( |
| --- | --- |
|  | "Id" integer NOT NULL, |
|  | "Title" character varying(128) NOT NULL, |
|  | "Content" text NOT NULL, |
|  | "DateCreated" timestamp with time zone NOT NULL, |
|  | "DateModified" timestamp with time zone |
|  | ); |
|  |  |
|  |  |
|  | ALTER TABLE public.stickers OWNER TO postgres; |
|  |  |
|  | ALTER TABLE public.stickers ALTER COLUMN "Id" ADD GENERATED ALWAYS AS IDENTITY ( |
|  | SEQUENCE NAME public."stickers_Id_seq" |
|  | START WITH 1 |
|  | INCREMENT BY 1 |
|  | NO MINVALUE |
|  | NO MAXVALUE |
|  | CACHE 1 |
|  | ); |


```

数据库创建成功之后，就可以进行下一步：设计和实现PostgreSQL的数据访问层了。


# PostgreSQL数据访问层


在[上一讲](https://github.com)中，我们已经设计好了数据访问层的基本结构，现在只需要在上面进行扩展即可，详细的UML类图如下：


![](https://img2024.cnblogs.com/blog/119825/202410/119825-20241020093110436-550788707.png)


上图中淡黄色部分就是这次新增的类以及对外部类库的依赖。我们会新建一个`Stickers.DataAccess.PostgreSQL`的类库（包，或者称之为.NET Assembly），它包含一个主要类：`PostgreSqlDataAccessor`，对标之前我们设计的`Stickers.DataAccess.InMemory`类库和`InMemoryDataAccessor`类，这个类也实现了`ISimplifiedDataAccessor`接口，只不过它的具体实现部分需要通过Npgsql来访问PostgreSQL。正如上面所说，这里还引用了Dapper库，用来简化ADO.NET的操作。



> 将PostgreSqlDataAccessor置于一个单独的类库中，其原因在前一篇文章中我也介绍过，因为PostgreSqlDataAccessor有外部依赖，我们不应该将这种依赖“污染”到核心库（Stickers.Common）中，这样的隔离还可以带来另一个好处，那就是不同的Simplified Data Accessor（SDAC）可以被打包成不同的组件库，这样，不仅可以提高系统的可测试性，而且在实现应用程序时，可以选择不同的组件接入，提供系统的稳定性和灵活性。


从上图还可以看出，接下来`StickersController`将会被注入`PostgreSqlDataAccessor`的实例，从此处开始，`InMemoryDataAccessor`将退出历史舞台。


接下来，我将介绍一下`PostgreSqlDataAccessor`中的一些实现细节。


## 对象关系映射（ORM）


我们没有引入ORM框架，虽然Dapper的官方定义是一个简单的对象映射框架，在直接使用Dapper的时候，可以在Dapper上以各种不同的方式实现对象映射（从C\#对象到数据库表、字段的映射），但是在我们的设计中，却没有办法在`PostgreSqlDataAccessor`中得知，我们应该基于哪个对象进行映射操作，因为这里的对象模型`Sticker`类是一个业务概念，它是定义在`StickersController`中的，我们不能假设在`PostgreSqlDataAccessor`中操作的对象类型就是`Sticker`类，所以也就没有办法在`PostgreSqlDataAccessor`中直接写出能被Dapper所使用的映射方式。所以，我们需要自己定义一个简单的对象关系映射机制，然后在`PostgreSqlDataAccessor`中基于这个机制来生成Dapper所能使用的映射方式。


目前我们的设计还是比较简单，只有一个业务对象：`Sticker`，并且也没有其它对象跟它有直接关系，所以，就怎么简单怎么做，问题也就变成了：将一个对象映射到一张数据表上，并将该对象的属性映射到表的字段上，而且，在应用程序中，都是使用已有的数据表和字段，并不存在需要在应用程序启动的时候去创建数据表和字段的需求，所以，在这个映射上，我们甚至不需要去定义各个字段的类型以及约束条件。


我们可以使用C\#的Attribute，用来指定被修饰的对象和其属性应该映射到哪个表的哪些字段，例如可以在Stickers.Common中定义两个Attribute：



```


|  | [AttributeUsage(AttributeTargets.Class)] |
| --- | --- |
|  | public sealed class TableAttribute(string tableName) : Attribute |
|  | { |
|  | public string TableName { get; } = tableName; |
|  | } |
|  |  |
|  | [AttributeUsage(AttributeTargets.Property)] |
|  | public sealed class FieldAttribute(string fieldName) : Attribute |
|  | { |
|  | public string FieldName { get; } = fieldName; |
|  | } |


```

然后，在模型类型上，应用这些Attribute：



```


|  | [Table("Stickers")] |
| --- | --- |
|  | public class Sticker(string title, string content): IEntity |
|  | { |
|  | public Sticker() : this(string.Empty, string.Empty) { } |
|  |  |
|  | public int Id { get; set; } |
|  |  |
|  | [Required] |
|  | [StringLength(50)] |
|  | public string Title { get; set; } = title; |
|  |  |
|  | public string Content { get; set; } = content; |
|  |  |
|  | [Field("DateCreated")] |
|  | public DateTime CreatedOn { get; set; } = DateTime.UtcNow; |
|  |  |
|  | [Field("DateModified")] |
|  | public DateTime? ModifiedOn { get; set; } |
|  | } |


```

在上面的代码中，`Sticker`类通过`TableAttribute`映射到`Stickers`数据表，而`CreatedOn`和`ModifiedOn`属性通过`FieldAttribute`映射到了`DateCreated`和`DateModified`字段，而其它的没有使用`FieldAttribute`字段的属性，则默认使用属性名作为字段名。于是，就可以使用下面的方法来根据传入的对象类型来获得数据库中的表名和字段名：



```


|  | private static string GetTableName<TEntity>() where TEntity : class, IEntity |
| --- | --- |
|  | { |
|  | return typeof(TEntity).IsDefined(typeof(TableAttribute), false) |
|  | ? $"public.{typeof(TEntity).GetCustomAttribute()?.TableName ?? typeof(TEntity).Name}" |
|  | : $"public.{typeof(TEntity).Name}"; |
|  | } |
|  | private static IEnumerable<KeyValuePair<string, string>> GetColumnNames<TEntity>( |
|  | params Predicate[] excludes) |
|  | where TEntity : class, IEntity |
|  | { |
|  | return from p in typeof(TEntity).GetProperties() |
|  | where p.CanRead && p.CanWrite && !excludes.Any(pred => pred(p)) |
|  | select p.IsDefined(typeof(FieldAttribute), false) |
|  | ? new KeyValuePair<string, string>(p.Name, p.GetCustomAttribute()?.FieldName ?? p.Name) |
|  | : new KeyValuePair<string, string>(p.Name, p.Name); |
|  | } |


```

然后就可以在`PostgreSqlDataAccessor`中使用这两个方法来构建SQL语句了。比如在`AddAsync`方法中：



```


|  | public async Task<int> AddAsync<TEntity>(TEntity entity, CancellationToken cancellationToken = default) |
| --- | --- |
|  | where TEntity : class, IEntity |
|  | { |
|  | var tableName = GetTableName(); |
|  | var tableFieldNames = string.Join(", ", |
|  | GetColumnNames(p => p.Name == nameof(IEntity.Id)) |
|  | .Select(n => $"\"{n.Value}\"")); |
|  | var fieldValueNames = string.Join(", ", |
|  | GetColumnNames(p => p.Name == nameof(IEntity.Id)) |
|  | .Select(n => $"@{n.Key}")); |
|  | var sql = $@"INSERT INTO {tableName} ({tableFieldNames}) VALUES ({fieldValueNames}) RETURNING ""Id"""; |
|  | await using var sqlConnection = new NpgsqlConnection(connectionString); |
|  | var id = await sqlConnection.ExecuteScalarAsync<int>(sql, entity); |
|  | entity.Id = id; |
|  | return id; |
|  | } |


```

其中`GetColumnNames`方法的`excludes`参数是一个断言委托，通过它可以指定不需要获得字段名的属性。比如上面的`AddAsync`方法中，向数据表增加一条记录的`INSERT`语句，并不需要将Id值插入（相反，是需要获得数据库返回的Id值），所以构建这条SQL语句的时候，就不需要获取Id属性所对应的字段名。


## 根据Lambda表达式构建SQL字段名


在ISimplifiedDataAccessor中，GetPaginatedEntitiesAsync方法的第一个参数是一个Lambda表达式，它通过Lambda表达式指定用于排序的对象属性：



```


|  | Task> GetPaginatedEntitiesAsync(Expression> orderByExpression, |
| --- | --- |
|  | bool sortAscending = true, int pageSize = 25, int pageNumber = 0, |
|  | Expressionbool>>? filterExpression = null, CancellationToken cancellationToken = default) |
|  | where TEntity : class, IEntity; |


```

然而，我们使用Dapper和Npgsql查询数据库的时候，是需要构建SQL语句的，因此，这里就需要将Lambda表达式转换为SQL语句，重点就是如何通过Lambda表达式来获得`ORDER BY`子句的字段名。基本思路是，将Lambda表达式转换为MemberExpression对象，然后通过MemberExpression获得Member属性的Name属性即可。有一种特殊的情况，就是传入的Lambda表达式有可能是一个Convert方法调用（参考[上文中](https://github.com):[楚门加速器p](https://tianchuang88.com)`StickersController.GetStickersAsync`方法的实现），此时就需要先将Convert方法的参数转换为MemberExpression，然后再获得属性名。代码如下：



```


|  | private static string BuildSqlFieldName(Expression expression) |
| --- | --- |
|  | { |
|  | if (expression is not MemberExpression && expression is not UnaryExpression) |
|  | throw new NotSupportedException("Expression is not a member expression"); |
|  | var memberExpression = expression switch |
|  | { |
|  | MemberExpression expr => expr, |
|  | UnaryExpression { NodeType: ExpressionType.Convert } unaryExpr => |
|  | (MemberExpression)unaryExpr.Operand, |
|  | _ => null |
|  | }; |
|  | if (memberExpression is null) |
|  | throw new NotSupportedException("Can't infer the member expression from the given expression."); |
|  | return memberExpression.Member.IsDefined(typeof(FieldAttribute), false) |
|  | ? memberExpression.Member.GetCustomAttribute()?.FieldName ?? |
|  | memberExpression.Member.Name |
|  | : memberExpression.Member.Name; |
|  | } |


```

那么，将Lambda表达式转换成ORDER BY子句，就可以这样实现：



```


|  | var sortExpression = BuildSqlFieldName(orderByExpression.Body); |
| --- | --- |
|  | if (!string.IsNullOrEmpty(sortExpression)) |
|  | { |
|  | sqlBuilder.Append($@"ORDER BY ""{sortExpression}"" "); |
|  | // ... |
|  | } |


```

到这里，或许你会有疑问，在`StickersController.GetStickersAsync`方法中，它是从客户端RESTful API请求中以字符串形式读入排序字段名的，在该方法中，会将这个字符串的排序字段名转换为Lambda表达式然后传给`ISimplifiedDataAccessor`（其实也就是这里的`PostgreSqlDataAccessor`），可是到`PostgreSqlDataAccessor`中，又将这个Lambda表达式转回了字符串形式用来拼接SQL语句，岂不是多此一举？其实不然，因为从整体设计的角度，`ISimplifiedDataAccessor`中使用Lambda表达式来定义排序和筛选字段，这个是合理的，因为这样的设计不仅可以满足本文介绍的这种SQL字符串拼接的实现方式，而且还可以满足基于Lambda表达式的数据访问组件的设计（比如Entity Framework）。因此，不能因为一种具体实现的特殊性而影响一种更为通用的设计，说得具体些，此处将Lambda表达式再转换成字段名字符串，是`PostgreSqlDataAccessor`的特殊性导致的，而不是设计本身的通用性引起的问题。当然，Dapper也有一些非官方的扩展库，允许在排序和筛选的时候直接使用Lambda表达式，不过在这里我还是不想引入过多的外部依赖，把问题变得直接简单一些，同时也可以引出这里的设计问题和解决思路。


## 根据Lambda表达式构建WHERE子句


在StickersController中，有下面的代码，这段代码的目的是在创建一个“贴纸”的时候，先判断相同标题的贴纸是否已经存在：



```


|  | var exists = await dac.ExistsAsync(s => s.Title == title); |
| --- | --- |
|  | if (exists) return Conflict($"""Sticker "{sticker.Title}" already exists."""); |


```

在`ISimplifiedDataAccessor`中，`ExistsAsync`方法通过一个Lambda表达式参数来决定数据的筛选条件，于是根据`PostgreSqlDataAccessor`的实现方式，这里就需要将这个Lambda表达式转换成WHERE子句，从而可以在Dapper上执行SQL语句进行查询。比如，假设上面的代码中，title的值是“这是一张测试贴纸”，那么，产生的WHERE子句就应该是这样：



```


|  | WHERE "Title" = '这是一张测试贴纸' |
| --- | --- |


```

下面就是将Lambda表达式转换为SQL WHERE子句的代码：



```


|  | private static string BuildSqlWhereClause(Expression expression) |
| --- | --- |
|  | { |
|  | // 根据expression的类型来确定SQL中的比较操作符 |
|  | var oper = expression.NodeType switch |
|  | { |
|  | ExpressionType.Equal => "=", |
|  | ExpressionType.NotEqual => "<>", |
|  | ExpressionType.GreaterThan => ">", |
|  | ExpressionType.GreaterThanOrEqual => ">=", |
|  | ExpressionType.LessThan => "<", |
|  | ExpressionType.LessThanOrEqual => "<=", |
|  | _ => null |
|  | }; |
|  |  |
|  | // 目前仅支持上面列出的这些操作符 |
|  | if (string.IsNullOrEmpty(oper)) throw new NotSupportedException("The filter expression is not supported."); |
|  |  |
|  | if (expression is not BinaryExpression { Left: MemberExpression leftMemberExpression } binaryExpression) |
|  | throw new NotSupportedException("The filter expression is not supported."); |
|  |  |
|  | // 获得数据库中对应的字段名 |
|  | var fieldName = BuildSqlFieldName(leftMemberExpression); |
|  | string? fieldValue = null; |
|  |  |
|  | // 获得用于WHERE子句的字段的值，如果BinaryExpression |
|  | // 的右边部分是一个常量表达式，则直接使用常量，否则就要用 |
|  | // 反射的方式计算获得字段的值。 |
|  | switch (binaryExpression.Right) |
|  | { |
|  | case ConstantExpression rightConstExpression: |
|  | fieldValue = FormatValue(rightConstExpression.Value); |
|  | break; |
|  | case MemberExpression rightMemberExpression: |
|  | var rightConst = rightMemberExpression.Expression as ConstantExpression; |
|  | var member = rightMemberExpression.Member.DeclaringType; |
|  | fieldValue = FormatValue(member?.GetField(rightMemberExpression.Member.Name) |
|  | ?.GetValue(rightConst?.Value)); |
|  | break; |
|  | } |
|  |  |
|  | // 返回WHERE子句的组成部分 |
|  | if (!string.IsNullOrEmpty(fieldValue)) |
|  | return $""" |
|  | "{fieldName}" {oper} {fieldValue} |
|  | """; |
|  |  |
|  | throw new NotSupportedException("The filter expression is not supported."); |
|  |  |
|  | // 将字段值进行格式化的本地方法，比如如果值是一个字符串，就要根据PostgreSQL的 |
|  | // SQL语句规范，将值的两边加上单引号 |
|  | string? FormatValue(object? value) |
|  | { |
|  | return value switch |
|  | { |
|  | null => null, |
|  | string => $"'{value}'", |
|  | _ => value.ToString() |
|  | }; |
|  | } |
|  | } |


```

我加了一些注释，但基本上可以归纳为：


1. 确保Lambda表达式是一个Binary Expression
2. 从该表达式获得比较操作符
3. 从Binary Expression的左边部分获得字段名
4. 从Binary Expression的右边部分获得字段值
5. 将字段名、操作符、字段值拼接成一个条件语句
6. 目前仅支持有限的比较操作符，并且不支持复杂的条件表达式（AND、OR这些）


如果有兴趣还可以继续扩展上面的方法，使其能够支持更为复杂的WHERE子句的构建逻辑，这里就不再展开了。


## PostgreSQL下分页查询的实现


分页查询需要在数据库这一层实现，其原因在《[在ASP.NET Core Web API上动态构建Lambda表达式实现指定字段的数据排序](https://github.com)》一文中我已经介绍过，就不多啰嗦了。这里的一个重点是，如何在一次数据库查询中同时获得当前页的数据集以及一共有多少条记录。在PostgreSQL中，要实现这个逻辑有不少方法，我选择了一个比较简单的，虽然它的性能并不一定是最好的，但目前来说已经够用了。


可以使用类似这样的SELECT语句来实现同时返回分页的数据集和总数据条数：



```


|  | SELECT "Id" Id, |
| --- | --- |
|  | "Title" Title, |
|  | "Content" Content, |
|  | "DateCreated" CreatedOn, |
|  | "DateModified" ModifiedOn, |
|  | COUNT(*) OVER() AS "TotalCount" |
|  | FROM public.Stickers |
|  | ORDER BY "Id" |
|  | OFFSET 0 LIMIT 20 |


```

在执行完这条SQL语句之后，我们不能通过Dapper的`connection.QueryAsync(sql)`调用直接将结果集转换为`Sticker`对象列表，因为这条SQL语句的返回字段中，不仅包含了`Sticker`对象的所有字段，而且还包含了一个用于保存记录总条数的`TotalCount`字段。直接将查询结果集转换为`Sticker`对象列表会丢失`TotalCount`信息。


所以，这里只能使用`connection.QueryAsync(sql)`这个函数重载（注意这里并不带有泛型参数）来获取一个动态（`dynamic`）对象的列表，由于这些动态对象本身都是`IDictionary`的实现类型，所以，在读入这些对象的时候，只需要直接创建Sticker对象，然后通过反射，把每个字段的值赋上去就行了：



```


|  | private static TEntity? BuildEntityFromDictionary(IDictionary<string, object> dictionary) |
| --- | --- |
|  | where TEntity : class, IEntity |
|  | { |
|  | var properties = from p in typeof(TEntity).GetProperties() |
|  | where p.CanRead && p.CanWrite && dictionary.ContainsKey(p.Name.ToLower()) |
|  | select p; |
|  | var obj = (TEntity?)Activator.CreateInstance(typeof(TEntity)); |
|  | foreach (var property in properties) property.SetValue(obj, dictionary[property.Name.ToLower()]); |
|  |  |
|  | return obj; |
|  | } |


```


> 在这里使用dynamic和反射，会影响性能吗？使用dynamic和反射当然没有直接访问对象的性能好，但是相对于与外部存储机制的访问和网络传输的延迟而言，这点性能损耗还是可以忽略不计的。事实上如果不是自己拼接SQL语句，而是使用ORM框架，那么不仅在某些技术的实现部分（映射、表联结JOIN等），而且在一些附加的功能实现（对象状态跟踪等）上，都会有性能损耗，所以不必多虑。


实现分页的详细代码这里就不列出来了，请直接参考本章案例代码即可。


# 在StickersController上使用PostgreSqlDataAccessor


在成功实现了`PostgreSqlDataAccessor`之后，在`StickersController`上使用它就变得非常简单。首先，将PostgreSQL的连接字符串作为配置参数提供给ASP.NET Core Web API应用程序，目前我们还处于调试阶段，暂时先放在`appsettings.Development.json`文件中即可，在这个文件中加一个db的属性，然后指定连接字符串：



```


|  | { |
| --- | --- |
|  | "Logging": { |
|  | "LogLevel": { |
|  | "Default": "Information", |
|  | "Microsoft.AspNetCore": "Warning" |
|  | } |
|  | }, |
|  | "db": { |
|  | "connectionString": "User ID=postgres;Password=postgres;Host=localhost;Port=5432;Database=stickersdb;Pooling=true;Connection Lifetime=0;" |
|  | } |
|  | } |


```

然后，在`Stickers.WebApi`项目上添加对`Stickers.DataAccess.PostgreSQL`项目的引用，再在`Program.cs`文件中，将`PostgreSqlDataAccessor`注入即可：



```


|  | var dbConnectionString = builder.Configuration["db:connectionString"]; |
| --- | --- |
|  | if (string.IsNullOrWhiteSpace(dbConnectionString)) |
|  | throw new ApplicationException("The database connection string is missing."); |
|  |  |
|  | builder.Services.AddSingleton(_ => |
|  | new PostgreSqlDataAccessor(dbConnectionString)); |


```

至此，`StickersController`就可以通过`PostgreSqlDataAccessor`来使用PostgreSQL数据库了，由于我们采用了合理的设计，因此在数据访问层实现了无缝替换，整个过程没有修改`StickersController`中的一行代码。


# 让程序运行起来


首先启动PostgreSQL数据库（我用docker启动）：



```


|  | $ cd docker |
| --- | --- |
|  | $ docker compose -f docker-compose.dev.yaml up |


```

然后，在Visual Studio 2022或者JetBrains Rider中，按下F5直接调试Stickers.WebApi应用程序，然后新建几条贴纸数据（为了减少篇幅，这里我就只建一条）：



```


|  | daxnet@daxnet-HP-ZBook:~/Projects/stickers/docker$ curl -X 'POST' \ |
| --- | --- |
|  | 'http://localhost:5141/stickers' \ |
|  | -H 'accept: */*' \ |
|  | -H 'Content-Type: application/json-patch+json' \ |
|  | -d '{ |
|  | "title": "测试贴纸", |
|  | "content": "这是一张测试贴纸。" |
|  | }' -v && echo |
|  | Note: Unnecessary use of -X or --request, POST is already inferred. |
|  | * Host localhost:5141 was resolved. |
|  | * IPv6: ::1 |
|  | * IPv4: 127.0.0.1 |
|  | *   Trying [::1]:5141... |
|  | * Connected to localhost (::1) port 5141 |
|  | > POST /stickers HTTP/1.1 |
|  | > Host: localhost:5141 |
|  | > User-Agent: curl/8.5.0 |
|  | > accept: */* |
|  | > Content-Type: application/json-patch+json |
|  | > Content-Length: 73 |
|  | > |
|  | < HTTP/1.1 201 Created |
|  | < Content-Type: application/json; charset=utf-8 |
|  | < Date: Tue, 22 Oct 2024 13:26:19 GMT |
|  | < Server: Kestrel |
|  | < Location: http://localhost:5141/stickers/7 |
|  | < Transfer-Encoding: chunked |
|  | < |
|  | * Connection #0 to host localhost left intact |
|  | {"id":7,"title":"测试贴纸","content":"这是一张测试贴纸。","createdOn":"2024-10-22T13:26:19.834994Z","modifiedOn":null} |


```

然后，调用获取贴纸API，每页2条记录，以贴纸创建时间做降序排列：



```


|  | daxnet@daxnet-HP-ZBook:~/Projects/stickers/docker$ curl "http://localhost:5141/stickers?sort=CreatedOn&asc=false&page=0&size=2" | jq |
| --- | --- |
|  | % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current |
|  | Dload  Upload   Total   Spent    Left  Speed |
|  | 100   310    0   310    0     0  91122      0 --:--:-- --:--:-- --:--:--  100k |
|  | { |
|  | "items": [ |
|  | { |
|  | "id": 7, |
|  | "title": "测试贴纸", |
|  | "content": "这是一张测试贴纸。", |
|  | "createdOn": "2024-10-22T13:26:19.834994Z", |
|  | "modifiedOn": null |
|  | }, |
|  | { |
|  | "id": 6, |
|  | "title": "this is a text", |
|  | "content": "test", |
|  | "createdOn": "2024-10-22T12:20:44.83564Z", |
|  | "modifiedOn": null |
|  | } |
|  | ], |
|  | "pageIndex": 0, |
|  | "pageSize": 2, |
|  | "totalCount": 5, |
|  | "totalPages": 3 |
|  | } |


```

# 题外话：数据库和数据表的初始化


有一个问题，就是如果我是第一次运行PostgreSQL数据库容器，或者是容器所依赖的卷被删掉了，再次运行PostgreSQL容器时，数据库和数据表都没有了，那我怎么去新建数据库和数据表呢？其实，PostgreSQL Docker镜像本身是支持数据库初始化脚本的，也就是在数据库正常启动之后，PostgreSQL会从`/docker-entrypoint-initdb.d`目录中按字母顺序逐个读入SQL文件然后逐一执行。于是，问题就变得非常简单了，就是把数据库和数据表的初始化SQL脚本文件放在这个目录下就行了，但是一定要注意：**SQL文件中的语句必须是幂等的**。


在docker\\postgresql目录下的Dockerfile中多加一行：



```


|  | FROM postgres:17.0-alpine |
| --- | --- |
|  | COPY *.sql /docker-entrypoint-initdb.d/ |


```

然后，新建一个名为`1_create_stickers_db.sql`的SQL文件，内容为：



```


|  | SET statement_timeout = 0; |
| --- | --- |
|  | SET lock_timeout = 0; |
|  | SET idle_in_transaction_session_timeout = 0; |
|  | SET transaction_timeout = 0; |
|  | SET client_encoding = 'UTF8'; |
|  | SET standard_conforming_strings = on; |
|  | SELECT pg_catalog.set_config('search_path', '', false); |
|  | SET check_function_bodies = false; |
|  | SET xmloption = content; |
|  | SET client_min_messages = warning; |
|  | SET row_security = off; |
|  |  |
|  | DROP TABLE IF EXISTS public.stickers; |
|  | SET default_tablespace = ''; |
|  | SET default_table_access_method = heap; |
|  |  |
|  | CREATE TABLE public.stickers ( |
|  | "Id" integer NOT NULL, |
|  | "Title" character varying(128) NOT NULL, |
|  | "Content" text NOT NULL, |
|  | "DateCreated" timestamp with time zone NOT NULL, |
|  | "DateModified" timestamp with time zone |
|  | ); |
|  |  |
|  |  |
|  | ALTER TABLE public.stickers OWNER TO postgres; |
|  |  |
|  | ALTER TABLE public.stickers ALTER COLUMN "Id" ADD GENERATED ALWAYS AS IDENTITY ( |
|  | SEQUENCE NAME public."stickers_Id_seq" |
|  | START WITH 1 |
|  | INCREMENT BY 1 |
|  | NO MINVALUE |
|  | NO MAXVALUE |
|  | CACHE 1 |
|  | ); |


```

重新编译Docker镜像重新启动PostgreSQL容器，此时如果该容器所使用的卷不存在的话，数据库和数据表就会自动创建。


# 总结


本文比较长，主要是从几个关键点介绍了`PostgreSqlDataAccessor`的实现过程中遇到的问题，然后针对`StickersController`切换到`PostgreSqlDataAccessor`以及数据库的初始化等内容进行了简单的介绍。下一讲将开始整合认证与授权部分，强烈建议有兴趣的读者先阅读下面的文章：


* 《[在Keycloak中实现多租户并在ASP.NET Core下进行验证](https://github.com)》
* 《[Keycloak中授权的实现](https://github.com)》
* 《[ASP.NET Core Web API下基于Keycloak的多租户用户授权的实现](https://github.com)》


# 源代码


本章节相关源代码在此：


[https://gitee.com/daxnet/stickers/tree/chapter\_3/](https://github.com)


下载源代码后，进入docker目录，然后编译并启动容器：



```


|  | $ docker compose -f docker-compose.dev.yaml build |
| --- | --- |
|  | $ docker compose -f docker-compose.dev.yaml up |


```

现在就可以直接用Visual Studio 2022或者JetBrains Rider打开stickers.sln解决方案文件，并启动Stickers.WebApi进行调试运行了。


