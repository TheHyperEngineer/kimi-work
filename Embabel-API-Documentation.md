# Embabel Agent Framework: Complete API Documentation

**Version:** 0.3.5-SNAPSHOT  
**Last Updated:** March 2026  
**Target Audience:** Java/Kotlin Developers building AI Agents

---

## Table of Contents

1. [Package Overview](#1-package-overview)
2. [Core Annotations (com.embabel.agent.api.annotation)](#2-core-annotations)
3. [Common API (com.embabel.agent.api.common)](#3-common-api)
4. [Core Types (com.embabel.agent.core)](#4-core-types)
5. [Tool API (com.embabel.agent.api.tool)](#5-tool-api)
6. [Agentic Tools (com.embabel.agent.api.tool.agentic)](#6-agentic-tools)
7. [Invocation API (com.embabel.agent.api.invocation)](#7-invocation-api)
8. [Event System (com.embabel.agent.api.event)](#8-event-system)
9. [Chat API (com.embabel.chat)](#9-chat-api)
10. [DSL (com.embabel.agent.api.dsl)](#10-dsl)
11. [Workflow API (com.embabel.agent.api.common.workflow)](#11-workflow-api)
12. [Thinking API (com.embabel.agent.api.common.thinking)](#12-thinking-api)
13. [Validation API (com.embabel.agent.api.validation)](#13-validation-api)
14. [Model Constants (com.embabel.agent.api.models)](#14-model-constants)
15. [Complete Class Reference](#15-complete-class-reference)

---

## 1. Package Overview

The Embabel Agent Framework is organized into the following main packages:

| Package | Description |
|---------|-------------|
| `com.embabel.agent.api.annotation` | Core annotations for defining agents, actions, and goals |
| `com.embabel.agent.api.common` | Common interfaces and types for agent operations |
| `com.embabel.agent.core` | Core agent platform, process, and blackboard types |
| `com.embabel.agent.api.tool` | Framework-agnostic tool interface |
| `com.embabel.agent.api.tool.agentic` | Agentic tool implementations |
| `com.embabel.agent.api.invocation` | Agent invocation APIs |
| `com.embabel.agent.api.event` | Event system for agent processes |
| `com.embabel.chat` | Chatbot and conversation management |
| `com.embabel.agent.api.dsl` | Kotlin DSL for agent creation |
| `com.embabel.agent.api.models` | LLM model constants |

---

## 2. Core Annotations

### Package: `com.embabel.agent.api.annotation`

#### @Agent

Indicates that a class is an agent. This is a Spring stereotype annotation.

```kotlin
@Target(allowedTargets = [AnnotationTarget.CLASS])
annotation class Agent(
    val name: String = "",
    val provider: String = "",
    val description: String,
    val version: String = DEFAULT_VERSION,
    val planner: PlannerType = PlannerType.GOAP,
    val scan: Boolean = true,
    val beanName: String = "",
    val opaque: Boolean = false,
    val actionRetryPolicy: ActionRetryPolicy = ActionRetryPolicy.DEFAULT,
    val actionRetryPolicyExpression: String = ""
)
```

**Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | String | `""` | Agent name (defaults to class name) |
| `provider` | String | `""` | Provider identifier |
| `description` | String | **Required** | Human-readable description |
| `version` | String | `"1.0.0"` | Agent version |
| `planner` | PlannerType | `GOAP` | Planning strategy (GOAP, UTILITY, SUPERVISOR) |
| `scan` | Boolean | `true` | Enable classpath scanning |
| `beanName` | String | `""` | Spring bean name |
| `opaque` | Boolean | `false` | Hide internal actions from external view |

**Example:**
```java
@Agent(description = "Agent that writes and reviews stories")
public class WriteAndReviewAgent {
    // Actions...
}
```

---

#### @Action

Indicates a method implementing an action within an agent.

```kotlin
@Target(allowedTargets = [AnnotationTarget.FUNCTION])
annotation class Action(
    val description: String = "",
    val pre: Array<String> = [],
    val post: Array<String> = [],
    val canRerun: Boolean = false,
    val readOnly: Boolean = false,
    val clearBlackboard: Boolean = false,
    val outputBinding: IoBinding = IoBinding.DEFAULT_BINDING,
    val cost: Double = 0.0,
    val value: Double = 0.0,
    val costMethod: String = "",
    val valueMethod: String = "",
    val trigger: KClass<*> = Unit::class,
    val actionRetryPolicy: ActionRetryPolicy = ActionRetryPolicy.DEFAULT,
    val actionRetryPolicyExpression: String = ""
)
```

**Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `description` | String | `""` | Human-readable action description |
| `pre` | Array<String> | `[]` | SpEL preconditions |
| `post` | Array<String> | `[]` | SpEL postconditions |
| `canRerun` | Boolean | `false` | Allow action to be rerun |
| `readOnly` | Boolean | `false` | Action has no side effects |
| `clearBlackboard` | Boolean | `false` | Clear blackboard after execution |
| `cost` | Double | `0.0` | Relative cost (0-1) |
| `value` | Double | `0.0` | Relative value (0-1) |
| `trigger` | KClass<*> | `Unit::class` | Trigger type for reactive actions |

**Example:**
```java
@Action(
    description = "Extract person information",
    pre = {"spel:assessment.urgency > 0.5"},
    cost = 0.1,
    value = 0.8
)
public Person extractPerson(UserInput input, OperationContext context) {
    // Implementation
}
```

---

#### @AchievesGoal

Indicates that an action achieves the agent's goal.

```kotlin
@Target(allowedTargets = [AnnotationTarget.FUNCTION])
annotation class AchievesGoal(
    val description: String = ""
)
```

**Example:**
```java
@AchievesGoal(description = "Review a story and provide feedback")
@Action
public ReviewedStory reviewStory(Story story, OperationContext context) {
    // Final action
}
```

---

#### @EmbabelComponent

Indicates a class that exposes actions, goals, and conditions but is not an agent itself.

```kotlin
@Target(allowedTargets = [AnnotationTarget.CLASS])
annotation class EmbabelComponent(
    val scan: Boolean = true
)
```

**Example:**
```java
@EmbabelComponent
public class SharedActions {
    @Action
    public CommonResult commonAction(Input input) {
        // Shared action
    }
}
```

---

#### @State

Marks a class representing a state within a workflow.

```kotlin
@Target(allowedTargets = [AnnotationTarget.CLASS])
annotation class State
```

**Example:**
```java
@State
record ProcessingState(String data, int iteration) implements LoopOutcome {
    @Action(clearBlackboard = true)
    LoopOutcome process() {
        // State transition logic
    }
}
```

---

#### @LlmTool

Marks a method as a tool that can be invoked by an LLM.

```kotlin
@Target(allowedTargets = [AnnotationTarget.FUNCTION])
annotation class LlmTool(
    val description: String = "",
    val name: String = "",
    val returnDirect: Boolean = false,
    val category: String = ""
)
```

**Example:**
```java
public class MathTools {
    @LlmTool(description = "Add two numbers together")
    public double add(
        @LlmTool.Param(description = "First number") double a,
        @LlmTool.Param(description = "Second number") double b
    ) {
        return a + b;
    }
}
```

---

#### @Provided

Marks an action method parameter as being provided by the platform context.

```kotlin
@Target(allowedTargets = [AnnotationTarget.VALUE_PARAMETER])
annotation class Provided
```

**Example:**
```java
@Action
public Output process(Input input, @Provided MyService service) {
    // Service is injected from Spring context
}
```

---

#### @Cost

Annotates a method that computes the dynamic cost or value of an action.

```kotlin
@Target(allowedTargets = [AnnotationTarget.FUNCTION])
annotation class Cost(
    val name: String = ""
)
```

**Example:**
```java
@Cost(name = "processingCost")
public double computeProcessingCost(@Nullable LargeDataSet data) {
    return (data != null && data.size() > 1000) ? 0.9 : 0.1;
}

@Action(costMethod = "processingCost")
public ProcessedData process(RawData input) {
    // Action implementation
}
```

---

#### @Condition

Annotates a method that evaluates a condition.

```kotlin
@Target(allowedTargets = [AnnotationTarget.FUNCTION])
annotation class Condition(
    val name: String = "",
    val cost: Double = 0.0
)
```

---

#### @Export

Specifies how a goal should be exposed (local or remote/MCP).

```kotlin
annotation class Export(
    val name: String? = null,
    val remote: Boolean = false,
    val local: Boolean = true,
    val startingInputTypes: Set<Class<*>> = emptySet()
)
```

---

## 3. Common API

### Package: `com.embabel.agent.api.common`

#### OperationContext

Context for any operation. Exposes blackboard and process context.

```kotlin
interface OperationContext : Blackboard, ToolGroupConsumer {
    fun ai(): Ai
    fun processContext(): ProcessContext
}
```

**Key Methods:**
| Method | Returns | Description |
|--------|---------|-------------|
| `ai()` | `Ai` | Gateway to AI functionality |
| `processContext()` | `ProcessContext` | Access to process state |

---

#### ActionContext

Context for actions with access to blackboard and event listening.

```kotlin
interface ActionContext : Blackboard, AgenticEventListener {
    fun processContext(): ProcessContext
    fun action(): Action
    fun sendMessage(message: Message)
}
```

---

#### Ai

Gateway to AI functionality including LLM and embedding models.

```kotlin
interface Ai {
    fun withDefaultLlm(): PromptRunner
    fun withLlm(llm: LlmOptions): PromptRunner
    fun withLlmByRole(role: String): PromptRunner
    fun withLlm(model: String): PromptRunner
}
```

**Methods:**
| Method | Returns | Description |
|--------|---------|-------------|
| `withDefaultLlm()` | `PromptRunner` | Use platform default LLM |
| `withLlm(llm: LlmOptions)` | `PromptRunner` | Use specific LLM options |
| `withLlmByRole(role: String)` | `PromptRunner` | Use LLM by configured role |
| `withLlm(model: String)` | `PromptRunner` | Use specific model name |

---

#### PromptRunner

User-facing interface for executing prompts. The primary way to interact with LLMs.

```kotlin
interface PromptRunner : LlmUse, PromptRunnerOperations, ToolChaining<PromptRunner>
```

**Core Methods:**
| Method | Returns | Description |
|--------|---------|-------------|
| `createObject(prompt: String, clazz: Class<T>): T` | `T` | Create typed object from prompt |
| `createObjectIfPossible(prompt: String, clazz: Class<T>): T?` | `T?` | Try to create object, return null on failure |
| `generateText(prompt: String): String` | `String` | Generate simple text response |
| `withToolGroup(role: String): PromptRunner` | `PromptRunner` | Add tool group |
| `withToolObject(obj: Any): PromptRunner` | `PromptRunner` | Add object with @LlmTool methods |
| `withImage(image: AgentImage): PromptRunner` | `PromptRunner` | Add image for vision |

---

#### AgentImage

Represents an image for Agent API operations.

```kotlin
data class AgentImage(
    val mimeType: String,
    val data: ByteArray
)
```

**Factory Methods:**
```kotlin
// From file
val image = AgentImage.fromFile(File("image.png"))

// From path
val image = AgentImage.fromPath(Path.of("image.png"))

// From bytes
val image = AgentImage("image/png", byteArray)
```

---

#### MultimodalContent

Represents multimodal content (text + images) for agent operations.

```kotlin
data class MultimodalContent(
    val text: String,
    val images: List<AgentImage> = emptyList()
)
```

**Builder:**
```kotlin
val content = multimodal("Describe this image")
    .withImage(imageFile)
    .build()
```

---

#### PlannerType

Specifies the type of planner an agent uses.

```kotlin
enum class PlannerType {
    GOAP,       // Goal-Oriented Action Planning (default)
    UTILITY,    // Utility-based selection
    SUPERVISOR  // LLM-orchestrated planning
}
```

---

## 4. Core Types

### Package: `com.embabel.agent.core`

#### AgentPlatform

The central interface for running agents. Can also act as an agent itself.

```kotlin
interface AgentPlatform : AgentScope {
    fun run(agent: Agent, input: Any?, options: ProcessOptions): AgentProcess
    fun deploy(agent: Agent): AgentPlatform
    fun typedOps(): TypedOps
}
```

**Key Methods:**
| Method | Returns | Description |
|--------|---------|-------------|
| `run(agent, input, options)` | `AgentProcess` | Execute an agent |
| `deploy(agent)` | `AgentPlatform` | Deploy agent to platform |
| `typedOps()` | `TypedOps` | Get typed operations API |

---

#### AgentProcess

Represents a running instance of an agent.

```kotlin
interface AgentProcess : Blackboard, 
    OperationStatus<AgentProcessStatusCode>, 
    LlmInvocationHistory
```

**Status Codes:**
| Status | Description |
|--------|-------------|
| `NOT_STARTED` | Process has not started |
| `RUNNING` | Process is executing |
| `COMPLETED` | Process completed successfully |
| `FAILED` | Process failed |
| `TERMINATED` | Process terminated by policy |
| `KILLED` | Process killed by user |
| `STUCK` | Cannot formulate a plan |
| `WAITING` | Waiting for user input |
| `PAUSED` | Paused due to scheduling |

---

#### Blackboard

Shared memory system for maintaining state throughout agent execution.

```kotlin
interface Blackboard : Bindable, MayHaveLastResult {
    fun <T> get(clazz: Class<T>): T?
    fun <T> get(name: String, clazz: Class<T>): T?
    fun addObject(value: Any): Blackboard
    fun addObject(name: String, value: Any): Blackboard
    fun <T> objectsOfType(clazz: Class<T>): List<T>
    fun <T> last(clazz: Class<T>): T?
    fun <T> count(clazz: Class<T>): Int
}
```

**Key Methods:**
| Method | Returns | Description |
|--------|---------|-------------|
| `get(clazz)` | `T?` | Get object by type |
| `get(name, clazz)` | `T?` | Get object by name and type |
| `addObject(value)` | `Blackboard` | Add object to blackboard |
| `objectsOfType(clazz)` | `List<T>` | Get all objects of type |
| `last(clazz)` | `T?` | Get most recent object of type |

---

#### Agent

Data class representing an agent definition.

```kotlin
data class Agent(
    val name: String,
    val provider: String,
    val version: Semver = Semver(),
    val description: String,
    val conditions: Set<Condition> = emptySet(),
    val actions: List<Action> = emptyList(),
    val goals: Set<Goal> = emptySet(),
    val stuckHandler: StuckHandler? = null,
    val opaque: Boolean = false,
    val domainTypes: Collection<DomainType> = emptyList()
)
```

---

#### Goal

Represents an agent platform goal with GOAP metadata.

```kotlin
data class Goal(
    val name: String,
    val description: String,
    val pre: Set<String> = emptySet(),
    val inputs: Set<IoBinding> = emptySet(),
    val outputType: DomainType? = null,
    val value: CostComputation = { 0.0 },
    val tags: Set<String> = emptySet(),
    val examples: Set<String> = emptySet(),
    val export: Export = Export()
)
```

---

#### Action (Core)

Core action model in the agent system.

```kotlin
interface Action : DataFlowStep, ConditionAction, ActionRunner, 
    DataDictionary, ToolGroupConsumer {
    val name: String
    val description: String
    val inputs: Set<IoBinding>
    val outputs: Set<IoBinding>
    val preconditions: EffectSpec
    val effects: EffectSpec
    val canRerun: Boolean
    val readOnly: Boolean
    val qos: ActionQos
}
```

---

#### ProcessOptions

Configuration for how to run an AgentProcess.

```kotlin
data class ProcessOptions(
    val contextId: ContextId? = null,
    val identities: Identities = Identities(),
    val blackboard: Blackboard? = null,
    val verbosity: Verbosity = Verbosity(),
    val budget: Budget = Budget(),
    val processControl: ProcessControl = ProcessControl(),
    val prune: Boolean = false,
    val listeners: List<AgenticEventListener> = emptyList(),
    val outputChannel: OutputChannel = DevNullOutputChannel,
    val plannerType: PlannerType = PlannerType.GOAP
)
```

**Builder Methods:**
```kotlin
val options = ProcessOptions()
    .withVerbosity(Verbosity().withShowPrompts(true))
    .withBudget(Budget(cost = 10.0, actions = 50))
    .withProcessControl(ProcessControl(
        earlyTerminationPolicy = EarlyTerminationPolicy.maxActions(100)
    ))
```

---

#### Budget

Budget constraints for an agent process.

```kotlin
data class Budget(
    val cost: Double = DEFAULT_COST_LIMIT,
    val actions: Int = DEFAULT_ACTION_LIMIT,
    val tokens: Int = DEFAULT_TOKEN_LIMIT
)
```

---

#### Verbosity

Controls log output detail.

```kotlin
data class Verbosity(
    val showPrompts: Boolean = false,
    val showLlmResponses: Boolean = false,
    val debug: Boolean = false,
    val showPlanning: Boolean = false
) : LlmVerbosity
```

---

## 5. Tool API

### Package: `com.embabel.agent.api.tool`

#### Tool

Framework-agnostic tool that can be invoked by an LLM.

```kotlin
interface Tool : ToolInfo {
    suspend fun call(input: Tool.Input): Tool.Result
    
    interface Input {
        fun text(): String
        fun json(): JsonNode
        fun <T> parse(clazz: Class<T>): T
    }
    
    interface Result {
        fun text(): String
        fun json(): JsonNode
        
        companion object {
            fun text(content: String): Result
            fun json(content: JsonNode): Result
            fun error(message: String): Result
        }
    }
}
```

**Factory Methods (Tool.Companion):**
| Method | Returns | Description |
|--------|---------|-------------|
| `create(name, description, function)` | `Tool` | Create simple tool |
| `fromFunction(name, description, inputClass, outputClass, function)` | `Tool` | Create typed tool |
| `fromInstance(instance)` | `List<Tool>` | Create tools from @LlmTool annotated object |
| `fromMethod(method, instance)` | `Tool` | Create tool from single method |

---

#### TypedTool

Tool with strongly typed input and output.

```kotlin
open class TypedTool<I : Any, O : Any>(
    name: String,
    description: String,
    inputType: Class<I>,
    outputType: Class<O>,
    val metadata: Tool.Metadata = Tool.Metadata.DEFAULT,
    objectMapper: ObjectMapper = jacksonObjectMapper(),
    function: Function<I, O>
) : Tool
```

**Example:**
```java
Tool addTool = Tool.fromFunction(
    "add",
    "Adds two numbers",
    AddRequest.class,
    AddResult.class,
    request -> new AddResult(request.a() + request.b())
);
```

---

#### Tool.InputSchema

Schema definition for tool inputs.

```kotlin
interface InputSchema {
    fun toJsonSchema(): JsonNode
    
    companion object {
        fun of(vararg parameters: Parameter): InputSchema
    }
}
```

---

#### Tool.Parameter

Parameter definition for tool inputs.

```kotlin
interface Parameter {
    val name: String
    val description: String
    val type: String
    val required: Boolean
    
    companion object {
        fun string(name: String, description: String, required: Boolean = true): Parameter
        fun integer(name: String, description: String, required: Boolean = true): Parameter
        fun number(name: String, description: String, required: Boolean = true): Parameter
        fun boolean(name: String, description: String, required: Boolean = true): Parameter
    }
}
```

---

#### Subagent

Tool that delegates to another agent as a subagent/handoff.

```kotlin
class Subagent : Tool {
    companion object {
        fun ofClass(agentClass: Class<*>): SubagentBuilder
        fun ofAgent(agent: Agent): SubagentBuilder
    }
}
```

**Example:**
```java
context.ai()
    .withDefaultLlm()
    .withTool(Subagent.ofClass(PerformanceFinder.class)
                     .consuming(WorksToFind.class))
    .creating(Concert.class)
    .fromPrompt("Assemble a concert");
```

---

## 6. Agentic Tools

### Package: `com.embabel.agent.api.tool.agentic`

#### AgenticTool

An agentic tool that uses an LLM to orchestrate other tools.

```kotlin
data class AgenticTool(
    val definition: Tool.Definition,
    val metadata: Tool.Metadata = Tool.Metadata.DEFAULT,
    val llm: LlmOptions = LlmOptions(),
    val tools: List<Tool> = emptyList(),
    val systemPromptCreator: SystemPromptCreator = { defaultSystemPrompt(definition.description) },
    val captureNestedArtifacts: Boolean = false
) : Tool
```

---

#### SimpleAgenticTool

Simple agentic tool with all tools available immediately.

```kotlin
class SimpleAgenticTool(
    name: String,
    description: String
) {
    fun withTools(vararg tools: Tool): SimpleAgenticTool
    fun withParameter(parameter: Tool.Parameter): SimpleAgenticTool
    fun withLlm(llm: LlmOptions): SimpleAgenticTool
}
```

**Example:**
```java
SimpleAgenticTool mathOrchestrator = new SimpleAgenticTool(
    "math-orchestrator",
    "Orchestrates math operations"
)
.withTools(addTool, multiplyTool, divideTool)
.withParameter(Tool.Parameter.string("expression", "Math expression"))
.withLlm(LlmOptions.withModel("gpt-4"));
```

---

#### PlaybookTool

Progressive tool unlock via conditions.

```kotlin
class PlaybookTool(
    name: String,
    description: String
) {
    fun withTools(vararg tools: Tool): PlaybookTool
    fun withTool(tool: Tool): PlaybookToolBuilder
}

class PlaybookToolBuilder {
    fun unlockedBy(prerequisiteTool: Tool): PlaybookTool
}
```

**Example:**
```java
PlaybookTool researcher = new PlaybookTool("researcher", "Research topics")
    .withTools(searchTool, fetchTool)
    .withTool(analyzeTool).unlockedBy(searchTool)
    .withTool(summarizeTool).unlockedBy(analyzeTool);
```

---

#### StateMachineTool

State-based tool availability.

```kotlin
class StateMachineTool<S : Enum<S>>(
    name: String,
    description: String,
    stateClass: Class<S>
) {
    fun withInitialState(state: S): StateMachineTool<S>
    fun inState(state: S): StateBuilder<S>
}
```

---

## 7. Invocation API

### Package: `com.embabel.agent.api.invocation`

#### AgentInvocation

Defines the contract for invoking an agent.

```kotlin
interface AgentInvocation<T : Any> : TypedInvocation<T> {
    fun invoke(input: Any?): T
    fun invokeAsync(input: Any?): CompletableFuture<T>
}
```

**Factory Methods:**
```kotlin
// Create invocation
val invocation = AgentInvocation.create(agentPlatform, ResultType::class.java)

// Build with options
val invocation = AgentInvocation.builder(agentPlatform)
    .options(ProcessOptions().withVerbosity(Verbosity().withShowPrompts(true)))
    .build(ResultType::class.java)
```

---

#### TypedInvocation

Invocation with a specific return type.

```kotlin
interface TypedInvocation<T : Any, THIS : TypedInvocation<T, THIS>> : BaseInvocation<THIS> {
    fun invoke(input: Any?): T
    fun invokeAsync(input: Any?): CompletableFuture<T>
}
```

---

#### SupervisorInvocation

Invoker for supervisor-orchestrated agents.

```kotlin
data class SupervisorInvocation<T : Any>(
    agentPlatform: AgentPlatform,
    goalType: Class<T>,
    goalDescription: String = "Produce ",
    processOptions: ProcessOptions = ProcessOptions(),
    agentScopeBuilder: AgentScopeBuilder = agentPlatform,
    agentName: String? = null
) : TypedInvocation<T, SupervisorInvocation<T>>
```

---

#### UtilityInvocation

Invoker for utility-based agents.

```kotlin
data class UtilityInvocation(
    agentPlatform: AgentPlatform,
    processOptions: ProcessOptions = ProcessOptions(),
    agentScopeBuilder: AgentScopeBuilder = agentPlatform,
    agentName: String? = null
) : BaseInvocation<UtilityInvocation>, ScopedInvocation<UtilityInvocation>
```

---

## 8. Event System

### Package: `com.embabel.agent.api.event`

#### AgenticEvent

Root of the event hierarchy.

```kotlin
sealed interface AgenticEvent
```

**Event Types:**
| Event | Description |
|-------|-------------|
| `AgentPlatformEvent` | System events like deployment |
| `AgentProcessEvent` | Events related to a specific process |

---

#### AgentProcessEvent

Events relating to a specific agent process.

```kotlin
interface AgentProcessEvent : AgenticEvent, InProcess {
    val agentProcess: AgentProcess
}
```

**Key Events:**
| Event | Description |
|-------|-------------|
| `AgentProcessCreationEvent` | Process created |
| `AgentProcessCompletedEvent` | Process completed successfully |
| `AgentProcessFailedEvent` | Process failed |
| `AgentProcessStuckEvent` | Process unable to plan |
| `AgentProcessWaitingEvent` | Process waiting for input |
| `GoalAchievedEvent` | Goal was achieved |
| `ActionExecutionStartEvent` | Action started |
| `ActionExecutionResultEvent` | Action completed |
| `LlmRequestEvent` | LLM request made |
| `LlmResponseEvent` | LLM response received |
| `ToolCallRequestEvent` | Tool call requested |
| `ToolCallResponseEvent` | Tool call responded |

---

#### AgenticEventListener

Listen to events related to processes and the platform.

```kotlin
interface AgenticEventListener {
    fun onAgentProcessCreation(event: AgentProcessCreationEvent)
    fun onAgentProcessCompleted(event: AgentProcessCompletedEvent)
    fun onAgentProcessFailed(event: AgentProcessFailedEvent)
    fun onGoalAchieved(event: GoalAchievedEvent)
    fun onActionExecutionStart(event: ActionExecutionStartEvent)
    fun onActionExecutionResult(event: ActionExecutionResultEvent)
    fun onLlmRequest(event: LlmRequestEvent<*>)
    fun onLlmResponse(event: LlmResponseEvent<*>)
    fun onToolCallRequest(event: ToolCallRequestEvent)
    fun onToolCallResponse(event: ToolCallResponseEvent)
    // ... and more
}
```

---

## 9. Chat API

### Package: `com.embabel.chat`

#### Chatbot

Interface for chatbot implementations.

```kotlin
interface Chatbot {
    fun createSession(
        user: User,
        outputChannel: OutputChannel,
        conversationId: String? = null,
        contextId: ContextId? = null
    ): ChatSession
}
```

**Factory Methods:**
```kotlin
// Utility-based chatbot
val chatbot = AgentProcessChatbot.utilityFromPlatform(agentPlatform)

// GOAP-based chatbot
val chatbot = AgentProcessChatbot.goapFromPlatform(agentPlatform)
```

---

#### ChatSession

Conversation session implementation.

```kotlin
interface ChatSession {
    fun onUserMessage(message: UserMessage)
    fun conversation(): Conversation
    fun conversationId(): String
}
```

---

#### Conversation

Mutable conversation shim for agent system.

```kotlin
interface Conversation : AssetView {
    fun messages(): List<Message>
    fun addMessage(message: Message): Conversation
    fun lastMessage(): Message?
    fun messageCount(): Int
}
```

---

#### Message Types

| Type | Description |
|------|-------------|
| `UserMessage` | Message from user (supports multimodal) |
| `AssistantMessage` | Message from assistant |
| `AssistantMessageWithToolCalls` | Assistant message with tool calls |
| `SystemMessage` | System message |
| `ToolResultMessage` | Result of tool execution |

---

#### MessageRole

Role of the message sender.

```kotlin
enum class MessageRole {
    USER,
    ASSISTANT,
    SYSTEM,
    TOOL
}
```

---

## 10. DSL

### Package: `com.embabel.agent.api.dsl`

#### agent() Function

Create an agent using Kotlin DSL.

```kotlin
fun agent(
    name: String,
    provider: String = "embabel",
    version: Semver = Semver(),
    description: String,
    promptContributors: List<PromptContributor> = emptyList(),
    block: AgentBuilder.() -> Unit
): Agent
```

**Example:**
```kotlin
val myAgent = agent(
    name = "story-writer",
    description = "Writes creative stories"
) {
    action<StoryRequest, Story>(
        name = "writeStory",
        description = "Write a story based on request"
    ) { context ->
        context.ai().withDefaultLlm()
            .createObject("Write a story about: ${input.topic}", Story::class.java)
    }
    
    goal<Story>(
        name = "produceStory",
        description = "Produce a complete story"
    )
}
```

---

#### chain() Function

Chain actions together.

```kotlin
inline fun <A, B, C> chain(
    noinline a: (context: InputActionContext<A>) -> B,
    noinline b: (context: InputActionContext<B>) -> C
): TypedAgentScopeBuilder<C>
```

---

#### andThen() Function

Compose functions sequentially.

```kotlin
inline fun <A, B, C> Function<A, B>.andThen(
    crossinline that: (B) -> C
): TypedAgentScopeBuilder<C>
```

---

#### aggregate() Function

Run multiple transforms and merge results.

```kotlin
inline fun <A, B, C> aggregate(
    transforms: List<(context: InputActionContext<A>) -> B>,
    noinline merge: (list: List<B>, context: OperationContext) -> C
): TypedAgentScopeBuilder<C>
```

---

#### branch() Function

Branch execution based on conditions.

```kotlin
inline fun <A, B, C> branch(
    noinline a: (context: InputActionContext<A>) -> Branch<B, C>
): TypedAgentScopeBuilder<Branch<B, C>>
```

---

#### repeat() Function

Repeat actions until condition is met.

```kotlin
inline fun <C> repeat(
    noinline what: () -> TypedAgentScopeBuilder<C>,
    noinline until: (c: C, context: OperationContext) -> Boolean
): TypedAgentScopeBuilder<C>
```

---

## 11. Workflow API

### Package: `com.embabel.agent.api.common.workflow`

#### WorkflowBuilder

Common base class for building workflows.

```kotlin
abstract class WorkflowBuilder<RESULT : Any>(
    resultClass: Class<RESULT>,
    inputClass: Class<out Any>?
)
```

---

#### WorkflowBuilderConsuming

Ensures consistent naming for builders that consume input.

```kotlin
interface WorkflowBuilderConsuming<T>
```

---

#### WorkflowBuilderReturning

Ensures consistent naming for builders that return results.

```kotlin
interface WorkflowBuilderReturning<T>
```

---

## 12. Thinking API

### Package: `com.embabel.agent.api.common.thinking`

#### ThinkingPromptRunnerOperations

Interface for executing prompts with thinking block extraction.

```kotlin
interface ThinkingPromptRunnerOperations {
    fun <T> createObject(prompt: String, clazz: Class<T>): ThinkingResponse<T>
}
```

---

#### ThinkingResponse

Response containing both result and thinking blocks.

```kotlin
class ThinkingResponse<T> {
    fun getResult(): T
    fun getThinkingBlocks(): List<ThinkingBlock>
}
```

**Example:**
```java
ThinkingResponse<MonthItem> response = runner.thinking()
    .createObject("What is the hottest month in Florida?", MonthItem.class);

MonthItem result = response.getResult();
List<ThinkingBlock> thinkingBlocks = response.getThinkingBlocks();
```

---

## 13. Validation API

### Package: `com.embabel.agent.api.validation`

#### ContentValidator

Generic validation interface.

```kotlin
interface ContentValidator<T> {
    fun validate(content: T): ValidationResult
}
```

---

#### AgentValidationManager

Central validation manager.

```kotlin
interface AgentValidationManager
```

---

## 14. Model Constants

### Package: `com.embabel.agent.api.models`

#### OpenAiModels

```kotlin
class OpenAiModels {
    companion object {
        const val GPT_4O = "gpt-4o"
        const val GPT_4O_MINI = "gpt-4o-mini"
        const val GPT_41 = "gpt-4.1"
        const val GPT_41_MINI = "gpt-4.1-mini"
        const val GPT_41_NANO = "gpt-4.1-nano"
        const val O1 = "o1"
        const val O1_MINI = "o1-mini"
        const val O3_MINI = "o3-mini"
        const val O4_MINI = "o4-mini"
        const val GPT_45_PREVIEW = "gpt-4.5-preview"
        const val TEXT_EMBEDDING_3_SMALL = "text-embedding-3-small"
        const val TEXT_EMBEDDING_3_LARGE = "text-embedding-3-large"
    }
}
```

---

#### AnthropicModels

```kotlin
class AnthropicModels {
    companion object {
        const val CLAUDE_35_SONNET = "claude-3-5-sonnet-20241022"
        const val CLAUDE_35_HAIKU = "claude-3-5-haiku-20241022"
        const val CLAUDE_37_SONNET = "claude-3-7-sonnet-20250219"
        const val CLAUDE_SONNET_4_5 = "claude-sonnet-4-5-20251001"
    }
}
```

---

#### GeminiModels

```kotlin
class GeminiModels {
    companion object {
        const val GEMINI_2_0_FLASH = "gemini-2.0-flash"
        const val GEMINI_2_0_FLASH_LITE = "gemini-2.0-flash-lite"
        const val GEMINI_2_5_FLASH = "gemini-2.5-flash"
        const val GEMINI_2_5_PRO = "gemini-2.5-pro"
        const val GEMINI_1_5_FLASH = "gemini-1.5-flash"
        const val GEMINI_1_5_PRO = "gemini-1.5-pro"
        const val TEXT_EMBEDDING_004 = "text-embedding-004"
    }
}
```

---

#### OllamaModels

```kotlin
class OllamaModels {
    companion object {
        const val LLAMA_3_1 = "llama3.1"
        const val LLAMA_3_2 = "llama3.2"
        const val LLAMA_3_3 = "llama3.3"
        const val MISTRAL = "mistral"
        const val QWEN_2_5 = "qwen2.5"
        const val QWEN_3 = "qwen3"
        const val DEEPSEEK_R1 = "deepseek-r1"
        const val PHI_4 = "phi4"
    }
}
```

---

## 15. Complete Class Reference

### Annotation Classes

| Class | Package | Description |
|-------|---------|-------------|
| `@Agent` | `api.annotation` | Marks class as agent |
| `@Action` | `api.annotation` | Marks method as action |
| `@AchievesGoal` | `api.annotation` | Marks action as achieving goal |
| `@EmbabelComponent` | `api.annotation` | Marks class as component |
| `@State` | `api.annotation` | Marks class as state |
| `@LlmTool` | `api.annotation` | Marks method as LLM tool |
| `@Provided` | `api.annotation` | Marks parameter as provided |
| `@Cost` | `api.annotation` | Marks cost computation method |
| `@Condition` | `api.annotation` | Marks condition method |
| `@Export` | `api.annotation` | Specifies goal export |
| `@ToolGroup` | `api.annotation` | Specifies tool group role |

### Core Interfaces

| Interface | Package | Description |
|-----------|---------|-------------|
| `AgentPlatform` | `core` | Central platform for running agents |
| `AgentProcess` | `core` | Running agent instance |
| `Blackboard` | `core` | Shared memory system |
| `Action` | `core` | Core action model |
| `Goal` | `core` | Goal definition |
| `Condition` | `core` | Condition model |
| `OperationContext` | `api.common` | Operation context |
| `ActionContext` | `api.common` | Action context |
| `Ai` | `api.common` | AI functionality gateway |
| `PromptRunner` | `api.common` | LLM prompt execution |
| `Tool` | `api.tool` | Framework-agnostic tool |
| `TypedTool` | `api.tool` | Strongly typed tool |
| `AgentInvocation` | `api.invocation` | Agent invocation |
| `Chatbot` | `chat` | Chatbot interface |
| `ChatSession` | `chat` | Chat session |
| `Conversation` | `chat` | Conversation management |

### Data Classes

| Class | Package | Description |
|-------|---------|-------------|
| `Agent` | `core` | Agent definition |
| `Goal` | `core` | Goal definition |
| `ActionMetadata` | `core` | Action metadata |
| `AgentMetadata` | `core` | Agent metadata |
| `ProcessOptions` | `core` | Process configuration |
| `Budget` | `core` | Budget constraints |
| `Verbosity` | `core` | Log verbosity |
| `ProcessControl` | `core` | Process control |
| `AgentImage` | `api.common` | Image representation |
| `MultimodalContent` | `api.common` | Multimodal content |
| `Tool.Input` | `api.tool` | Tool input |
| `Tool.Result` | `api.tool` | Tool result |
| `Tool.Definition` | `api.tool` | Tool definition |
| `Tool.Metadata` | `api.tool` | Tool metadata |
| `Subagent` | `api.tool` | Subagent tool |
| `AgenticTool` | `api.tool.agentic` | Agentic tool |
| `UserMessage` | `chat` | User message |
| `AssistantMessage` | `chat` | Assistant message |
| `SystemMessage` | `chat` | System message |
| `ToolCall` | `chat` | Tool call |
| `ToolResultMessage` | `chat` | Tool result message |

### Enums

| Enum | Package | Values |
|------|---------|--------|
| `PlannerType` | `api.common` | `GOAP`, `UTILITY`, `SUPERVISOR` |
| `AgentProcessStatusCode` | `core` | `NOT_STARTED`, `RUNNING`, `COMPLETED`, `FAILED`, `TERMINATED`, `KILLED`, `STUCK`, `WAITING`, `PAUSED` |
| `ActionStatusCode` | `core` | `PENDING`, `RUNNING`, `COMPLETED`, `FAILED` |
| `MessageRole` | `chat` | `USER`, `ASSISTANT`, `SYSTEM`, `TOOL` |
| `ConversationStoreType` | `chat` | `IN_MEMORY`, `STORED` |
| `ActionRetryPolicy` | `core` | `DEFAULT`, `IMMEDIATE`, `EXPONENTIAL_BACKOFF` |
| `Cardinality` | `core` | `ONE`, `LIST`, `SET` |
| `Delay` | `core` | `NONE`, `SHORT`, `MEDIUM`, `LONG` |

### Events

| Event | Package | Description |
|-------|---------|-------------|
| `AgenticEvent` | `api.event` | Root event interface |
| `AgentPlatformEvent` | `api.event` | Platform events |
| `AgentProcessEvent` | `api.event` | Process events |
| `AgentProcessCreationEvent` | `api.event` | Process created |
| `AgentProcessCompletedEvent` | `api.event` | Process completed |
| `AgentProcessFailedEvent` | `api.event` | Process failed |
| `AgentProcessStuckEvent` | `api.event` | Process stuck |
| `GoalAchievedEvent` | `api.event` | Goal achieved |
| `ActionExecutionStartEvent` | `api.event` | Action started |
| `ActionExecutionResultEvent` | `api.event` | Action completed |
| `LlmRequestEvent` | `api.event` | LLM request |
| `LlmResponseEvent` | `api.event` | LLM response |
| `ToolCallRequestEvent` | `api.event` | Tool call request |
| `ToolCallResponseEvent` | `api.event` | Tool call response |

### Exceptions

| Exception | Package | Description |
|-----------|---------|-------------|
| `SpecialReturnException` | `api.annotation` | Base for special returns |
| `SubagentExecutionRequest` | `api.annotation` | Subagent execution signal |
| `ReplanRequestedException` | `core` | Replanning signal |
| `NoSuchAgentException` | `api.common` | Agent not found |
| `ToolControlFlowSignal` | `api.tool` | Tool control flow |

---

## Quick Reference: Common Patterns

### Creating an Agent

```java
@Agent(description = "My agent")
public class MyAgent {
    
    @Action
    public Output process(Input input, OperationContext context) {
        return context.ai()
            .withDefaultLlm()
            .createObject("Process: " + input, Output.class);
    }
    
    @AchievesGoal(description = "Complete processing")
    @Action
    public FinalOutput finalize(Output output) {
        return new FinalOutput(output);
    }
}
```

### Using Tools

```java
@Action
public Result process(Input input, OperationContext context) {
    return context.ai()
        .withDefaultLlm()
        .withToolObject(new MyTools())
        .withToolGroup("web")
        .createObject("Process with tools: " + input, Result.class);
}
```

### Invoking an Agent

```java
// Typed invocation
AgentInvocation<Result> invocation = AgentInvocation.create(platform, Result.class);
Result result = invocation.invoke(input);

// Async invocation
CompletableFuture<Result> future = invocation.invokeAsync(input);

// With options
AgentInvocation<Result> invocation = AgentInvocation.builder(platform)
    .options(new ProcessOptions()
        .withVerbosity(new Verbosity().withShowPrompts(true)))
    .build(Result.class);
```

### Handling Events

```java
@Component
public class MyEventListener implements AgenticEventListener {
    @Override
    public void onGoalAchieved(GoalAchievedEvent event) {
        System.out.println("Goal achieved: " + event.getGoal().getName());
    }
    
    @Override
    public void onLlmRequest(LlmRequestEvent<?> event) {
        System.out.println("LLM request: " + event.getInteraction().getLlm());
    }
}
```

---

## Resources

- **Full API Docs**: [docs.embabel.com/embabel-agent/api-docs](https://docs.embabel.com/embabel-agent/api-docs)
- **User Guide**: [docs.embabel.com/embabel-agent/guide](https://docs.embabel.com/embabel-agent/guide)
- **GitHub Examples**: [github.com/embabel/embabel-agent-examples](https://github.com/embabel/embabel-agent-examples)

---

**© 2024-2026 Embabel Pty, Ltd**
