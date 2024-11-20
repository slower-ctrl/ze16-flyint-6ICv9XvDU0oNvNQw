
在2024年10月8日，Spring AI再次进行了更新，尽管当前版本仍为非稳定版本（1\.0\.0\-M3），但博主将持续关注这些动态，并从流行的智能体视角深入解析其技术底层。目前，Spring AI仍处于小众状态，尚未经过开源社区多年的维护和稳定化过程，这与已经较为成熟的Spring框架形成鲜明对比。即便是Spring AI的稳定版本（1\.0\.0\-SNAPSHOT），在常见的maven仓库中也难以找到，仍需通过Spring的jfrog仓库进行访问。


好的，我们不再绕圈子，直接进入主题。在1\.0\.0\-M3版本中，进行了许多重要的更新，我将逐一详细讲解这些特性。今天的重点是深入解析Advisors的概念，因为它与我们当前工作中所使用的一些技术有很多相似之处，能够帮助大家更容易地理解相关内容。因此，我相信通过这部分的讲解，大家将能更好地掌握Spring AI的核心功能。现在，就让我们开始吧！


# 什么是 Spring AI Advisors？


Spring AI Advisor的核心功能在于拦截并可能修改AI应用程序中聊天请求和响应流的组件。在这个系统中，AroundAdvisor是关键参与者，它允许开发人员在这些交互过程中动态地转换或利用信息。


使用Advisor的主要优势包括：


1. **重复任务的封装**：能够将常见的生成式AI模式打包成可重用的单元，简化开发过程。
2. **数据转换**：增强发送给语言模型（LLM）的数据，并优化返回给客户端的响应格式，以提高交互质量。
3. **可移植性**：创建可跨不同模型和用例工作的可重用转换组件，提升代码的灵活性和适应性。


或许你会觉得，这与我们在Spring中使用AspectJ的方式颇为相似。实际上，当你阅读完今天的文章后，会发现这不过是换了个名称而已，主要功能其实是一致的，都是为了增强应用程序的能力。


## Advisors VS Advised


这里我们简单澄清一下“Advisors”和“Advised”这两个术语。实际上，它们之间并没有直接关系，只是因为在查看源码时，我们常常会遇到这两个词。Advisors是指我们创建的各种增强功能类，它们负责对请求链路进行不同的处理。而“Advised”则是一个形容词，用来描述某个类已不再是普通类，它经过增强后具备了新的特性。尽管它们也对请求类进行了增强，但这种增强主要是通过属性迁移的方式实现的。


下面的图示将帮助进一步理解这一点。


![image](https://img2024.cnblogs.com/blog/1423484/202411/1423484-20241112081852780-1109044291.png)


# Advisors如何运作


如果你以前编写过AspectJ的注解类，那么你应该能够很容易地推测出Advisors是如何运作的。在Advisor系统中，各个Advisor以链式结构运行，序列中的每个Advisor都有机会对传入的请求和传出的响应进行处理。这种链式处理机制确保了每个Advisor可以在请求和响应流中添加自己的逻辑，从而实现更灵活和可定制的功能。


为了帮助理解这一流程，下面我们先来看一下官方提供的流程图。这张图详细展示了各个Advisor如何在请求链中进行交互，以及它们如何协同工作以增强整体功能。


![image](https://img2024.cnblogs.com/blog/1423484/202411/1423484-20241112081858337-1407740098.png)


我来大致讲解一下整个流程：


首先，我们会封装各种请求参数配置，如前面AdvisedRequest的截图所示，这里就不再详细说明。接下来，链中的每个Advisor都会处理请求，可能对其进行修改，并将执行流程转发给链中的下一个Advisor。值得注意的是，某些Advisor也可以选择不调用下一个实体，从而阻止请求继续传递。


最终，Advisor将请求发送到Chat Model。聊天模型的响应将通过Advisor链传递回原请求路径，形成原始上下文和建议上下文的组合。每个Advisor都有机会处理或修改这个响应，确保其符合预期。最后，系统将返回一个AdvisedResponse给客户端。


接下来，我们将深入探讨如何实际使用Advisor。


# 使用 Advisor


## 内嵌的Advisor


作为Spring AI的一部分，系统内置了多个官方Advisor示例，这些示例不仅数量不多，而且功能各异，能够很好地展示Advisor的实际应用场景。我们不妨一起来逐一查看这些内置Advisor的作用和特点，深入了解它们如何在请求处理链中发挥各自的功能。


* MessageChatMemoryAdvisor是我们之前提到过的一个常用类，它在请求处理流程中扮演着重要的角色。这个Advisor的主要功能是将用户提出的问题和模型的回答添加到历史记录中，从而形成一个上下文记忆的增强机制。通过这种方式，系统能够更好地理解用户的需求，提供更加连贯和相关的响应。


	+ 需要注意的是，并非所有的AI模型都支持这种上下文记忆的存储和管理方式。某些模型可能没有实现相应的历史记录功能，因此在使用MessageChatMemoryAdvisor时，确保所使用的模型具备此支持是至关重要的。
* PromptChatMemoryAdvisor的功能在MessageChatMemoryAdvisor的基础上进一步增强，其主要作用在于上下文聊天记录的处理方式。与MessageChatMemoryAdvisor不同，PromptChatMemoryAdvisor并不将上下文记录直接传入messages参数中，而是巧妙地将其封装到systemPrompt提示词中。这一设计使得无论所使用的模型是否支持messages参数，系统都能够有效地增加上下文历史记忆。
* QuestionAnswerAdvisor的主要功能是执行RAG（Retrieval\-Augmented Generation）检索，这一过程涉及对知识库的高效调用。当用户提出问题时，QuestionAnswerAdvisor会首先对知识库进行检索，并将匹配到的相关引用文本添加到用户提问的后面，从而为生成的回答提供更为丰富和准确的上下文。


	+ 此外，该Advisor设定了一个默认提示词，旨在确保回答的质量和相关性。如果在知识库中无法找到匹配的文本，系统将拒绝回答用户的问题。
* SafeGuardAdvisor的核心功能是进行敏感词校验，以确保系统在处理用户输入时的安全性和合规性。当用户提交的信息触发了敏感词机制，SafeGuardAdvisor将立即对该请求进行中途拦截，避免继续调用大型模型进行处理。
* SimpleLoggerAdvisor：这是一个用于日志打印的工具，我们之前已经对其进行了练习和深入了解，因此在这里不再赘述。
* VectorStoreChatMemoryAdvisor：该组件实现了长期记忆功能，能够将每次用户提出的问题及模型的回答存储到向量数据库中。在用户每次提问时，系统会进行一次检索，将检索到的信息累加到系统提示词的后面，以便为大模型提供更准确的上下文提示。然而，这里需要注意的是，如果没有妥善维护 `chat_memory_conversation_id`，可能会导致无限制的写入和检索，从而引发潜在的灾难性bug。因此，确保这一标识的管理和更新至关重要，以避免系统的不稳定性和数据混乱。


在这里，我们主要讨论 `chat_memory_conversation_id` 参数，它在所有 Advisor 中都是一个关键要素。我们必须为每位用户妥善维护这个参数，避免每次默认生成新 ID，以防在向量数据库中产生大量垃圾数据。此外，维护好该参数后，可以在后台利用它清理向量数据库中的旧数据。需要注意的是，这里使用的是存储在 metadata 中的 `chat_memory_conversation_id`，而不是简单的 ID，因此在删除和清理时，需先进行查询。


## 自定义Advisor


其实，我们之前已经实现过一个简单的日志记录 Advisor。今天，我们将基于 Re\-Reading （Re2）技术，打造一个更高级的 Advisor。实际上，理解这一过程并不复杂，核心在于提前对请求的问题进行包装。


对于有兴趣的同学，我推荐你们查看 Re2 技术的相关实现及效果，详细内容可以参考这篇论文：[Re2技术实现](https://github.com)。


此外，关于提示词的优化，如果你对这方面特别感兴趣，我建议你浏览一下免费开源的博客文档，这里有很多有价值的资源可以参考：[提示词指南](https://github.com):[FlowerCloud机场](https://hushicha.org)。


接下来，我们不再啰嗦，直接来看一下官方的示例代码：



```
public class ReReadingAdvisor implements CallAroundAdvisor, StreamAroundAdvisor {
  private static final String DEFAULT_USER_TEXT_ADVISE = """
      {re2_input_query}
      Read the question again: {re2_input_query}
      """;

  @Override
  public String getName() {
      return this.getClass().getSimpleName();
  }

  @Override
  public int getOrder() {
      return 0;
  }

  private AdvisedRequest before(AdvisedRequest advisedRequest) {

      String inputQuery = advisedRequest.userText(); //original user query

      Map params = new HashMap<>(advisedRequest.userParams());        
      params.put("re2_input_query", inputQuery);

      return AdvisedRequest.from(advisedRequest)
              .withUserText(DEFAULT_USER_TEXT_ADVISE)
              .withUserParams(params)
              .build();
  }

  @Override
  public AdvisedResponse aroundCall(AdvisedRequest advisedRequest, CallAroundAdvisorChain chain) {
      return chain.nextAroundCall(before(advisedRequest));
  }

  @Override
  public Flux aroundStream(AdvisedRequest advisedRequest, StreamAroundAdvisorChain chain) {
      return chain.nextAroundStream(before(advisedRequest));
  }
}

```

可以看到，在这里的实现中，实际上并没有过多的代码编写，仅仅是声明了一个全局的文本模板，并在请求之前对其进行了简单的封装。这种设计思路的核心在于通过模板的预处理，提升请求的有效性和上下文的相关性。根据 Re2 的官方说明，这种方法不仅能够简化代码结构，还能显著提升模型的回答效果。


## 共享参数Advisor


在之前的讲解中，例如我们讨论的 `messageChatMemoryAdvisor`，在 Bean 声明时，实际上是专门为参数配置编写了默认设置。尽管如此，我们仍然可以通过参数传入的方式进行动态配置，这种灵活性让我们能够根据实际需求调整参数。你可以传入任何所需的参数，并在重写的方法中进行读取和应用，从而使 Advisor 更加灵活和适应不同场景的需求。


在这里，我们将以官方的 `messageChatMemoryAdvisor` 为例，展示以前的写法，以便更好地理解这一配置过程。



```
ChatDataPO functionGenerationByText(@RequestParam("userInput")  String userInput) { 
    OpenAiChatOptions openAiChatOptions = OpenAiChatOptions.builder()
            .withModel("hunyuan-pro").withTemperature(0.5).build();
    String content = this.myChatClientWithSystem
            .prompt()
            .system("请你作为一个小雨的AI小助手，请将工具返回的数据格式化后以友好的方式回复用户的问题。制定的旅游攻略要有航班、酒店、火车信息")
            .user(userInput)
            .options(openAiChatOptions)
            .advisors(messageChatMemoryAdvisor,myLoggerAdvisor,promptChatKnowledageAdvisor)
//配置类如下：            
@Bean
MessageChatMemoryAdvisor messageChatMemoryAdvisor() {
    InMemoryChatMemory chatMemory = new InMemoryChatMemory();
    return new MessageChatMemoryAdvisor(chatMemory,"123",10);
}            

```

传递参数的效果可以通过以下方式进行改写，具体实现中省略了一些冗余的重复代码，以便更加清晰地展示主要逻辑：



```
.advisors(messageChatMemoryAdvisor,myLoggerAdvisor,promptChatKnowledageAdvisor)
.advisors(advisor -> advisor.param("chat_memory_conversation_id", "678")
        .param("chat_memory_response_size", 100))

```

这样，我们就能够实时读取参数，并在调用之前进行恰当的配置，下面是具体的源码示例：


![image](https://img2024.cnblogs.com/blog/1423484/202411/1423484-20241112081915363-145303261.png)


官方已经为我们封装了常用的读取模板，提供了一系列高效且易于使用的功能接口。我们只需直接调用这些预定义的模板，便可以快速实现所需的操作。


### 更新参数


除了在开始调用之前设置一些共享参数外，我们还可以在运行期间动态调整这些参数，以便更好地适应实时变化的需求和环境：



```
@Override
public AdvisedResponse aroundCall(AdvisedRequest advisedRequest, CallAroundAdvisorChain chain) {

    this.advisedRequest = advisedRequest.updateContext(context -> {
        context.put("aroundCallBefore" + getName(), "AROUND_CALL_BEFORE " + getName());  // Add multiple key-value pairs
        context.put("lastBefore", getName());  // Add a single key-value pair
        return context;
    });

    // Method implementation continues...
}

```

今天的Advisors介绍就到这里，希望能为你带来一些新的启发和思考。


# 总结


Spring AI Advisors 提供了一种强大而灵活的方法，旨在显著增强你的 AI 应用程序的功能和性能。通过充分利用这一 API，你能够创建出更复杂、可重用且易于维护的 AI 组件，从而提升开发效率和系统的可扩展性。


无论你是在实施自定义逻辑以满足特定业务需求，管理对话历史记录以优化用户体验，还是改进模型推理以获得更准确的结果，Advisors 都能为你提供简洁且高效的解决方案。这种灵活性使得开发者能够快速响应变化，同时保持代码的整洁和可读性，进而为用户提供更加流畅和智能的体验。




---


我是努力的小雨，一名 Java 服务端码农，潜心研究着 AI 技术的奥秘。我热爱技术交流与分享，对开源社区充满热情。同时也是一位腾讯云创作之星、阿里云专家博主、华为云云享专家、掘金优秀作者。


💡 我将不吝分享我在技术道路上的个人探索与经验，希望能为你的学习与成长带来一些启发与帮助。


🌟 欢迎关注努力的小雨！🌟


