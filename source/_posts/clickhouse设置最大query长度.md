---
title: ClickHouse语法解析大揭秘
date: 2021-10-29 14:09:55
tags:
  - clickhouse
categories:
  - bigdata
---
> 首先来简单入门解决个小问题，那就是我们如何给ck传入更大的query呢？
### 通过TCP方式请求
> 通过tcp方式连接clickhouse，在会话session里先使用**set max_query_size=xx**的方式让当前这个会话修改query的长度，如下图
![clickhouse](/images/clickhouse/maxquerysize/1.png)

### 通过HTTP方式请求
> 通过http方式请求，http://ip:port/database?user=xx&password=yy&max_query_size=xx，ck会传递这个参数给setting重写
注意chproxy只允许最大max_query_size为512mb，超过此长度会直接报错
### 通过sql创建setting授权给登陆用户
1. 创建setting profile
```
create settings profile if not exists role_max_query_size SETTINGS max_query_size = 100000000000;
```
2. 将profile赋值给某个用户
```
grant role_max_query_size to prod_voyager_stats_events;
```

### 源码解析ck是如何处理max_query_size的
![clickhouse](/images/clickhouse/maxquerysize/2.png)
1. 如上图所示，在`HTTPHandler.cpp`下进行各种http的协议处理时，有一个变量叫**HTMLForm**类型的`params`，承载的是http请求里的`uri`，并且在代码的484行进行了此变量的处理，如下
```
for (const auto & [key, value] : params)
{
    if (key == "database")
    {
        if (database.empty())
            database = value;
    }
    else if (key == "default_format")
    {
        if (default_format.empty())
            default_format = value;
    }
    else if (param_could_be_skipped(key))
    {
    }
    else
    {
        /// Other than query parameters are treated as settings.
        if (!customizeQueryParam(context, key, value))
            settings_changes.push_back({key, value});
    }
}
```
2. 而`customizeQueryParam`会判断该参数是否param是否等于query，如果是则不会进入setting的设置，再判断是否是param_开头的如果是则会传入context（理解为这次session会话中需要设置的各种上下文内容）则也不会进行setting处理，不是前面2个case则进行setting处理，重载系统默认的setting里的参数，如下图
![clickhouse](/images/clickhouse/maxquerysize/3.png)
3. 虽然第二步已经设置了setting，但注意代码的512行，这行代码会走向**SettingsConstraints.cpp**类的**checkImpl**校验逻辑里，有一些配置是不允许修改的，例如**profile**，例如配置就是如果已经通过grant授权配置了**setting profile**了，会去看这个用户的相关权限，如果不符合则会直接抛出exception，不再进行处理，注意这里还有一个问题，抛了异常后，不再将此次请求写入到system.query_log中，之后我们会修复此问题
```
/// For external data we also want settings
context.checkSettingsConstraints(settings_changes);
context.applySettingsChanges(settings_changes);
```
4. 做完setting的约束校验后，都符合条件，则我们已经重载了setting里的**max_query_size**，之后就走入了**executeQuery.cpp**执行query逻辑，在它的构造函数里我们就可以看到query是根据**max_query_size**来读取的，如下图
![clickhouse](/images/clickhouse/maxquerysize/4.png)

### 源码解析ck是如何处理settings的
介绍完了**max_query_size**的处理逻辑后，其实我们已经大致明白ck在query的处理流程是如何流转的，那么现在问题来了，我们知道可以通过`select xx from tb SETTINGS max_query_size=12112`这种方式传入自定义的setting参数，但是有些参数有生效，而select语法却对max_query_size不生效，原因是什么呢？好了，别着急现在我们就来解答ck是如何处理setting这层逻辑的。
1. 为什么max_query_size的select中不生效？
![clickhouse](/images/clickhouse/maxquerysize/5.png)
原因很简单，只要我们读过了上面的流转过程，就知道max_query_size这个参数的处理系统默认是256kb，那么如果未通过uri方式传入**max_query_size**，则在截取query长度前，默认都是256kb，注意截取query时是还未进行ck的parser逻辑处理的，我们可以在而query里的setting是需要经过ck的parser解析后，才会重载进去(如下图6)，所以呢如果你的select query在256kb范围内，则截取完整query后，经过ck的parser解析出ast树，是会带上新的setting，但此时已经没有意义了，而相反的如果你的query超过了256kb，则只截取到256kb前的query，此时setting也不会走到**ParserSelectQuery**里，同时因为你的query被不完整截取后，会直接报ast语法错误
![图6](/images/clickhouse/maxquerysize/6.png)

2. 那么其他的ddl和dml都有哪些是支持setting的呢？
#### 1. HTTPHandler.cpp => processQuery
> 每一个http请求都在clickhouse都会起一个叫**HTTPHandler**的线程去处理，根据http请求header和body，初始化请求上下文环境：包括session、用户信息、当前database、响应信息等，另外还处理限流，用户权限，根据配置取到setting信息进行设置，本文重点是调用`executeQuery`方法处理`query`
```
executeQuery(*in, *used_output.out_maybe_delayed_and_compressed, /* allow_into_outfile = */ false, context,
        [&response] (const String & current_query_id, const String & content_type, const String & format, const String & timezone)
        {
            response.setContentType(content_type);
            response.add("X-ClickHouse-Query-Id", current_query_id);
            response.add("X-ClickHouse-Format", format);
            response.add("X-ClickHouse-Timezone", timezone);
        }
    );
```

#### 2. executeQuery.cpp => executeQuery
从流中读出字节到buffer里，根据设置的`max_query_size`判断buffer是否已满，复制到LimitReadBuffer里，重点是执行**executeQueryImpl**，返回tuple类型的(ast, stream)，从stream里提取出**pipeline(流水线)**，根据ast构造出`IBlockInputStream`或者`IBlockOutputStream`，传给pipeline后执行pipeline的execute方法
```
std::tie(ast, streams) = executeQueryImpl(begin, end, context, false, QueryProcessingStage::Complete, may_have_tail, &istr);
```

#### 2. executeQuery.cpp => executeQueryImpl
按照解析出的ast，构造出Interpreter，调用Interpreter的exec方法去执行后返回pipeline，执行结果记录到query_log里，最后把构造出对应的ast和pipeline返回
```
// 这里是实现了ParserQuery对象，继承了IParserBase，IParserBase继承自IParser，等下走到6时，才知道虚函数parseImpl会通过ParserQuery对象实现
ParserQuery parser(end, settings.enable_debug_queries);
.........
ast = parseQuery(parser, begin, end, "", max_query_size, settings.max_parser_depth);
.........
auto interpreter = InterpreterFactory::get(ast, context, stage);
.........
res = interpreter->execute();
QueryPipeline & pipeline = res.pipeline;
.........
return std::make_tuple(ast, std::move(res));
```

#### 3. parseQuery.cpp => parseQueryAndMovePosition
```
ASTPtr res = tryParseQuery(parser, pos, end, error_message, false, query_description, allow_multi_statements, max_query_size, max_parser_depth);
```

#### 4. parseQuery.cpp => tryParseQuery
> 尝试解析SQL，将sql通过语法树规则装入TokenIterator，返回ASTPtr
```
// ClickHouse词法分析器是由Tokens和Lexer类来实现，token是最基础的元祖，之间是没有任何关联的，只是一堆词组和符号，通过lexer语法进行解析后，把元祖里的token建立起关系。
Tokens tokens(pos, end, max_query_size);
IParser::Pos token_iterator(tokens, max_parser_depth);
// 注意这里，TokenIterator对->使用了重载，在重载函数里去初始化TOKEN，主要是从第一个字符开始使用pos++的方式进行判断，可以进入Token Lexer::nextTokenImpl()进行查看
if (token_iterator->isEnd() || token_iterator->type == TokenType::Semicolon) {
    out_error_message = "Empty query";
    pos = token_iterator->begin;
    return nullptr;
}
.....
Expected expected;
......
ASTPtr res;
bool parse_res = parser.parse(token_iterator, res, expected);
```
> 注意：`IParser`的`parse`方法是`virtual`虚函数，`IParser`作为接口角色，被`IParserBase`继承，在`IParserBase`里实现了`parse`方法。

#### 5. IParserBase.cpp => parse
> 在解每个token时都会根据当前的token进行预判（parseImpl返回的结果），返回true才会进入下一个子token
```
bool IParserBase::parse(Pos & pos, ASTPtr & node, Expected & expected)
{
    expected.add(pos, getName());

    return wrapParseImpl(pos, IncreaseDepthTag{}, [&]
    {
        bool res = parseImpl(pos, node, expected);
        if (!res)
            node = nullptr;
        return res;
    });
}
```
> 注意到parseImpl在IParserBase中是一个虚函数，将被继承自IParserBase类的子类实现，而在 *** 第2步 ***中我们定义的子类是ParserQuery，所以此时是直接调用到ParserQuery子类的parseImpl方法

#### 6. ParserQuery.cpp => parseImpl
> Parser的主要类（也都是继承自IParserBase）分别定义出来后，每个去尝试解析，如果都不在这几个主要Parser里，则返回false，否则返回true，clickhouse把query分类成以下14类，但本质上可以归纳为2类，第一类是有结果输出可对应show/select/desc/create等，第二类是无结果输出可对应insert/use/set等
```
bool ParserQuery::parseImpl(Pos & pos, ASTPtr & node, Expected & expected)
{
    ParserQueryWithOutput query_with_output_p(enable_explain);
    ParserInsertQuery insert_p(end);
    ParserUseQuery use_p;
    ParserSetQuery set_p;
    ParserSystemQuery system_p;
    ParserCreateUserQuery create_user_p;
    ParserCreateRoleQuery create_role_p;
    ParserCreateQuotaQuery create_quota_p;
    ParserCreateRowPolicyQuery create_row_policy_p;
    ParserCreateSettingsProfileQuery create_settings_profile_p;
    ParserDropAccessEntityQuery drop_access_entity_p;
    ParserGrantQuery grant_p;
    ParserSetRoleQuery set_role_p;
    ParserExternalDDLQuery external_ddl_p;

    bool res = query_with_output_p.parse(pos, node, expected)
        || insert_p.parse(pos, node, expected)
        || use_p.parse(pos, node, expected)
        || set_role_p.parse(pos, node, expected)
        || set_p.parse(pos, node, expected)
        || system_p.parse(pos, node, expected)
        || create_user_p.parse(pos, node, expected)
        || create_role_p.parse(pos, node, expected)
        || create_quota_p.parse(pos, node, expected)
        || create_row_policy_p.parse(pos, node, expected)
        || create_settings_profile_p.parse(pos, node, expected)
        || drop_access_entity_p.parse(pos, node, expected)
        || grant_p.parse(pos, node, expected)
        || external_ddl_p.parse(pos, node, expected);

    return res;
}
```
注意看这个parseImpl方法，进来后都会先去接`ParserQueryWithOutput`类解析相关ast，这里类涉及到了`explain`、`select`、`show`，`create`、`alter`等相关语法的解析，如果解析不过，则直接报错，解析成功后会处理我们这篇文章中提到的**SETTING**，如下图7定义的，将setting传入的变量存入到`s_settings`指针中。
![图7](/images/clickhouse/maxquerysize/7.png)

clcikhouse的parser总结就是：
+ 1. ClickHouse词法分析器
词法解析的主要任务是读入源程序的输入字符、将它们组成词素，生成并输出一个词法单元（Token）序列，每个词法单元对应于一个词素。`ClickHouse`中的每个词法单元（`Token`）使用一个`struct Tocken`结构体对象来进行存储，结构体中存储了词法单元的`type`和`value`。
ClickHouse词法分析器是由`Tokens`和`Lexer`类来实现， ***DB::Lexer::nextTokenImpl()***函数用来对`SQL`语句进行词法分析的具体实现

+ 2. ClickHouse语法解析器
ClickHouse中定义了不同的Parser用来对不同类型的SQL语句进行语法分析，例如：ParserInsertQuery（Insert语法分析器）、ParserCreateQuery（Create语法分析器）、ParserAlterQuery（Alter语法分析器）等等。
Parser首先判断输入的Token序列是否是该类型的SQL，若是该类型的SQL，则继续检查语法的正确性，正确则生成AST返回，语法错误的则抛出语法错误异常，否则直接返回空AST语法树

![clickhouse](/images/clickhouse/parser/1.png)

#### 解答setting生效问题
![图7](/images/clickhouse/maxquerysize/8.png)
如上图所示，当前原生ck只支持InterpreterSelectQuery和InterpreterInsertQuery对query传入setting进行了重载处理。
InterpreterSelectQuery是在自己的构造函数里初始化了setting到context里
```
void InterpreterSelectQuery::initSettings()
{
    auto & query = getSelectQuery();
    if (query.settings())
        InterpreterSetQuery(query.settings(), *context).executeForCurrentContext();
}
```
InterpreterInsertQuery是在parser解析出ast后，在`executeQueryImpl`进行的setting重载context。
```
auto * insert_query = ast->as<ASTInsertQuery>();

if (insert_query && insert_query->settings_ast)
    InterpreterSetQuery(insert_query->settings_ast, context).executeForCurrentContext();
```
剩下的setting都已经是通过Interpreter执行结束后再处理的，对于我们需要在前置传入时没有效果了。

### 解决业务困扰
当前我们的离线导入hive2ck，其实是通过将数据写入到临时表，这张临时表是按照目标表的表结构重新创建了一个MergeTree表，通过spark任务将hive数据以流式方式写入到临时表分区，生产出分区对应的多个part，生产过程中我们会将part拉回到临时表对应的detach目录，这个过程叫[离线构建](http://way.xiaojukeji.com/article/29557)，再将再将part通过ck的attach命令激活，这时候临时表就对该分区可见了，然后再通过replace partition的方式，将临时表的分区替换到我们的目标表去，这整个过程，就是我们的hive2ck处理流程，如下图所示：
![图10](/images/clickhouse/maxquerysize/9.png)

> 我们将此离线构建继承到数梦的同步中心后，陆续遇到了业务方来咨询相关问题，下面是问题汇总及如何解决的
#### 如何在离线导入中将明细数据写入到关联的物化视图
熟悉clickhouse的同学们都知道，原生ck对于物化视图的写入，唯一的方式是在明细表通过insert写入时，才会将数据经过物化视图的触发器写入关联的物化视图，而在离线构建过程中，ck是不支持的，但很多业务方跟我们提出这个需求，希望离线构建可以支持将分区数据写入到关联物化视图去，于是我们对ck的replace partition 进行了改造。
改造前的语法：
```
ALTER TABLE test.visits_basic REPLACE PARTITION '20221102' FROM test.visits_basic_tmp;
```
改造后的语法：
```
ALTER TABLE test.visits_basic REPLACE PARTITION '20221101' FROM test.visits_basic_tmp AND TRIGGER VIEW;
```
在离线构建走到替换分区这一步时，我们改造了AstAlterQuery，让`ParserAlterQuery`增加了对`and trigger view`的语法解析，解析之后进入到`InterpreterAlterQuery`时，如果ast返回的trigger view是true，则程序流程会流转到取出明细表元数据，查询是否有关联物化视图，重新构造出类似下面的sql，交过pipeline进行执行，由此将该分区数据写入到物化视图去。
```
insert into 物化视图 select from 明细表 where 分区=xx
```

#### 分区过大导入失败如何解决
```
xxx has timed out! (120000ms) Possible deadlock avoided. Client should retry
```
![图10](/images/clickhouse/maxquerysize/10.png)
如上图所示，替换分区前，会给该明细表加一把锁，并设定锁时间（lock_acquire_timeout），系统默认时120s，如果该分区过大，替换过程超过120s，则会爆上面错误，而本文最开始已经讲解过如何处理setting，考虑到ck原生只支持insert和select时interpreter对setting重载，由此进行改造让InterpreterAlterQuery也支持通过sql传入锁时间，如下面语法：
```
ALTER TABLE test.visits_basic REPLACE PARTITION '20221108' FROM test.visits_basic_tmp AND TRIGGER VIEW SETTINGS lock_acquire_timeout=86400000;
```
因为ParserQueryWithOutput已经对setting进行了解析，而AstAlterQuery其实是继承自ASTQueryWithOutput，所以我们已经获得了setting这一块的ast，无需再自己初始化新的ast，只要在InterpreterAlterQuery里把setting重载就行了，如图11
![图10](/images/clickhouse/maxquerysize/11.png)

#### 替换分区成功，物化视图数据写入报错如何解决
1. 首先我们在数梦平台上控制了相关的ddl语句修改，如果用户要删明细表字段，则必须先去处理关联的物化视图字段，如果用户要删明细表，则必须先删物化视图
2. 遇到替换分区成功，而物化视图写入失败，报错都是锁明细表超时，对于这种case可直接解锁明细表的锁，让物化视图自己去写，不再锁明细表，所以只需要做简单的锁释放便可



以上是对ck进行改造过程中遇到的3个问题，此改造过程主要是满足离线导入可写入物化视图，未来我们还将对ck进行更多改造，以满足不同业务需求，各个 业务线大佬们如果在使用ck过程中有遇到任何问题，欢迎加入ck用户群，和我们一起沟通解决。