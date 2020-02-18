### spring配合Junit进行单元测试

​          在测试类上添加==@RunWith==注解指定使用springJunit的测试运行器,==@ContextConfiguration==注解指定测试用的spring配置文件的位置，之后我们就可以注入我们需要测试的bean进行测试,Junit在运行测试之前会先解析spring的配置文件,初始化spring中配置的bean

##### 基于spring的单元测试步骤

* 首先加载spring的jar包 

  ​        spring-test-4.0.4.RELEASE（注意版本）commons-logging-1.2.jar

* 在applicationContext.xml中，扫描service实现包

​           <context:component-scan base-package="service.impl"></context:component-scan>

* 在UserServiceImpl实现类上使用springmvc 注解@Service("userService")

* 编写spring单元测试(如下)，点击运行

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"classpath:spring/application-          context.xml","classpath:spring/datasource-beans.xml", "classpath:spring/properties.xml"})
public class sendContentTest {
	@Autowired
	public HotEventJobScheduler hotEventJobScheduler;

	@Test
	public void sendContent(){
          hotEventJobScheduler.processCurrentHotGrade();
	}
}
```

##### 基本注解

* @Test：将一个普通的方法修饰成为一个测试方法，可以接受异

  ​            @Test(expected=XX.class)  接受异常
  ​            @Test(timeout=毫秒)   定时结束

* @BeforClass:它会在所有的方法运行前被执行，只执行一次，static修饰,用来加载配置文件

* @AfterClass:它会在所有的方法运行结束后被执行，static修饰，用来释放资源

* @Before：会在每一个测试方法被运行前执行一次

* @After:会在每一个测试方法运行后被执行

* @Ignore：所修饰的方法会被测试运行器忽略

* @RunWith：可以更改测试运行器 只要你的测试器继承org.junit.runner.Runner

* @ContextConfiguration(locations={"classpath:applicationContext.xml"})加载配置文件，locations参数是一个数组，可以加载多个，配置文件

### Mock以及Mockito的使用

#### Mock

​     所谓的mock就是创建一个类的虚假的对象，在测试环境中，用来替换掉真实的对象，以达到两大目的：

1. 验证这个对象的某些方法的调用情况，调用了多少次，参数是什么等等
2. 指定这个对象的某些方法的行为，返回特定的值，或者是执行特定的动作

使用mock，就要用到mock框架，[Mockito](http://mockito.org/) 这个框架，是Java界使用最广泛的一个mock框架

#### MockMvc

​        是SpringMVC提供的Controller测试类，每次进行单元测试时，都是预先执行@Before中的setup方法，初始healthArticleController单元测试环境

​        一定要把待测试的Controller实例进行==MockMvcBuilders.standaloneSetup(xxxxController).build();==    否则会抛出无法找到@RequestMapping路径的异常：No mapping found for HTTP request with URI [/cms/app/getArticleList] in DispatcherServlet

```java
import com.susq.mbts.controller.UserController;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;
import org.springframework.test.web.servlet.ResultActions;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"classpath:applicationContext.xml"})
public class ConTest {
    @Autowired
    private UserController userController;

    private MockMvc mockMvc;

    @Before
    public void setup(){
        mockMvc = MockMvcBuilders.standaloneSetup(userController).build();
    }

    @Test
    public void Ctest() throws Exception {
        ResultActions resultActions = this.mockMvc.perform(MockMvcRequestBuilders.post("/show_user3").param("id", "1"));
        MvcResult mvcResult = resultActions.andReturn();
        String result = mvcResult.getResponse().getContentAsString();
        System.out.println("=====客户端获得反馈数据:" + result);
        // 也可以从response里面取状态码，header,cookies...
//        System.out.println(mvcResult.getResponse().getStatus());
    }
}
```

