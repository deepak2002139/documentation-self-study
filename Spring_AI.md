# Spring AI - Complete Implementation Guide

## Table of Contents
1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Project Setup](#project-setup)
4. [Configuration](#configuration)
5. [Core Concepts](#core-concepts)
6. [Implementation Examples](#implementation-examples)
7. [Advanced Features](#advanced-features)
8. [Best Practices](#best-practices)
9. [Troubleshooting](#troubleshooting)
10. [Resources](#resources)

---

## Introduction

Spring AI is a framework that provides abstractions for integrating AI capabilities into Spring applications. It offers a unified API for working with multiple AI models and services including OpenAI, Azure OpenAI, Hugging Face, Ollama, and more.

### Key Features
- **Model Portability**: Switch between AI providers with minimal code changes
- **Chat & Text Generation**: Support for conversational AI and text completion
- **Embeddings**: Vector representations for semantic search
- **Image Generation**: AI-powered image creation
- **Function Calling**: Enable AI models to call Java methods
- **RAG Support**: Retrieval Augmented Generation with vector stores
- **Prompt Templates**: Reusable prompt management

---

## Prerequisites

### Required
- Java 17 or higher
- Maven 3.6+ or Gradle 7.5+
- Spring Boot 3.2.0 or higher
- IDE (IntelliJ IDEA, VS Code, or Eclipse)

### AI Provider Account (Choose one or more)
- OpenAI API key (https://platform.openai.com)
- Azure OpenAI credentials
- Hugging Face API key
- Ollama (local installation)
- Anthropic Claude API key
- Google Vertex AI credentials

---

## Project Setup

### Step 1: Create a New Spring Boot Project

#### Using Spring Initializr (Web)
1. Go to https://start.spring.io
2. Select:
   - Project: Maven or Gradle
   - Language: Java
   - Spring Boot: 3.2.0+
   - Java: 17+
3. Add Dependencies:
   - Spring Web
   - Spring Boot DevTools (optional)

#### Using Spring CLI
```bash
spring init --dependencies=web,devtools spring-ai-demo
cd spring-ai-demo
```

#### Using Maven Archetype
```bash
mvn archetype:generate \
  -DgroupId=com.example \
  -DartifactId=spring-ai-demo \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DinteractiveMode=false
```

### Step 2: Add Spring AI Dependencies

#### Maven (pom.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>spring-ai-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>spring-ai-demo</name>
    <description>Spring AI Demo Project</description>
    
    <properties>
        <java.version>17</java.version>
        <spring-ai.version>1.0.0-M3</spring-ai.version>
    </properties>
    
    <dependencies>
        <!-- Spring Boot Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <!-- Spring AI OpenAI -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
        </dependency>
        
        <!-- Spring AI Ollama (for local models) -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-ollama-spring-boot-starter</artifactId>
        </dependency>
        
        <!-- Spring AI Vector Store (for RAG) -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
        </dependency>
        
        <!-- Spring Boot DevTools -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>
        
        <!-- Lombok (optional, for cleaner code) -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        
        <!-- Test Dependencies -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.ai</groupId>
                <artifactId>spring-ai-bom</artifactId>
                <version>${spring-ai.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
    
    <!-- Spring Milestones Repository -->
    <repositories>
        <repository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>https://repo.spring.io/milestone</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
        <repository>
            <id>spring-snapshots</id>
            <name>Spring Snapshots</name>
            <url>https://repo.spring.io/snapshot</url>
            <releases>
                <enabled>false</enabled>
            </releases>
        </repository>
    </repositories>
</project>
```

#### Gradle (build.gradle)

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '17'

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
    maven { url 'https://repo.spring.io/milestone' }
    maven { url 'https://repo.spring.io/snapshot' }
}

ext {
    set('springAiVersion', "1.0.0-M3")
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.ai:spring-ai-openai-spring-boot-starter'
    implementation 'org.springframework.ai:spring-ai-ollama-spring-boot-starter'
    implementation 'org.springframework.ai:spring-ai-pgvector-store-spring-boot-starter'
    
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    
    developmentOnly 'org.springframework.boot:spring-boot-devtools'
    
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.ai:spring-ai-bom:${springAiVersion}"
    }
}

tasks.named('test') {
    useJUnitPlatform()
}
```

---

## Configuration

### Step 3: Application Configuration

Create or update `src/main/resources/application.properties` or `application.yml`

#### application.properties

```properties
# Server Configuration
server.port=8080
spring.application.name=spring-ai-demo

# OpenAI Configuration
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.chat.options.model=gpt-4
spring.ai.openai.chat.options.temperature=0.7
spring.ai.openai.chat.options.max-tokens=500

# Ollama Configuration (for local models)
spring.ai.ollama.base-url=http://localhost:11434
spring.ai.ollama.chat.options.model=llama2
spring.ai.ollama.chat.options.temperature=0.7

# Azure OpenAI Configuration (if using Azure)
spring.ai.azure.openai.api-key=${AZURE_OPENAI_API_KEY}
spring.ai.azure.openai.endpoint=${AZURE_OPENAI_ENDPOINT}
spring.ai.azure.openai.chat.options.deployment-name=gpt-4

# Vector Store Configuration (PostgreSQL with pgvector)
spring.datasource.url=jdbc:postgresql://localhost:5432/vectordb
spring.datasource.username=postgres
spring.datasource.password=postgres
spring.ai.vectorstore.pgvector.dimensions=1536
spring.ai.vectorstore.pgvector.distance-type=COSINE_DISTANCE

# Logging
logging.level.org.springframework.ai=DEBUG
logging.level.com.example=DEBUG
```

#### application.yml

```yaml
server:
  port: 8080

spring:
  application:
    name: spring-ai-demo
  
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4
          temperature: 0.7
          max-tokens: 500
    
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama2
          temperature: 0.7
    
    azure:
      openai:
        api-key: ${AZURE_OPENAI_API_KEY}
        endpoint: ${AZURE_OPENAI_ENDPOINT}
        chat:
          options:
            deployment-name: gpt-4
    
    vectorstore:
      pgvector:
        dimensions: 1536
        distance-type: COSINE_DISTANCE
  
  datasource:
    url: jdbc:postgresql://localhost:5432/vectordb
    username: postgres
    password: postgres

logging:
  level:
    org.springframework.ai: DEBUG
    com.example: DEBUG
```

### Step 4: Set Environment Variables

Create a `.env` file or set system environment variables:

```bash
# .env file (use with spring-boot-dotenv or docker-compose)
OPENAI_API_KEY=sk-your-openai-api-key-here
AZURE_OPENAI_API_KEY=your-azure-key
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com
```

Or export in terminal:
```bash
export OPENAI_API_KEY=sk-your-openai-api-key-here
```

---

## Core Concepts

### ChatModel
The primary interface for conversational AI. Supports single messages or conversation history.

### StreamingChatModel
Enables streaming responses for real-time output.

### EmbeddingModel
Generates vector embeddings for text, used in semantic search and RAG.

### ImageModel
Creates images from text descriptions.

### VectorStore
Stores and retrieves embeddings for similarity search.

### Prompt
Template-based prompt management with variable substitution.

---

## Implementation Examples

### Example 1: Basic Chat Completion

#### Controller Class

```java
package com.example.springai.controller;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.ChatResponse;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/chat")
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    @PostMapping("/simple")
    public String simpleChat(@RequestBody String message) {
        return chatClient.call(message);
    }

    @PostMapping("/detailed")
    public ChatResponse detailedChat(@RequestBody String message) {
        Prompt prompt = new Prompt(new UserMessage(message));
        return chatClient.call(prompt);
    }
}
```

#### Request Example
```bash
curl -X POST http://localhost:8080/api/chat/simple \
  -H "Content-Type: text/plain" \
  -d "What is Spring AI?"
```

### Example 2: Streaming Chat Response

```java
package com.example.springai.controller;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.StreamingChatClient;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;

@RestController
@RequestMapping("/api/stream")
public class StreamingChatController {

    private final StreamingChatClient streamingChatClient;

    public StreamingChatController(StreamingChatClient streamingChatClient) {
        this.streamingChatClient = streamingChatClient;
    }

    @GetMapping(value = "/chat", produces = "text/event-stream")
    public Flux<String> streamChat(@RequestParam String message) {
        Prompt prompt = new Prompt(new UserMessage(message));
        
        return streamingChatClient.stream(prompt)
                .map(response -> response.getResult().getOutput().getContent());
    }
}
```

#### Test with Browser or curl
```bash
curl -N http://localhost:8080/api/stream/chat?message=Tell%20me%20a%20story
```

### Example 3: Prompt Templates

```java
package com.example.springai.service;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.PromptTemplate;
import org.springframework.stereotype.Service;

import java.util.Map;

@Service
public class PromptTemplateService {

    private final ChatClient chatClient;

    public PromptTemplateService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String generateCodeExplanation(String code, String language) {
        String template = """
                Explain the following {language} code:
                
                ```{language}
                {code}
                ```
                
                Provide a detailed explanation including:
                1. What the code does
                2. Key concepts used
                3. Potential improvements
                """;

        PromptTemplate promptTemplate = new PromptTemplate(template);
        Prompt prompt = promptTemplate.create(Map.of(
                "code", code,
                "language", language
        ));

        return chatClient.call(prompt).getResult().getOutput().getContent();
    }

    public String translateText(String text, String targetLanguage) {
        String template = "Translate the following text to {language}: {text}";
        
        PromptTemplate promptTemplate = new PromptTemplate(template);
        Prompt prompt = promptTemplate.create(Map.of(
                "text", text,
                "language", targetLanguage
        ));

        return chatClient.call(prompt).getResult().getOutput().getContent();
    }
}
```

### Example 4: Conversation Memory

```java
package com.example.springai.service;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.AssistantMessage;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class ConversationService {

    private final ChatClient chatClient;
    private final ConcurrentHashMap<String, List<Message>> conversationHistory;

    public ConversationService(ChatClient chatClient) {
        this.chatClient = chatClient;
        this.conversationHistory = new ConcurrentHashMap<>();
    }

    public String chat(String sessionId, String userMessage) {
        // Get or create conversation history
        List<Message> messages = conversationHistory.computeIfAbsent(
                sessionId, 
                k -> new ArrayList<>()
        );

        // Add user message
        messages.add(new UserMessage(userMessage));

        // Create prompt with full history
        Prompt prompt = new Prompt(messages);

        // Get response
        String response = chatClient.call(prompt)
                .getResult()
                .getOutput()
                .getContent();

        // Add assistant response to history
        messages.add(new AssistantMessage(response));

        return response;
    }

    public void clearHistory(String sessionId) {
        conversationHistory.remove(sessionId);
    }
}
```

#### Controller for Conversation

```java
package com.example.springai.controller;

import com.example.springai.service.ConversationService;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/conversation")
public class ConversationController {

    private final ConversationService conversationService;

    public ConversationController(ConversationService conversationService) {
        this.conversationService = conversationService;
    }

    @PostMapping("/{sessionId}")
    public String chat(@PathVariable String sessionId, 
                      @RequestBody String message) {
        return conversationService.chat(sessionId, message);
    }

    @DeleteMapping("/{sessionId}")
    public void clearHistory(@PathVariable String sessionId) {
        conversationService.clearHistory(sessionId);
    }
}
```

### Example 5: Function Calling

```java
package com.example.springai.service;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.ChatResponse;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.openai.OpenAiChatOptions;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.function.Function;

@Service
public class FunctionCallingService {

    private final ChatClient chatClient;

    public FunctionCallingService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    // Define a function that AI can call
    public record WeatherRequest(String city, String unit) {}
    public record WeatherResponse(String city, double temperature, String condition, String unit) {}

    public Function<WeatherRequest, WeatherResponse> weatherFunction() {
        return request -> {
            // Simulate weather API call
            return new WeatherResponse(
                    request.city(),
                    25.5,
                    "Sunny",
                    request.unit() != null ? request.unit() : "celsius"
            );
        };
    }

    public String getWeatherWithFunction(String userMessage) {
        OpenAiChatOptions options = OpenAiChatOptions.builder()
                .withFunction("getWeather", 
                        "Get weather information for a city", 
                        weatherFunction())
                .build();

        Prompt prompt = new Prompt(
                List.of(new UserMessage(userMessage)),
                options
        );

        ChatResponse response = chatClient.call(prompt);
        return response.getResult().getOutput().getContent();
    }
}
```

### Example 6: Embeddings and Vector Search

```java
package com.example.springai.service;

import org.springframework.ai.document.Document;
import org.springframework.ai.embedding.EmbeddingClient;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

@Service
public class EmbeddingService {

    private final EmbeddingClient embeddingClient;
    private final VectorStore vectorStore;

    public EmbeddingService(EmbeddingClient embeddingClient, 
                           VectorStore vectorStore) {
        this.embeddingClient = embeddingClient;
        this.vectorStore = vectorStore;
    }

    public void addDocuments(List<String> texts) {
        List<Document> documents = texts.stream()
                .map(text -> new Document(text, Map.of("source", "user-input")))
                .toList();

        vectorStore.add(documents);
    }

    public List<Document> searchSimilar(String query, int topK) {
        return vectorStore.similaritySearch(
                SearchRequest.query(query).withTopK(topK)
        );
    }

    public float[] getEmbedding(String text) {
        return embeddingClient.embed(text);
    }
}
```

#### Controller for Vector Search

```java
package com.example.springai.controller;

import com.example.springai.service.EmbeddingService;
import org.springframework.ai.document.Document;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/embeddings")
public class EmbeddingController {

    private final EmbeddingService embeddingService;

    public EmbeddingController(EmbeddingService embeddingService) {
        this.embeddingService = embeddingService;
    }

    @PostMapping("/add")
    public String addDocuments(@RequestBody List<String> texts) {
        embeddingService.addDocuments(texts);
        return "Documents added successfully";
    }

    @GetMapping("/search")
    public List<Document> search(@RequestParam String query,
                                 @RequestParam(defaultValue = "5") int topK) {
        return embeddingService.searchSimilar(query, topK);
    }
}
```

### Example 7: RAG (Retrieval Augmented Generation)

```java
package com.example.springai.service;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.SystemPromptTemplate;
import org.springframework.ai.document.Document;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Service
public class RAGService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public RAGService(ChatClient chatClient, VectorStore vectorStore) {
        this.chatClient = chatClient;
        this.vectorStore = vectorStore;
    }

    public String askWithContext(String question) {
        // 1. Retrieve relevant documents
        List<Document> relevantDocs = vectorStore.similaritySearch(
                SearchRequest.query(question).withTopK(3)
        );

        // 2. Build context from documents
        String context = relevantDocs.stream()
                .map(Document::getContent)
                .collect(Collectors.joining("\n\n"));

        // 3. Create prompt with context
        String systemPromptTemplate = """
                You are a helpful assistant. Answer the user's question based on the following context.
                If the answer cannot be found in the context, say so.
                
                Context:
                {context}
                """;

        SystemPromptTemplate systemTemplate = new SystemPromptTemplate(systemPromptTemplate);
        Message systemMessage = systemTemplate.createMessage(Map.of("context", context));
        Message userMessage = new UserMessage(question);

        // 4. Generate response
        Prompt prompt = new Prompt(List.of(systemMessage, userMessage));
        return chatClient.call(prompt).getResult().getOutput().getContent();
    }
}
```

### Example 8: Image Generation

```java
package com.example.springai.service;

import org.springframework.ai.image.ImageClient;
import org.springframework.ai.image.ImagePrompt;
import org.springframework.ai.image.ImageResponse;
import org.springframework.ai.openai.OpenAiImageOptions;
import org.springframework.stereotype.Service;

@Service
public class ImageGenerationService {

    private final ImageClient imageClient;

    public ImageGenerationService(ImageClient imageClient) {
        this.imageClient = imageClient;
    }

    public ImageResponse generateImage(String description) {
        ImagePrompt prompt = new ImagePrompt(
                description,
                OpenAiImageOptions.builder()
                        .withModel("dall-e-3")
                        .withHeight(1024)
                        .withWidth(1024)
                        .build()
        );

        return imageClient.call(prompt);
    }

    public String generateImageUrl(String description) {
        ImageResponse response = generateImage(description);
        return response.getResult().getOutput().getUrl();
    }
}
```

### Example 9: Multi-Provider Configuration

```java
package com.example.springai.config;

import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.ollama.OllamaChatClient;
import org.springframework.ai.ollama.api.OllamaApi;
import org.springframework.ai.openai.OpenAiChatClient;
import org.springframework.ai.openai.api.OpenAiApi;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

@Configuration
public class MultiProviderConfig {

    @Value("${spring.ai.openai.api-key}")
    private String openAiApiKey;

    @Value("${spring.ai.ollama.base-url}")
    private String ollamaBaseUrl;

    @Bean
    @Primary
    public ChatClient openAiChatClient() {
        return new OpenAiChatClient(new OpenAiApi(openAiApiKey));
    }

    @Bean
    public ChatClient ollamaChatClient() {
        return new OllamaChatClient(new OllamaApi(ollamaBaseUrl));
    }
}
```

```java
package com.example.springai.service;

import org.springframework.ai.chat.ChatClient;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

@Service
public class MultiProviderService {

    private final ChatClient openAiClient;
    private final ChatClient ollamaClient;

    public MultiProviderService(
            @Qualifier("openAiChatClient") ChatClient openAiClient,
            @Qualifier("ollamaChatClient") ChatClient ollamaClient) {
        this.openAiClient = openAiClient;
        this.ollamaClient = ollamaClient;
    }

    public String chatWithOpenAI(String message) {
        return openAiClient.call(message);
    }

    public String chatWithOllama(String message) {
        return ollamaClient.call(message);
    }
}
```

### Example 10: Complete Application Class

```java
package com.example.springai;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringAiDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringAiDemoApplication.class, args);
    }
}
```

---

## Advanced Features

### 1. Custom Output Parsers

```java
package com.example.springai.parser;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.ai.chat.ChatResponse;
import org.springframework.ai.parser.OutputParser;

public class JsonOutputParser<T> implements OutputParser<T> {

    private final ObjectMapper objectMapper;
    private final Class<T> targetClass;

    public JsonOutputParser(Class<T> targetClass) {
        this.objectMapper = new ObjectMapper();
        this.targetClass = targetClass;
    }

    @Override
    public T parse(String text) {
        try {
            return objectMapper.readValue(text, targetClass);
        } catch (Exception e) {
            throw new RuntimeException("Failed to parse JSON", e);
        }
    }

    @Override
    public String getFormat() {
        return "JSON format matching " + targetClass.getSimpleName();
    }
}
```

### 2. Custom Retry Logic

```java
package com.example.springai.config;

import org.springframework.ai.retry.RetryUtils;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.retry.support.RetryTemplate;

@Configuration
public class RetryConfig {

    @Bean
    public RetryTemplate retryTemplate() {
        return RetryUtils.DEFAULT_RETRY_TEMPLATE;
    }
}
```

### 3. Request/Response Logging

```java
package com.example.springai.aspect;

import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

@Aspect
@Component
@Slf4j
public class ChatLoggingAspect {

    @Around("execution(* org.springframework.ai.chat.ChatClient.call(..))")
    public Object logChatCall(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("AI Request: {}", joinPoint.getArgs()[0]);
        
        long startTime = System.currentTimeMillis();
        Object result = joinPoint.proceed();
        long duration = System.currentTimeMillis() - startTime;
        
        log.info("AI Response received in {}ms", duration);
        return result;
    }
}
```

### 4. Token Usage Tracking

```java
package com.example.springai.service;

import lombok.Data;
import org.springframework.ai.chat.ChatResponse;
import org.springframework.stereotype.Service;

import java.util.concurrent.atomic.AtomicInteger;

@Service
public class TokenTrackingService {

    private final AtomicInteger totalTokens = new AtomicInteger(0);

    @Data
    public static class TokenUsage {
        private int promptTokens;
        private int completionTokens;
        private int totalTokens;
    }

    public void trackTokens(ChatResponse response) {
        if (response.getMetadata() != null) {
            Integer tokens = (Integer) response.getMetadata().get("total-tokens");
            if (tokens != null) {
                totalTokens.addAndGet(tokens);
            }
        }
    }

    public int getTotalTokensUsed() {
        return totalTokens.get();
    }

    public void resetTokens() {
        totalTokens.set(0);
    }
}
```

### 5. Rate Limiting

```java
package com.example.springai.config;

import io.github.bucket4j.Bandwidth;
import io.github.bucket4j.Bucket;
import io.github.bucket4j.Refill;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

@Configuration
public class RateLimitConfig {

    @Bean
    public Bucket rateLimitBucket() {
        // Allow 10 requests per minute
        Bandwidth limit = Bandwidth.classic(10, 
                Refill.intervally(10, Duration.ofMinutes(1)));
        return Bucket.builder()
                .addLimit(limit)
                .build();
    }
}
```

---

## Best Practices

### 1. Security
- **Never hardcode API keys** in source code
- Use environment variables or secure vaults (AWS Secrets Manager, Azure Key Vault)
- Implement rate limiting to prevent abuse
- Validate and sanitize user inputs
- Add authentication/authorization to AI endpoints

### 2. Performance
- Use streaming for long responses
- Implement caching for repeated queries
- Set appropriate timeout values
- Use async processing for non-blocking operations
- Monitor token usage and costs

### 3. Error Handling

```java
package com.example.springai.exception;

import org.springframework.ai.retry.NonTransientAiException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(NonTransientAiException.class)
    public ResponseEntity<String> handleAiException(NonTransientAiException ex) {
        return ResponseEntity
                .status(HttpStatus.SERVICE_UNAVAILABLE)
                .body("AI service error: " + ex.getMessage());
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<String> handleInvalidInput(IllegalArgumentException ex) {
        return ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body("Invalid input: " + ex.getMessage());
    }
}
```

### 4. Prompt Engineering Tips
- Be specific and clear in prompts
- Provide context and examples
- Use system messages to set behavior
- Break complex tasks into smaller steps
- Test and iterate on prompts

### 5. Testing

```java
package com.example.springai.service;

import org.junit.jupiter.api.Test;
import org.springframework.ai.chat.ChatClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
class ChatServiceTest {

    @Autowired
    private ChatClient chatClient;

    @Test
    void testSimpleChat() {
        String response = chatClient.call("What is 2+2?");
        assertThat(response).isNotEmpty();
        assertThat(response.toLowerCase()).contains("4");
    }
}
```

---

## Troubleshooting

### Common Issues

#### 1. API Key Not Found
```
Error: The API key is not set
```
**Solution**: Ensure environment variable is set correctly
```bash
export OPENAI_API_KEY=your-key-here
```

#### 2. Connection Timeout
```
Error: Connection timeout
```
**Solution**: Increase timeout in configuration
```properties
spring.ai.openai.chat.options.timeout=60s
```

#### 3. Rate Limit Exceeded
```
Error: Rate limit exceeded
```
**Solution**: Implement exponential backoff and retry logic

#### 4. Model Not Found
```
Error: Model not found
```
**Solution**: Verify model name in configuration
```properties
spring.ai.openai.chat.options.model=gpt-4
```

#### 5. Dependency Conflicts
**Solution**: Check Spring AI BOM version compatibility
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.0-M3</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Debug Mode

Enable detailed logging:
```properties
logging.level.org.springframework.ai=TRACE
logging.level.org.springframework.web=DEBUG
```

---

## Resources

### Official Documentation
- Spring AI Documentation: https://docs.spring.io/spring-ai/reference/
- Spring AI GitHub: https://github.com/spring-projects/spring-ai
- Spring Boot Documentation: https://spring.io/projects/spring-boot

### AI Provider Documentation
- OpenAI API: https://platform.openai.com/docs
- Azure OpenAI: https://learn.microsoft.com/en-us/azure/ai-services/openai/
- Ollama: https://ollama.ai/
- Hugging Face: https://huggingface.co/docs

### Community Resources
- Spring AI Examples: https://github.com/spring-projects/spring-ai/tree/main/spring-ai-docs/src/main/antora/modules/ROOT/examples
- Stack Overflow: Tag `spring-ai`
- Spring Community: https://spring.io/community

### Video Tutorials
- Spring AI Official Playlist (YouTube)
- DaShaun Carter - Spring AI Deep Dive
- Josh Long - Bootiful AI

### Books & Articles
- "Building AI Applications with Spring AI" (upcoming)
- Spring Blog - AI Category: https://spring.io/blog/category/ai

---

## Quick Start Checklist

- [ ] Java 17+ installed
- [ ] Spring Boot 3.2+ project created
- [ ] Spring AI dependencies added
- [ ] API key configured (OpenAI/Ollama/etc.)
- [ ] Basic ChatClient autowired
- [ ] First chat endpoint created
- [ ] Application tested with curl/Postman
- [ ] Error handling implemented
- [ ] Logging configured
- [ ] Ready for production features!

---

## Next Steps

1. **Experiment with different models** (GPT-4, Claude, Llama2)
2. **Implement RAG** with your own documents
3. **Add function calling** for external API integration
4. **Build a chatbot UI** (React, Vue, or Thymeleaf)
5. **Deploy to cloud** (AWS, Azure, or Google Cloud)
6. **Monitor and optimize** token usage and costs

---

## Contributing

If you find improvements for this guide, please contribute to the Spring AI community!

---

**Happy Coding with Spring AI! 🚀**
