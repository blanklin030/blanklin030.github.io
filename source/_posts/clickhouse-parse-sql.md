---
title: 聊一聊clickhouse解析SQL的流程
date: 2022-02-30 22:57:11
tags:
  - clickhouse
categories:
  - bigdata
---

### 背景介绍
> 由于源码较多，只抽出具体实现函数进行源码讲解，本次讲解基于clickhouse v20.6.3.28-stable（该版本与最新版出入较大）。
> 目前滴滴的`clickhouse`对用户侧都是开放的`http`连接，所以以下我们都用`http`方式进行代码详解，举例场景如下：
```
curl --location --request POST 'http://localhost:8024?user=xx&password=xx&database=default' \
--header 'Content-Type: text/plain' \
--header 'Cookie: uid-walle-de=d5dd9c3598911e200270e07a680e1e200002772000' \
--data-raw 'EXPLAIN SELECT a,b FROM test'
```

### 源码解析
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
    .......
    ParserExternalDDLQuery external_ddl_p;

    bool res = query_with_output_p.parse(pos, node, expected)
    .......
        || external_ddl_p.parse(pos, node, expected);

    return res;
}
```
> 因为我们第一步假设的语法是explain，所以接下来会进入的ParserQueryWithOutput类调用parseImpl进行解析
#### 7. ParserQueryWithOutput.cpp=> parseImpl
> 又是一个ParserExplainQuery继承自IParserBase，又再次进入parse函数
```
......
ParserExplainQuery explain_p(enable_debug_queries);

ASTPtr query;

bool parsed = explain_p.parse(pos, query, expected)
....
```

#### 8. ParserExplainQuery.cpp => parseImpl
```
bool ParserExplainQuery::parseImpl(Pos & pos, ASTPtr & node, Expected & expected)
{
    // 定义ExplainKind类型
    ASTExplainQuery::ExplainKind kind;
    bool old_syntax = false;
    // 定义关键词
    ParserKeyword s_explain("EXPLAIN");
   ...............
   // 按照定义的关键词去进行case解析
    else if (s_explain.ignore(pos, expected))
    {
        kind = ASTExplainQuery::QueryPlan;

        if (s_ast.ignore(pos, expected))
            kind = ASTExplainQuery::ExplainKind::ParsedAST;
            ..........
        else if (s_plan.ignore(pos, expected))
            kind = ASTExplainQuery::ExplainKind::QueryPlan;
    }
    else
        return false;
}
```

#### 9. ParserKeyword.cpp => parseImpl
> TokenType::BareWord是指{‘a’… ‘z’| ‘A’ … ‘Z’| ‘_’}+，这里就是从第一个字母开始盘点，直到空格前，组成一个词法单元Token，空格之后，再重复解出下一个Token
```
bool ParserKeyword::parseImpl(Pos & pos, ASTPtr & /*node*/, Expected & expected)
{
    // 不是大小写字母或者_则直接返回false
    if (pos->type != TokenType::BareWord)
        return false;
    //取开头字符
    const char * current_word = s;
    // 判断字符长度
    size_t s_length = strlen(s);
    if (!s_length)
        throw Exception("Logical error: keyword cannot be empty string", ErrorCodes::LOGICAL_ERROR);
    //取最后字符
    const char * s_end = s + s_length;
    //死循环
    while (true)
    {
        expected.add(pos, current_word);//当前字符加入到变量收集器里
        if (pos->type != TokenType::BareWord)//再次判断该Token是否BareWord类型
            return false;
        //找到第一个空格
        const char * next_whitespace = find_first_symbols<' ', '\0'>(current_word, s_end);
        size_t word_length = next_whitespace - current_word;//找到空格的位置
        //判断空格位置是否等于Token长度
        if (word_length != pos->size())
            return false;
        //忽略大小写比较字符串是否一致
        if (0 != strncasecmp(pos->begin, current_word, word_length))
            return false;

        ++pos;

        if (!*next_whitespace)
            break;

        current_word = next_whitespace + 1;
    }

    return true;
}
```
#### 源码流程总结
> 以下是ClickHouse中token的类型列表：  

| name              | description            |
|  :----: |:----:|
| Whitespace | {‘ ’&#124;‘\t’&#124;‘\n’&#124;‘\r’&#124;‘\f’&#124; ‘\v’}+ |
| Comment | //     /*…*/ |
| BareWord |   {‘a’… ‘z’&#124; ‘A’ … ‘Z’&#124; ‘_’}+  |
|  Number |  二进制、八进制、十进制、十六进制等 |
| StringLiteral |  \’ |
| QuotedIdentifier | ‘    “ |
| OpeningRoundBracket |  ( |
| ClosingRoundBracket | ) |
| OpeningSquareBracket | [ |
| ClosingSquareBracket | ] |
| Comma | , |
| Semicolon | ; |
| Dot | . |
| Asterisk |* |
| Plus |  + |
| Minus | - |
| Slash | / |
| Percent | % |
| Arrow |  -> |
| QuestionMark | ? |
|Colon | : |
| Equals | == |
| NotEquals | !=    <> |
| Less | < |
| Creater | > |
| LessOrEquals | <= |
| GreaterOrEquals | >=  |
| Concatenation |  &#124;&#124; |

+ 1. ClickHouse词法分析器
词法解析的主要任务是读入源程序的输入字符、将它们组成词素，生成并输出一个词法单元（Token）序列，每个词法单元对应于一个词素。`ClickHouse`中的每个词法单元（`Token`）使用一个`struct Tocken`结构体对象来进行存储，结构体中存储了词法单元的`type`和`value`。
ClickHouse词法分析器是由`Tokens`和`Lexer`类来实现， ***DB::Lexer::nextTokenImpl()***函数用来对`SQL`语句进行词法分析的具体实现

+ 2. ClickHouse语法解析器
ClickHouse中定义了不同的Parser用来对不同类型的SQL语句进行语法分析，例如：ParserInsertQuery（Insert语法分析器）、ParserCreateQuery（Create语法分析器）、ParserAlterQuery（Alter语法分析器）等等。
Parser首先判断输入的Token序列是否是该类型的SQL，若是该类型的SQL，则继续检查语法的正确性，正确则生成AST返回，语法错误的则抛出语法错误异常，否则直接返回空AST语法树

![clickhouse](/images/clickhouse/parser/1.png)

### 自定义SQL

#### 背景介绍
> 在普通表的使用中，我们默认给用户创建2张表，分别如下：
+ 1. tb（Distributed）
该表是分布式表，不存储数据，而是通过读取或写入其他远端节点上的表进行数据处理的表引擎。该表引擎需要依赖各个节点的本地表来创建，本地表的存在是Distributed表创建的依赖条件。

+ 2. tb_local(MergeTree)
该表是数据存储表，在每个shard记录一部分数据，所以不允许用户使用_local表进行查询，这样只能查到某个shard，而不能聚合所有shard数据。

> 在物化视图的使用中，我们默认给用户创建了3张表，分别如下：
+ 1. tb_view（Distributed）
该表是分布式表，不存储数据，而是通过读取或写入其他远端节点上的表进行数据处理的表引擎。该表引擎需要依赖各个节点的本地表来创建，本地表的存在是Distributed表创建的依赖条件。

+ 2. tb_view_inner_local(AggregatingMergeTree/UniqueMergeTree/ReplacingMergeTree/MergeTree)
该表是数据存储表，在每个shard记录一部分数据，每一次写入实际上是由触发器触发到分布式表后，分布式表按配置确认是同步写入还是异步写入
    1.同步写
    在当前分片本地写入完成后，会等待所有分片写入完毕，才会返回写入成功的消息。
    2.异步写
    在当前分片本地写入完成后，即返回写入成功的消息。

+ 3. tb_view_local(Materialized View)
该表是触发器，在每一次底表写入（insert）时产生触发作用，把创建时对应sql聚合出的数据再次写入到对应的物化视图。

由于原生clickhouse的触发器只在insert时产出触发，但目前写入clickhouse的source有相当一部分来源于同步任务，接下来简单介绍下同步任务的原理
+ 创建一张跟底表结构一样的临时表tb_tmp_local
+ 指定source的分区数据清洗后，insert该分区数据写入到tb_tmp_local
+ 通过replace partition语句将临时表的分区最终替换到tb_local

由于是通过replace partition方式，则无法将该底表分区数据再次触发写入给物化视图，所以我们决定在对replace partition进行改造，以实现同步触发功能，以下是改造过程。

#### 改造 replace partition 语句
```
//  and trigger view是我们改造的部分
alter table tb replace partition '20220202' from tb_tmp_local and trigger view;
```
此处改造主要是修改`ParserAlterQuery`部分，增加对**and trigger view**语法的解析，用以产生正确的AST语法树
```
ParserKeyword s_trigger_view("AND TRIGGER VIEW");
if (s_trigger_view.ignore(pos, expected)) {
    query->need_trigger_view = true;
    return true;
}
```
#### 改造解释器
