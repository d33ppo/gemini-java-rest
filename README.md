### README.md

# AI Chatbot in Java using REST API

This project demonstrates how to create an AI chatbot in Java using the Gemini REST API. It includes a simple Spring Boot application that integrates with the Gemini model for generating responses to user prompts.

## Prerequisites

- Java 17
- Maven
- Google Cloud account to generate API key

## Project Structure

- **pom.xml**: Maven configuration file.
- **src/main/java/com/webosmotic/gemini/controller/Controller.java**: REST controller to handle chat requests.
- **src/main/java/com/webosmotic/gemini/service/AIService.java**: Service class for interacting with the Gemini API.

## Getting Started

### Step 1: Clone the Repository

```bash
git clone https://github.com/yourusername/ai-chatbot-java.git
cd ai-chatbot-java
```

### Step 2: Update API Key

Replace the `API_KEY` variable in `AIService.java` with your own Google Cloud API key.

### Step 3: Build the Project

```bash
mvn clean install
```

### Step 4: Run the Application

```bash
mvn spring-boot:run
```

### Step 5: Test the Chatbot

You can use Postman or any other HTTP client to test the chatbot. Send a POST request to `http://localhost:8080/api/chat` with a JSON body containing the prompt:

```json
{
    "prompt": "Hello, I am a user."
}
```

## Project Details

### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.0</version>
        <relativePath/>
    </parent>

    <groupId>com.webosmotic</groupId>
    <artifactId>gemini</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>gemini</name>
    <description>Gemini Demo project for Spring Boot</description>
    <url>https://example.com</url>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.googlecode.json-simple</groupId>
            <artifactId>json-simple</artifactId>
            <version>1.1.1</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-log4j2</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>2.7.0</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>
```

### Controller.java

```java
package com.webosmotic.gemini.controller;

import com.webosmotic.gemini.service.AIService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api")
public class Controller {

    @Autowired
    AIService aiService;

    @PostMapping("/chat")
    public String chat(@RequestBody String prompt) {
        return this.aiService.chat(prompt);
    }

}
```

### AIService.java

```java
package com.webosmotic.gemini.service;

import org.apache.logging.log4j.util.Strings;
import org.json.simple.JSONArray;
import org.json.simple.JSONObject;
import org.json.simple.parser.JSONParser;
import org.json.simple.parser.ParseException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.*;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.io.IOException;

@Service
public class AIService {

    private static final Logger logger = LoggerFactory.getLogger(AIService.class);

    private static final String GEMINI_MODEL = "gemini-1.5-flash";
    private static final String API_KEY = "YOUR_API_KEY_HERE";
    private String conversationHistory = "";

    public String chat(String prompt) {
        String fullPrompt = prompt;

        if (!Strings.isBlank(conversationHistory)) {
            fullPrompt = "[Context]" + conversationHistory + " [Content] " + prompt;
        }

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);

        fullPrompt = getPromptBody(fullPrompt);
        HttpEntity<String> requestEntity = new HttpEntity<>(fullPrompt, headers);

        RestTemplate restTemplate = new RestTemplate();
        ResponseEntity<String> responseEntity = restTemplate.exchange(
                "https://generativelanguage.googleapis.com/v1beta/models/" + GEMINI_MODEL + ":generateContent?key=" + API_KEY,
                HttpMethod.POST,
                requestEntity,
                String.class
        );

        HttpStatus statusCode = responseEntity.getStatusCode();

        if (statusCode == HttpStatus.OK) {
            String responseText = responseEntity.getBody();
            try {
                responseText = parseGeminiResponse(responseText);
                conversationHistory += prompt + "\n" + responseText + "\n";
            } catch (Exception e) {
                logger.error("Error parsing response", e);
            }
            return responseText;
        } else {
            throw new RuntimeException("API request failed with status code: " + statusCode + " and response: " + responseEntity.getBody());
        }
    }

    public String getPromptBody(String prompt) {
        JSONObject promptJson = new JSONObject();

        JSONArray contentsArray = new JSONArray();
        JSONObject contentsObject = new JSONObject();
        contentsObject.put("role", "user");

        JSONArray partsArray = new JSONArray();
        JSONObject partsObject = new JSONObject();
        partsObject.put("text", prompt);
        partsArray.add(partsObject);
        contentsObject.put("parts", partsArray);

        contentsArray.add(contentsObject);
        promptJson.put("contents", contentsArray);

        JSONArray safetySettingsArray = new JSONArray();

        JSONObject hateSpeechSetting = new JSONObject();
        hateSpeechSetting.put("category", "HARM_CATEGORY_HATE_SPEECH");
        hateSpeechSetting.put("threshold", "BLOCK_ONLY_HIGH");
        safetySettingsArray.add(hateSpeechSetting);

        JSONObject dangerousContentSetting = new JSONObject();
        dangerousContentSetting.put("category", "HARM_CATEGORY_DANGEROUS_CONTENT");
        dangerousContentSetting.put("threshold", "BLOCK_ONLY_HIGH");
        safetySettingsArray.add(dangerousContentSetting);

        JSONObject sexuallyExplicitSetting = new JSONObject();
        sexuallyExplicitSetting.put("category", "HARM_CATEGORY_SEXUALLY_EXPLICIT");
        sexuallyExplicitSetting.put("threshold", "BLOCK_ONLY_HIGH");
        safetySettingsArray.add(sexuallyExplicitSetting);

        JSONObject harassmentSetting = new JSONObject();
        harassmentSetting.put("category", "HARM_CATEGORY_HARASSMENT");
        harassmentSetting.put("threshold", "BLOCK_ONLY_HIGH");
        safetySettingsArray.add(harassmentSetting);

        promptJson.put("safetySettings", safetySettingsArray);

        JSONObject parametersJson = new JSONObject();
        parametersJson.put("temperature", 0.5);
        parametersJson.put("topP", 0.99);
        promptJson.put("generationConfig", parametersJson);

        return promptJson.toJSONString();
    }

    public String parseGeminiResponse(String jsonResponse) throws IOException, ParseException {
        JSONObject jsonObject = (JSONObject) new JSONParser().parse(jsonResponse);

        JSONArray candidatesArray = (JSONArray) jsonObject.get("candidates");

        JSONObject candidateObject = (JSONObject) candidatesArray.get(0);
        JSONObject contentObject = (JSONObject) candidateObject.get("content");

        JSONArray partsArray = (JSONArray) contentObject.get("parts");

        JSONObject partObject = (JSONObject) partsArray.get(0);
        return (String) partObject.get("text");
    }

}
```

## License

This project is licensed under the MIT License. See the LICENSE file for details.
