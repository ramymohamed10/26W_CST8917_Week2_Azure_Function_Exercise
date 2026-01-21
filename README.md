# Week 2: Your First Azure Function

**CST8917 - Serverless Applications**

---

## Overview

This lab introduces you to serverless computing by creating an HTTP-triggered Azure Function. You will build a **Text Analyzer API** that accepts text input and returns useful analytics including word count, character count, reading time, and more.

**Learning Objectives:**
- Create and configure an Azure Function App
- Implement an HTTP-triggered function using Python
- Test and monitor serverless functions

**Prerequisites:**
- Active Azure subscription (Azure for Students or Free Tier)
- Web browser

---

## Part 1: Create a Function App

A Function App is the container that hosts your serverless functions.

### 1.1 Access Azure Portal

1. Navigate to [https://portal.azure.com](https://portal.azure.com)
2. Sign in with your Azure account

### 1.2 Create the Function App

1. Click **+ Create a resource**
2. Search for **Function App** and select it
3. Click **Create**

### 1.3 Configure Basic Settings

| Setting           | Value                                            |
| ----------------- | ------------------------------------------------ |
| Subscription      | Your subscription                                |
| Resource Group    | Create new: `rg-serverless-lab`                  |
| Function App name | `func-textanalyzer-[yourname]` (globally unique) |
| Runtime stack     | Python                                           |
| Version           | 3.11 (latest)                                    |
| Region            | Canada Central                                   |
| Operating System  | Linux                                            |
| Hosting plan      | Consumption (Serverless)                         |

### 1.4 Configure Storage

1. Click **Next: Storage**
2. Create a new storage account: `stserverlesslab[yourname]` (lowercase, no special characters)

### 1.5 Deploy

1. Click **Review + create**
2. Click **Create**
3. Wait for deployment to complete

---

## Part 2: Create an HTTP Function

### 2.1 Navigate to Your Function App

1. Click **Go to resource**
2. In the left menu, click **Functions**

### 2.2 Create the Function

1. Click **+ Create**
2. Select **HTTP trigger**
3. Configure:
   - Name: `TextAnalyzer`
   - Authorization level: **Anonymous**
4. Click **Create**

> **Authorization Levels:**
> - **Anonymous**: No authentication required
> - **Function**: Requires function-specific key
> - **Admin**: Requires master key

---

## Part 3: Implement the Function

### 3.1 Open the Code Editor

1. Click on `TextAnalyzer`
2. Click **Code + Test**

### 3.2 Default Code Structure

The template provides this structure:

```python
import azure.functions as func
import logging

app = func.FunctionApp(http_auth_level=func.AuthLevel.ANONYMOUS)

@app.route(route="TextAnalyzer")
def TextAnalyzer(req: func.HttpRequest) -> func.HttpResponse:
    logging.info('Python HTTP trigger function processed a request.')

    name = req.params.get('name')
    if not name:
        try:
            req_body = req.get_json()
        except ValueError:
            pass
        else:
            name = req_body.get('name')

    if name:
        return func.HttpResponse(f"Hello, {name}.")
    else:
        return func.HttpResponse("Pass a name in the query string or request body.", status_code=200)
```

**Key Components:**
| Component               | Purpose                                          |
| ----------------------- | ------------------------------------------------ |
| `@app.route()`          | Defines the URL route that triggers the function |
| `req: func.HttpRequest` | Incoming HTTP request object                     |
| `func.HttpResponse`     | Response returned to the caller                  |
| `req.params.get()`      | Retrieves query string parameters                |
| `req.get_json()`        | Retrieves JSON from request body                 |

### 3.3 Replace with Custom Code

Replace the default code with:

```python
import azure.functions as func
import logging
import json
import re
from datetime import datetime

app = func.FunctionApp(http_auth_level=func.AuthLevel.ANONYMOUS)

@app.route(route="TextAnalyzer")
def TextAnalyzer(req: func.HttpRequest) -> func.HttpResponse:
    logging.info('Text Analyzer API was called!')

    # Get text from query parameter or request body
    text = req.params.get('text')

    if not text:
        try:
            req_body = req.get_json()
            text = req_body.get('text')
        except ValueError:
            pass

    if text:
        # Perform text analysis
        words = text.split()
        word_count = len(words)
        char_count = len(text)
        char_count_no_spaces = len(text.replace(" ", ""))
        sentence_count = len(re.findall(r'[.!?]+', text)) or 1
        paragraph_count = len([p for p in text.split('\n\n') if p.strip()])

        # Calculate reading time (average 200 words per minute)
        reading_time_minutes = round(word_count / 200, 1)

        # Calculate average word length
        avg_word_length = round(char_count_no_spaces / word_count, 1) if word_count > 0 else 0

        # Find longest word
        longest_word = max(words, key=len) if words else ""

        # Create response
        response_data = {
            "analysis": {
                "wordCount": word_count,
                "characterCount": char_count,
                "characterCountNoSpaces": char_count_no_spaces,
                "sentenceCount": sentence_count,
                "paragraphCount": paragraph_count,
                "averageWordLength": avg_word_length,
                "longestWord": longest_word,
                "readingTimeMinutes": reading_time_minutes
            },
            "metadata": {
                "analyzedAt": datetime.utcnow().isoformat(),
                "textPreview": text[:100] + "..." if len(text) > 100 else text
            }
        }

        return func.HttpResponse(
            json.dumps(response_data, indent=2),
            mimetype="application/json",
            status_code=200
        )
    else:
        # Return helpful instructions if no text provided
        instructions = {
            "error": "No text provided",
            "howToUse": {
                "option1": "Add ?text=YourText to the URL",
                "option2": "Send a POST request with JSON body: {\"text\": \"Your text here\"}",
                "example": "https://your-function-url/api/TextAnalyzer?text=Hello world"
            }
        }
        return func.HttpResponse(
            json.dumps(instructions, indent=2),
            mimetype="application/json",
            status_code=400
        )
```

### 3.4 Save

Click **Save** at the top of the editor.

---

## Part 4: Test Your Function

### 4.1 Test in Portal

1. Click **Test/Run**
2. Set HTTP method to **GET**
3. Add query parameter: `text` = `Serverless computing is a cloud execution model. The cloud provider manages the infrastructure.`
4. Click **Run**

Expected response:
```json
{
  "analysis": {
    "wordCount": 14,
    "characterCount": 95,
    "characterCountNoSpaces": 82,
    "sentenceCount": 2,
    "paragraphCount": 1,
    "averageWordLength": 5.9,
    "longestWord": "infrastructure.",
    "readingTimeMinutes": 0.1
  },
  "metadata": {
    "analyzedAt": "2026-01-21T14:30:00.000000",
    "textPreview": "Serverless computing is a cloud execution model. The cloud provider manages the infrastructure."
  }
}
```

### 4.2 Get Function URL

1. Click **Get function URL**
2. Copy the URL

### 4.3 Test in Browser

Open the URL in a browser with parameters:
```
https://func-textanalyzer-yourname.azurewebsites.net/api/TextAnalyzer?text=Azure Functions lets you run code without managing servers
```

### 4.4 Test with the Provided .http File

1. Open the `test-function.http` file in VS Code
2. Install the **REST Client** extension if not already installed
3. Replace `YOUR_FUNCTION_APP_NAME` in the `@baseUrl` variable with your function app name
4. Click **Send Request** above any test case to execute it

The file includes test cases for:
- Simple GET requests with query parameters
- POST requests with JSON body
- Multi-paragraph text analysis
- Error handling (no text provided)

---

## Part 5: Monitor Your Function

### 5.1 View Function App Metrics

1. Navigate to your Function App overview page
2. Click the **Metrics** tab
3. Review the following charts:
   - **Memory working set**: Memory consumption over time
   - **Function Execution Count**: Number of times functions were invoked
   - **MB Milliseconds**: Resource consumption (used for billing calculations)

### 5.2 View Function-Level Metrics

1. In the left menu, expand **Functions**
2. Click on `TextAnalyzer`
3. Click the **Metrics** tab
4. Review the following charts:
   - **Total Execution Count**: All function invocations
   - **Successful Execution Count**: Invocations that completed without errors
   - **Failed Execution Count**: Invocations that resulted in errors
   - **Http 2xx**: Successful HTTP responses
   - **Http 4xx**: Client error responses (e.g., 400 Bad Request)
   - **Http 5xx**: Server error responses

### 5.3 View Invocation Details

1. From your function, click **Invocations** tab
2. Click on any invocation to see detailed logs and timing information

---

## Part 6: Clean Up Resources

To avoid charges:

1. Go to **Resource groups**
2. Select `rg-serverless-lab`
3. Click **Delete resource group**
4. Confirm deletion

---

## Summary

You have completed the following:
- Created an Azure Function App with Consumption plan
- Implemented an HTTP-triggered Text Analyzer API in Python
- Tested the function via portal and browser
- Monitored function executions

**Key Concepts:**
- **Consumption Plan**: Pay only when functions execute; automatic scaling
- **HTTP Trigger**: Function invoked via HTTP requests
- **Cold Start**: Initial delay when function hasn't been called recently

---

## Resources

- [Azure Functions Documentation](https://learn.microsoft.com/en-us/azure/azure-functions/)
- [Azure Functions Python Guide](https://learn.microsoft.com/en-us/azure/azure-functions/functions-reference-python)

---

**This is an ungraded activity.**
