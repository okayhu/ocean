> 来源：[Clean Architecture with Spring Boot](https://www.baeldung.com/spring-boot-clean-architecture)
> 作者：[Gilvan Ornelas](https://www.baeldung.com/author/gilvanornelas/)

## 概述

当我们在开发长期的系统时，我们应该期待一个易变的环境。

一般来说，我们的功能需求、框架、I/O 设备，甚至我们的代码设计都可能因为各种原因而改变。考虑到这一点，架构整洁之道（Clean Architecture）是一个高可维护代码的准则，考虑到我们周围所有的不确定性。

在这篇文章中，我们将按照 [Robert C. Martin 的架构整洁之道](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) 创建一个用户注册 API 的例子。我们将使用他的原始层-实体、用例、接口适配器和框架/驱动。

## 整洁架构概述

整洁架构汇编了许多代码设计和原则，如 [SOLID](https://www.baeldung.com/solid-principles)，[稳定的抽象](https://wiki.c2.com/?StableAbstractionsPrinciple)，以及其他。但是，**核心思想是根据业务价值对系统进行层次划分**。因此，最高级别具有业务规则，每个较低级别都更接近 I/O 设备。

此外，我们还可以将级别转化为层。在这种情况下，情况恰恰相反。内层等同于最高层，以此类推。

![](https://www.baeldung.com/wp-content/uploads/2021/01/user-clean-architecture-layers-1.png)

考虑到这一点，**我们可以根据我们的业务需要拥有尽可能多的级别。但是，要始终考虑到依赖性规则：一个较高的级别绝不能取决于一个较低的级别。**

## 规则

让我们开始为我们的用户注册 API 定义系统规则。首先，业务规则：

- 用户的密码必须有五个以上的字符。

其次，我们有应用规则。它们可以以不同的形式出现，如用例或故事。我们将使用一个讲故事的短语。

- 系统收到用户名和密码，验证该用户是否存在，并将新用户和创建时间一起保存下来。

**请注意，这里没有提到任何数据库、用户界面或类似的东西。因为我们的业务并不关心这些细节**，我们的代码也不应该关心。

## 实体层

正如简洁的架构所建议的，让我们从我们的业务规则开始。

```java
interface User {
    boolean passwordIsValid();

    String getName();

    String getPassword();
}
```

此外，还有一个 UserFactory：

```java
interface UserFactory {
    User create(String name, String password);
}
```

我们创建一个用户工厂方法是出于两个原因。为了适应稳定的抽象原则和隔离用户的创建。

接下来，让我们来实现这两点：

```java
class CommonUser implements User {

    String name;
    String password;

    @Override
    public boolean passwordIsValid() {
        return password != null && password.length() > 5;
    }

    // Constructor and getters
}

class CommonUserFactory implements UserFactory {
    @Override
    public User create(String name, String password) {
        return new CommonUser(name, password);
    }
}
```

**如果我们有一个复杂的业务，那么我们应该尽可能清晰地构建我们的领域代码**。所以，这一层是应用[设计模式](https://www.baeldung.com/design-patterns-series)的好地方。特别是，**应该考虑到[领域驱动的设计](https://www.baeldung.com/java-modules-ddd-bounded-contexts)**。

### 单元测试
现在，让我们测试一下我们的 CommonUser：

```java
@Test
void given123Password_whenPasswordIsNotValid_thenIsFalse() {
    User user = new CommonUser("Baeldung", "123");

    assertThat(user.passwordIsValid()).isFalse();
}
```

我们可以看到，单元测试是非常清晰的。毕竟，没有 mocks 是这一层的一个好信号。
一般来说，如果我们在这里开始考虑 mocks，也许我们就把我们的实体和我们的用例混在一起了。

## 用例层

用例是与我们系统的自动化有关的规则。在架构整洁之道中，我们称它们为交互器（Interactors）。

### 用户注册交互器(UserRegisterInteractor)

首先，我们将建立我们的 UserRegisterInteractor，这样我们就可以知道我们要做的事情。然后，我们将创建并讨论所有使用的部分：

```java
class UserRegisterInteractor implements UserInputBoundary {

    final UserRegisterDsGateway userDsGateway;
    final UserPresenter userPresenter;
    final UserFactory userFactory;

    // Constructor

    @Override
    public UserResponseModel create(UserRequestModel requestModel) {
        if (userDsGateway.existsByName(requestModel.getName())) {
            return userPresenter.prepareFailView("User already exists.");
        }
        User user = userFactory.create(requestModel.getName(), requestModel.getPassword());
        if (!user.passwordIsValid()) {
            return userPresenter.prepareFailView("User password must have more than 5 characters.");
        }
        LocalDateTime now = LocalDateTime.now();
        UserDsRequestModel userDsModel = new UserDsRequestModel(user.getName(), user.getPassword(), now);

        userDsGateway.save(userDsModel);

        UserResponseModel accountResponseModel = new UserResponseModel(user.getName(), now.toString());
        return userPresenter.prepareSuccessView(accountResponseModel);
    }
}
```

正如我们所见，我们正在执行所有用例步骤。另外，这一层还负责控制实体的舞蹈。尽管如此，我们仍然没有对用户界面或数据库的工作方式做出任何假设。但是，我们正在使用 UserDsGateway 和 UserPresenter。那么，我们怎么能不知道它们呢？因为，与 UserInputBoundary 一起，这些是我们的输入和输出边界。

### 输入和输出边界

边界是定义组件如何交互的契约。**输入边界将我们的用例暴露给外层：**

```java
interface UserInputBoundary {
    UserResponseModel create(UserRequestModel requestModel);
}
```

接下来，我们有我们的输出边界，用于利用外层。首先，让我们定义数据源网关。

```java
interface UserRegisterDsGateway {
    boolean existsByName(String name);

    void save(UserDsRequestModel requestModel);
}
```

其次，视图呈现器：

```java
interface UserPresenter {
    UserResponseModel prepareSuccessView(UserResponseModel user);

    UserResponseModel prepareFailView(String error);
}
```

注意：**我们正在使用[依赖性倒置原则](https://www.baeldung.com/java-dependency-inversion-principle)，使我们的业务不受数据库和UI等细节的影响**。

### 解耦模式
在继续之前，**请注意边界是如何定义系统自然划分的契约**。但我们还必须决定我们的应用程序将如何交付：

- 单片 - 可能使用某些包结构进行组织
- 通过使用模块
- 通过使用服务/微服务

考虑到这一点，**我们可以通过任何解耦模式达到干净的架构目标**。因此，**我们应该准备根据我们当前和未来的业务需求，在这些策略之间进行改变**。在选择了我们的解耦模式后，代码的划分应该根据我们的边界来进行。

### 请求和响应模型

到目前为止，我们已经使用接口创建了跨层的操作。接下来，让我们看看如何跨这些边界传输数据。

注意我们所有的边界是如何只处理*String*或*Model*对象的：

```java
class UserRequestModel {

    String login;
    String password;

    // Getters, setters, and constructors
}
```

基本上，**只有简单的数据结构才能跨越边界**。此外，所有模型都只有字段和访问器。另外，数据对象属于内侧。所以，我们可以保持依赖规则。

但是为什么我们有这么多相似的对象呢？当我们得到重复的代码时，它可能有两种类型：

- 错误或意外重复。代码相似性是偶然的，因为每个对象都有不同的更改原因。如果我们试图删除它，我们将面临违反[单一责任原则](https://www.baeldung.com/java-single-responsibility-principle)的风险。
- 真正的重复。代码出于同样的原因而改变。因此，我们应该删除它

由于每个模型都有不同的职责，所以我们得到了这些对象。

### 测试用户注册交互器

现在，让我们创建我们的单元测试：

```java
@Test
void givenBaeldungUserAnd12345Password_whenCreate_thenSaveItAndPrepareSuccessView() {
    given(userDsGateway.existsByIdentifier("identifier"))
        .willReturn(true);

    interactor.create(new UserRequestModel("baeldung", "123"));

    then(userDsGateway).should()
        .save(new UserDsRequestModel("baeldung", "12345", now()));
    then(userPresenter).should()
        .prepareSuccessView(new UserResponseModel("baeldung", now()));
}
```

正如我们所见，大多数用例测试都是关于控制实体和边界请求。而且，我们的界面允许我们轻松地模拟细节。

##  接口适配器

至此，我们完成了所有业务。现在，让我们开始插入我们的详细信息。

**我们的业务应该只处理最方便的数据格式**，我们的外部代理（如 DB 或 UI）也应如此。**但是，这种格式通常是不同的**。为此，**接口适配器层负责数据的转换**。

### 使用 JPA 的 UserRegisterDsGateway

首先，让我们使用[JPA](https://www.baeldung.com/the-persistence-layer-with-spring-and-jpa)来映射我们的用户表：

```java
@Entity
@Table(name = "user")
class UserDataMapper {

    @Id
    String name;

    String password;

    LocalDateTime creationTime;

    //Getters, setters, and constructors
}
```

正如我们所见，Mapper 的目标是将我们的对象映射到数据库格式。

接下来，使用我们的[实体的](https://www.baeldung.com/jpa-entities)JpaRepository：

```java
@Repository
interface JpaUserRepository extends JpaRepository<UserDataMapper, String> {
}
```

鉴于我们将使用 spring-boot，那么这就是保存用户所需的全部内容。

现在，是时候实现我们的 UserRegisterDsGateway 了：

```java
class JpaUser implements UserRegisterDsGateway {

    final JpaUserRepository repository;

    // Constructor

    @Override
    public boolean existsByName(String name) {
        return repository.existsById(name);
    }

    @Override
    public void save(UserDsRequestModel requestModel) {
        UserDataMapper accountDataMapper = new UserDataMapper(requestModel.getName(), requestModel.getPassword(), requestModel.getCreationTime());
        repository.save(accountDataMapper);
    }
}
```

在大多数情况下，代码不言自明。除了我们的方法，请注意 UserRegisterDsGateway 的名称。如果我们选择 UserDsGateway 代替，那么其他用户用例可能会违反[接口隔离原则](https://www.baeldung.com/java-interface-segregation)。

### 用户注册 API

现在，让我们创建我们的 HTTP 适配器：

```java
@RestController
class UserRegisterController {

    final UserInputBoundary userInput;

    // Constructor

    @PostMapping("/user")
    UserResponseModel create(@RequestBody UserRequestModel requestModel) {
        return userInput.create(requestModel);
    }
}
```

正如我们所看到的，**这里唯一的目标是接收请求并将响应发送**给客户端。

### 准备响应

在回复之前，我们应该格式化我们的回复：

```java
class UserResponseFormatter implements UserPresenter {

    @Override
    public UserResponseModel prepareSuccessView(UserResponseModel response) {
        LocalDateTime responseTime = LocalDateTime.parse(response.getCreationTime());
        response.setCreationTime(responseTime.format(DateTimeFormatter.ofPattern("hh:mm:ss")));
        return response;
    }

    @Override
    public UserResponseModel prepareFailView(String error) {
        throw new ResponseStatusException(HttpStatus.CONFLICT, error);
    }
}
```

我们的 UserRegisterInteractor 专注于创建用户。尽管如此，表示规则只涉及适配器内部。此外，**凡是难以测试的东西，我们都应该把它分为可测试的对象和[不起眼的对象](https://martinfowler.com/bliki/HumbleObject.html)。** 因此，UserResponseFormatter 很容易让我们验证我们的表示规则：

```java
@Test
void givenDateAnd3HourTime_whenPrepareSuccessView_thenReturnOnly3HourTime() {
    UserResponseModel modelResponse = new UserResponseModel("baeldung", "2020-12-20T03:00:00.000");
    UserResponseModel formattedResponse = userResponseFormatter.prepareSuccessView(modelResponse);

    assertThat(formattedResponse.getCreationTime()).isEqualTo("03:00:00");
}
```

正如我们所见，我们在将其发送到视图之前测试了所有逻辑。因此，**只有不起眼的对象在不太可测试的部分**。

## 驱动程序和框架

事实上，我们通常不在这里编码。那是因为这一层**代表了与外部代理的最低级别的连接**。例如，用于连接数据库或 web 框架的 H2 驱动程序。在这种情况下，**我们将使用[spring-boot](https://www.baeldung.com/spring-boot)作为[web](https://www.baeldung.com/bootstraping-a-web-application-with-spring-and-java-based-configuration)和[依赖注入](https://www.baeldung.com/spring-dependency-injection)框架**。所以，我们需要它的启动点：

```java
@SpringBootApplication
public class CleanArchitectureApplication {
    public static void main(String[] args) {
      SpringApplication.run(CleanArchitectureApplication.class);
    }
}
```

到目前为止，**我们的业务中还没有使用任何** **[Spring 注解](https://www.baeldung.com/spring-bean-annotations)**。除了 spring-specifics 适配器之外，作为我们的 UserRegisterController。**这是因为我们应该将 spring-boot 视为任何其他细节**。

## 可怕的主类

终于，最后一块了！

到目前为止，我们遵循了[稳定抽象原则](https://wiki.c2.com/?StableAbstractionsPrinciple)。此外，我们通过[控制反转](https://www.baeldung.com/inversion-control-and-dependency-injection-in-spring)保护我们的内层免受外部代理的影响。最后，我们将所有对象的创建与使用分开。此时，由我们来**创建剩余的依赖项并将它们注入到我们的项目中**：

```java
@Bean
BeanFactoryPostProcessor beanFactoryPostProcessor(ApplicationContext beanRegistry) {
    return beanFactory -> {
        genericApplicationContext(
          (BeanDefinitionRegistry) ((AnnotationConfigServletWebServerApplicationContext) beanRegistry)
            .getBeanFactory());
    };
}

void genericApplicationContext(BeanDefinitionRegistry beanRegistry) {
    ClassPathBeanDefinitionScanner beanDefinitionScanner = new ClassPathBeanDefinitionScanner(beanRegistry);
    beanDefinitionScanner.addIncludeFilter(removeModelAndEntitiesFilter());
    beanDefinitionScanner.scan("com.baeldung.pattern.cleanarchitecture");
}

static TypeFilter removeModelAndEntitiesFilter() {
    return (MetadataReader mr, MetadataReaderFactory mrf) -> !mr.getClassMetadata()
      .getClassName()
      .endsWith("Model");
}
```

在我们的例子中，我们使用 spring-boot [依赖注入](https://www.baeldung.com/spring-dependency-injection)来创建我们所有的实例。由于我们没有使用[@Component](https://www.baeldung.com/spring-bean-annotations#component)，我们正在扫描我们的根包并仅忽略 Model objects 。

尽管这种策略看起来可能更复杂，但它使我们的业务与 DI 框架分离。另一方面，**主类控制了我们所有的系统**。这就是为什么干净的架构将它视为一个包含所有其他层的特殊层：

![](https://www.baeldung.com/wp-content/uploads/2021/01/user-clean-architecture-layers.png)

## 结论

在本文中，我们了解了 Bob 大叔的 **整洁架构是如何建立在许多设计模式和原则之上的**。此外，我们创建了一个使用 Spring Boot 应用它的用例。

尽管如此，我们还是把一些原则放在一边。但是，他们都朝着同一个方向前进。我们可以通过引用它的创建者来总结它：“一个好的架构师**必须最大化未做出的决定的数量**。”我们通过**使用边界保护我们的业务代码免受细节**影响来做到这一点。

像往常一样，完整的代码可以 [在 GitHub 上找到](https://github.com/eugenp/tutorials/tree/master/patterns/clean-architecture)。
