# Automated Testing in Low-Code Platforms

Automated testing is a critical component of modern software development practices, but its maintenance costs are often high, and it is generally difficult to achieve high automated testing coverage in projects with limited investment resources. It's only natural that someone would want to simplify the writing and maintenance of test cases with a visual, low-code platform. But ** The essential difficulty of automated test maintenance is not the problem of visualization, but the vulnerability of test cases. **.. In general, we write test cases that take an external perspective, providing input, calling functions, and then checking the returned results. However, business functions are rarely so-called pure functions, and their execution inevitably involves a large number of side effects, such as reading and writing databases, concurrent access, random number generation, and so on. As a result, the return result of calling the same test case with the same input parameters may be uncertain. For example, if the account balance decreases after executing the transfer operation, the same transfer operation may fail if executed again. To overcome this uncertainty, we were forced to write a lot of data initialization code by hand and write the result check in some form of fuzzy matching. Because this process is very lengthy, it is difficult for us to achieve very rigorous, especially when there is dirty data and frequent changes in data structures, test cases based on external perspectives are particularly fragile. This article will introduce the NopAutoTest automated testing framework used in the Nop platform, which is fully integrated with the Nop platform and co-designed with the back-end application automated testing framework. It makes full use of all kinds of model information in Nop platform, and effectively alleviates the vulnerability of test cases through a series of means such as recording and playback, data-driven, model transformation and so on. Refer to the nop-auto-test and nop-match modules for specific implementation.

[nop-autotest](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-autotest)

[nop-match](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-match)

## One. Specificity of low-code platforms

The essential particularity of the low-code platform is not the visual interface it provides, but its inherent logical structure of modeling. If a low-code platform is adequately modeled, then ** All side effects should be observable. **.


```java
  数据 = 输入 + 输出 + 副作用
```

If the observed side-effect data is added to the input-output data set, we get a complete set of information about the system ** Eliminate all the unknown side effects, restore the test cases to pure functions with complete certainty, and overcome the vulnerability problem caused by incomplete information. **.

Common side effects in application development include the following

1. Database read and write
   
   The behavior of an application system can vary greatly based on the state of the data in the database. Moreover, in addition to the result data returned by the interface, the impact of business operations on the entire system may be more reflected in the modification of core business data in the database. If we need to verify that the business implementation is correct, in general, in addition to checking the interface result data, the tester also needs to execute the database verification script to confirm that the data state in the database meets the integrity requirements.

2. Random number and time

In the program code, it is inevitable to use the system clock to record the current operation time, and to randomly generate the card number, order number, primary key ID and other variable data that are not repeated in each execution.

3. Asynchronous processing
   
   After the business interface returns the result, some asynchronous tasks may still be executed, and the final system state can be stabilized after a period of time.

4. Cache read and write Cache read and write is similar to database read and write. However, the cache is generally an optional component in the business. When verifying the correctness of the core business, you can consider disabling the cache or actively emptying the cache.

5. External infrastructure and environment
   
   When the system runs normally, it may need to rely on external infrastructure, such as registry, message queue, third-party services, etc. It may also have certain requirements for the configuration of the external network environment.

In the Nop platform, all business objects are managed through a dependency injection container, and all operations with side effects are exposed through a modeled engine or service interface. In this case, it provides the following technical support for automated testing:

1. Recording and playback of database reading and writing
   
   The NopOrm data access engine is a complete ORM engine like Hibernate that can log all database records read and modified by the application. When the test case is executed for the first time, the database reading and writing data will be recorded. After the snapshot execution mode is enabled, the test framework will use the recorded data to create a database table and insert an initialization data record. At the same time, after the test case is run, it will automatically verify that the database modifications are consistent with the recorded modifications.
   
   > Even if the batch update is executed through the EQL statement update XX set YY = ZZ, the ORM engine can parse the EQL statement and obtain the data records before and after modification.

2. Variable labeling of random numbers and times
   
   Data that changes with each execution, such as random numbers and time, can be marked as AutoTestVariables, and the testing framework is responsible for tracking the propagation of these variables. In particular, we can automatically identify all foreign key variables that reference variable data according to the primary foreign key relationship defined on the ORM model. In the final data validation process, the validation condition becomes a variable match

   
   ```java
     checkMatch("@var:NopAuthSession@sessionId", visitLog.sessionId)
   ```

3. The test framework waiting for asynchronous processing completion can monitor the execution state of the asynchronous processing queue and wait for all transient processing processes to be completed, so as to obtain a deterministic result meeting the final consistency.

   
   ```java
    waitUnti(()-> taskService.isAllProcessed(), 1000);
   ```

4. The integrated docker environment test framework can integrate the docker environment through the [ testcontainers ](https://www.testcontainers.org) library, and put some infrastructure required for testing into the docker environment to run.

## Two. Data-driven testing

The NopAutoTest testing framework is a data-driven testing framework, which means that in general, we do not need to write any code to prepare the input data and verify the output results, but only need to write a skeleton function and provide a batch of test data files. Let's look at an example.

[nop-auth/nop-auth-service/src/test/io/nop/auth/service/TestLoginApi.java](https://gitee.com/canonical-entropy/nop-entropy/blob/master/nop-auth/nop-auth-service/src/test/io/nop/auth/service/TestLoginApi.java)

[nop-auth/nop-auth-service/cases/io/nop/auth/service/TestLoginApi](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-auth/nop-auth-service/cases/io/nop/auth/service/TestLoginApi)


```java
class TestLoginApi extends JunitAutoTestCase {
   // @EnableSnapshot
    @Test
    public void testLogin() {
        LoginApi loginApi = buildLoginApi();

         //ApiRequest<LoginRequest> request = request("request.json5", LoginRequest.class);
        ApiRequest<LoginRequest> request = input("request.json5", new TypeReference<ApiRequest<LoginRequest>>(){}.getType());

        ApiResponse<LoginResult> result = loginApi.login(request);

        output("response.json5", result);
    }
}
```

The test case inherits from the JunitAutoTestCase class and is then used `input(fileName, javaType)` to read the external data file and cast the data to the type specified by javaType. The specific data format is determined by the suffix of the file name, which can be JSON/json5/yaml, etc.

After calling the function under test, save the result data to an external data file through output (fileName, result) instead of writing the result verification code.

### 2.1 Recording mode

When testLogin is executed in recording mode, the following data file is generated


```
TestLoginApi
    /input
       /tables
          nop_auth_user.csv
          nop_auth_user_role.csv
       request.json5    
    /output
       /tables
          nop_auth_session.csv
      response.json5    
```

All database records that have been read will be recorded in the/input/tables directory, and each table corresponds to a CSV file.

> Even if no data is read, the corresponding empty file is generated. Because in validation mode, you need to determine which tables need to be created in the test database based on the table names recorded here.

If we open the response.json5 file, we can see the following


```
{
  "data": {
    "accessToken": "@var:accessToken",
    "attrs": null,
    "expiresIn": 600,
    "refreshExpiresIn": 0,
    "refreshToken": "@var:refreshToken",
    "scope": null,
    "tokenType": "bearer",
    "userInfo": {
      "attrs": null,
      "locale": "zh-CN",
      "roles": [],
      "tenantId": null,
      "timeZone": null,
      "userName": "auto_test1",
      "userNick": "autoTestNick"
    }
  },
  "httpStatus": 0,
  "status": 0
}
```

Note that accessToken and refreshToken have been automatically replaced with variable matching expressions. This process requires no manual intervention by programmers at all.

As for the recorded NOP _ auth _ session. CSV, it is as follow


```csv
_chgType,SID,USER_ID,LOGIN_ADDR,LOGIN_DEVICE,LOGIN_APP,LOGIN_OS,LOGIN_TIME,LOGIN_TYPE,LOGOUT_TIME,LOGOUT_TYPE,LOGIN_STATUS,LAST_ACCESS_TIME,VERSION,CREATED_BY,CREATE_TIME,UPDATED_BY,UPDATE_TIME,REMARK
A,@var:NopAuthSession@sid,067e0f1a03cf4ae28f71b606de700716,,,,,@var:NopAuthSession@loginTime,1,,,,,0,autotest-ref,*,autotest-ref,*,
```

The first column, _ chgType, indicates the type of data change, A-Add, U-Modify, D-Delete. The randomly generated primary key has been replaced with a variable match expression `@var:NopAuthSession@sid`. At the same time, according to the information provided by the ORM model, the createTime field and the updateTime field are bookkeeping fields, which do not participate in data matching verification, so they are replaced by *, indicating that any value is matched.

### 2.2 Validation mode

When the testLogin function is executed successfully, we can open `@EnableSnapshot` the annotation to convert the test case from recording mode to verification mode. In verification mode, the test case performs the following actions during the setUp phase:

1. Adjust configuration such as jdbcUrl to force use of local in-memory database (H2)
2. Load the json5 file of the input/init _ vars. And initialize the variable environment (optional)
3. Collect the corresponding table names under the directories of input/tables and output/tables, and generate and execute the corresponding table creation statements according to the ORM model
4. Execute all the XXX. SQL script files under the input directory to customize the initialization of the new database (optional).
5. Insert data from the input/tables directory into the database

If the output function is called during the execution of the test case, the output JSON object is compared with the recorded data pattern file based on the MatchPattern mechanism. See the introduction in the next section for the specific comparison rules. If you expect the test function to throw an exception, you can use the error (fileName, runnable) function to describe it.


```java
@Test
public void testXXXThrowException(){
   error("response-error.json5",()-> xxx());
}
```

During the teardown phase, the test routine automatically performs the following operations:

1. Compare the data changes defined in output/tables with the state in the current database to determine if they match.
2. (Optional) Execute the validation SQL defined in the SQL _ check. Yaml file and compare with the expected result.

### 2.3 Test updates

If the code is modified later and the returned result of the test case changes, we can temporarily set the saveOutput property to true to update the recorded result under the output directory.


```java
@EnableSnapshot(saveOutput=true)
@Test
public void testLogin(){
    ....
}
```

## Three. Object Pattern Matching Based on Prefix-Guided Syntax

In the previous section, the matching criteria in the data template file used for matching only contain fixed values and variable expressions `@var:xx`, where the variable expressions use the so-called prefix-guided syntax (see my article [ DSL Layered Syntax Design and Prefix-Guided Syntax](https://zhuanlan.zhihu.com/p/548314138) for details), which is an extensible domain-specific syntax (DSL) design. First, we note that the `@var:` prefix can be extended to more cases, such as `@ge:3` meaning greater than or equal to 3. Second, this is an open design. ** We can always add more syntax support, and we can ensure that there are no syntax conflicts between them. **。 Third, this is a localized embedded syntax design, `String->DSL` a transformation that can enhance arbitrary strings into executable expressions, such as field matching conditions in a CSV file. Let's look at a more complex matching configuration.


```json
{
  "a": "@ge:3",
  "b": {
    "@prefix": "and",
    "patterns": [
      "@startsWith:a",
      "@endsWith:d"
    ]
  },
  "c": {
    "@prefix": "or",
    "patterns": [
      {
        "a": 1
      },
      [
        "@var:x",
        "s"
      ]
    ]
  },
  "d": "@between:1,5"
}
```

In `@prefix` this example, and/or matching conditions with complex structures are introduced. Similarly, we can introduce `if` `switch` the equal-conditional branch.


```json
{
    "@prefix":"if"
    "testExpr": "matchState.value.type == 'a'",
    "true": {...}
    "false": {...}
}
{
    ”@prefix":"switch",
    "chooseExpr": "matchState.value.type",
    "cases": {
       "a": {...},
       "b": {...}
    },
    "default": {...}
}
```

TestExpr is an XLang expression, where matchState corresponds to the current matching context object, and the data node currently being matched can be obtained through value. Depending on the return value, either the true or false branch is selected for matching.

Here "@ prefix" corresponds to the explode pattern of the prefix-guided syntax, which expands the DSL into an abstract syntax tree in Json format. If direct embedding of JSON is not allowed due to data structure limitations, such as when used in CSV files, we can still use the standard form of the prefix bootstrap syntax.


```
@if:{testExpr:'xx',true:{...},false:{...}}
```

Just convert the parameter corresponding to if into a string through JSON encoding, and then concatenate the `@if:` prefix.

The syntax design of prefix-guided syntax is very flexible, and it does not require the syntax format of different prefixes to be completely unified. For example, `@between:1,5` represents greater than or equal to 1 and less than or equal to 5. The data format after the prefix is only recognized by the parser of the prefix pair, and we can design the corresponding simplified syntax according to the situation.

If you want to verify that only some of the fields in an object meet the matching criteria, you can use the symbol `*` to indicate that the other fields are ignored


```json
{
    "a":1,
    "*": "*"
}
```

## Four. Multi-step correlation test

If we want to test multiple related business functions, we need to pass correlation information between multiple business functions. For example, after logging in to the system, the accessToken is obtained, and then the accessToken is used to obtain the user details. After other business operations are completed, the access token is passed as a parameter, and logout is called to exit.

Because there is a shared AutoTestVars context, correlation information can be automatically passed between business functions through AutoTestVariables. For example


```java
    @EnableSnapshot
    @Test
    public void testLoginLogout() {
        LoginApi loginApi = buildLoginApi();

        ApiRequest<LoginRequest> request = request("1_request.json5", LoginRequest.class);

        ApiResponse<LoginResult> result = loginApi.login(request);

        output("1_response.json5", result);

        ApiRequest<AccessTokenRequest> userRequest = request("2_userRequest.json5", AccessTokenRequest.class);

        ApiResponse<LoginUserInfo> userResponse = loginApi.getLoginUserInfo(userRequest);
        output("2_userResponse.json5", userResponse);

        ApiRequest<RefreshTokenRequest> refreshTokenRequest = request("3_refreshTokenRequest.json5", RefreshTokenRequest.class);
        ApiResponse<LoginResult> refreshTokenResponse = loginApi.refreshToken(refreshTokenRequest);
        output("3_refreshTokenResponse.json5", refreshTokenResponse);

        ApiRequest<LogoutRequest> logoutRequest = request("4_logoutRequest.json5", LogoutRequest.class);
        ApiResponse<Void> logoutResponse = loginApi.logout(logoutRequest);
        output("4_logoutResponse.json5", logoutResponse);
    }
```

Where 2 is _ user Request. The contents of json5


```json
{
  data: {
    accessToken: "@var:accessToken"
  }
}
```

We can use `@var:accessToken` to reference the accessToken variable returned from the previous step.

### Integration test support

If we cannot automatically identify and register the AutoTestVariable through the underlying engine in the integration test scenario, we can manually register it in the test case


```java
public void testXXX(){
     ....
    response = myMethod(request);
    setVar("v_myValue", response.myValue);
    // 后续的input文件中就可以通过@var:v_myValue来引用这里定义的变量
    request2 = input("request2.json", Request2.class);
    ...
}
```

In the integration test scenario, we need to access an external independently deployed test database, and we can no longer use a local in-memory database. At this point, we can configure localDb = false to disable the local database


```java
@Test
@EnableSnapshot(localDb=false)
public void integrationTest(){
    ...
}
```

EnableSnapshot has a variety of on-off controls that provide the flexibility to choose which automated test support is enabled


```java
public @interface EnableSnapshot {

    /**
     * 如果启用了快照机制，则缺省会强制使用本地数据库，并且会使用录制的数据来初始化数据库。
     */
    boolean localDb() default true;

    /**
     * 是否自动执行input目录下的sql文件
     */
    boolean sqlInit() default true;

    /**
     * 是否自动将input/tables目录下的数据插入到数据库中
     */
    boolean tableInit() default true;

    /**
     * 是否将收集到的输出数据保存到结果目录下。当saveOutput=true时，checkOutput的设置将会被忽略
     */
    boolean saveOutput() default false;

    /**
     * 是否校验录制的输出数据与数据库中的当前数据相匹配
     */
    boolean checkOutput() default true;
}
```

## Five. Data variation

One of the great advantages of data-driven testing is that it is easy to implement detailed testing of edge scenarios.

Suppose we need to test the system behavior after a user account is in arrears. We know that depending on the size of the user's arrears and the length of the arrears time, the system behavior may change greatly near some thresholds. However, it is a very complex task to construct a complete user consumption and settlement history, and it is difficult for us to construct a large number of user data with subtle differences in the database for edge scenario testing. If we use a data-driven automated testing framework, we can make a copy of the existing test data, and then make fine adjustments directly on it.

The NopAutoTest framework supports this refinement through the concept of data Variants. For example


```java
    @ParameterizedTest
    @EnableVariants
    @EnableSnapshot
    public void testVariants(String variant) {
        input("request.json", ...);
        output("displayName.json5",testInfo.getDisplayName());
    }
```

After adding `@EnableVariants` and `@ParameterizedTest` annotations, when we call the input function, the data it reads is the result of merging the data in the/variants/{ variant}/input directory with the data in the/input directory.


```
/input
   /tables
      my_table.csv
   request.json
/output
   response.json
/variants
   /x
      /input
         /tables
            my_table.csv
         request.json
      /output
         response.json
   /y
      /input
     ....
```

First, the test user will execute ignoring the variants configuration, and the data will be recorded in the input/tables directory. Then, after the variant mechanism is turned on, the test case is executed again according to each variant.

Taking the configuration of testVariants as an example, it will actually be executed three times. The first pass `variant=_default` means that it will be executed with the raw input/output directory data. The second pass executes the data in the variants/X directory, and the third pass executes the data in the variants/y directory.

Because the data between different variants is often very similar, we do not need to copy the original data completely. The NopAutoTest framework adopts the unified design of reversible computing theory, and can use the built-in delta merging mechanism of Nop platform to achieve configuration simplification. For example, in a file request. JSON/variants/X/input


```json
{
    "x:extends":"../../input/request.json"
    "amount": 300
}
```

 `x:extends` Is a standard deviation extension syntax introduced by reversible computing theory, which represents inheritance from the original request. JSON, but modifies the amount attribute to 300.

Similarly, for `/input/tables/my_table.csv` the data in, we can only add the primary key columns and the columns that need to be customized, and then the contents will be automatically merged with the corresponding files in the original directory. For example


```csv
SID, AMOUNT
1001, 300
```

The whole Nop platform is designed and implemented from scratch based on the principle of reversible computation, and its specific content can be found in the reference document at the end of this article.

To some extent, data-driven testing also reflects the so-called reversibility requirement of reversible computing, that is, the information we have expressed through DSL (JSON data and matching templates) can be extracted in reverse, and then converted into other information through reprocessing. For example, when the data structure or interface changes, we can migrate the test case data to the new structure by writing a unified data migration code without re-recording the test case.

## Six. Markdown as DSL Carrier

Reversible computing theory emphasizes the replacement of general imperative program coding by descriptive DSL, so as to reduce the amount of code corresponding to business logic in various fields and levels, and reduce the code through systematic solutions.

In terms of test data expression and verification, besides using JSON/yaml and other forms, Markdown format, which is closer to the document form, can also be considered.

In XLang testing, we specify a standardized markdown structure for expressing test cases.


```markdown
# 测试用例标题
 具体说明文字，可以采用一般的markdown语法，测试用例解析时会自动忽略这些说明
‘’‘测试代码块的语言
测试代码
’‘’

 * 配置名: 配置值
 * 配置名: 配置
```

See the test case [TestXpl](https://gitee.com/canonical-entropy/nop-entropy/tree/master/nop-xlang/src/test/resources/io/nop/xlang/xpls) of TestXpl for specific examples.

## Seven. Brief summary

Nop platform is a new generation of low-code platform built from scratch based on the principle of reversible computing. It ** It adopts the forward design scheme of DSL first, model first and automatic test first. ** is not based on the existing program framework combined with part of the low code transformation, and can overcome the difficulties of the low code scheme disclosed by the industry at present in many aspects.

NopAutoTest is an integral part of the Nop platform. It makes full use of the existing model information within the Nop platform for automatic derivation, and combines with the unique difference quantification structure definition syntax of reversible computing, which can effectively reduce the maintenance cost of automated test cases.

In the technical system of Nop platform, low code is positioned in the development stage, that is, low code will generate code according to the model and integrate closely with the code written by hand. The no-code is located in the running stage, which interacts with the user through the visual interface to realize the customization and adjustment of some logic. NopAutoTest is a testing framework that supports low-code development. It can be integrated into the DevOps process of general development without deploying additional services separately. It uses the existing maventest instructions to execute tests.

On the other hand, although NopAutoTest provides an integration solution for the JUnit testing framework, its core code is virtually independent of any unit testing framework, so it is possible to integrate its functionality into the runtime engine. For example, we can provide a debugging switch on the interface. When it is enabled, it means to record a test case. At this time, it will automatically track all subsequent background calls, record all database data accessed and changes made to the database, and then package it as an offline test case.

For a detailed introduction to the theory of reversible computation, see my previous article [ Reversible Computing: The Next Generation of Software Construction Theory ](https://zhuanlan.zhihu.com/p/64004026)[ Low Code Platform Design from the Perspective of Tensor Product ](https://zhuanlan.zhihu.com/p/531474176)[ What ORM engine is needed for low-code platforms (1) ](https://zhuanlan.zhihu.com/p/543252423)[ What ORM engine is needed for low-code platforms (2) ](https://zhuanlan.zhihu.com/p/545063021).