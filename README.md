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
# =============================================================================
# IMPORTS - Libraries we need for our function
# =============================================================================
import azure.functions as func  # Azure Functions SDK - required for all Azure Functions
import logging                  # Built-in Python library for printing log messages
import json                     # Built-in Python library for working with JSON data
import re                       # Built-in Python library for Regular Expressions (pattern matching)
from datetime import datetime   # Built-in Python library for working with dates and times

# =============================================================================
# CREATE THE FUNCTION APP
# =============================================================================
# This creates our Function App with anonymous access (no authentication required)
# Think of this as the "container" that holds all our functions
app = func.FunctionApp(http_auth_level=func.AuthLevel.ANONYMOUS)

# =============================================================================
# DEFINE THE TEXT ANALYZER FUNCTION
# =============================================================================
# The @app.route decorator tells Azure: "When someone visits /api/TextAnalyzer, run this function"
# This is called a "decorator" - it adds extra behavior to our function
@app.route(route="TextAnalyzer")
def TextAnalyzer(req: func.HttpRequest) -> func.HttpResponse:
    """
    This function analyzes text and returns statistics about it.

    Parameters:
        req: The incoming HTTP request (contains the text to analyze)

    Returns:
        func.HttpResponse: JSON response with analysis results
    """

    # Log a message so we can see in Azure Portal when the function is called
    logging.info('Text Analyzer API was called!')

    # =========================================================================
    # STEP 1: GET THE TEXT INPUT
    # =========================================================================
    # First, try to get 'text' from the URL query string
    # Example URL: /api/TextAnalyzer?text=Hello world
    # req.params.get('text') would return "Hello world"
    text = req.params.get('text')

    # If text wasn't in the URL, try to get it from the request body (JSON)
    if not text:
        try:
            # Try to parse the request body as JSON
            # Example body: {"text": "Hello world"}
            req_body = req.get_json()
            text = req_body.get('text')
        except ValueError:
            # If the body isn't valid JSON, just continue (text stays None)
            pass

    # =========================================================================
    # STEP 2: ANALYZE THE TEXT (if text was provided)
    # =========================================================================
    if text:
        # ----- Word Analysis -----
        # split() breaks the text into a list of words
        # "Hello world" becomes ["Hello", "world"]
        words = text.split()

        # len() counts items in a list
        # ["Hello", "world"] has length 2
        word_count = len(words)

        # ----- Character Analysis -----
        # len() on a string counts characters (including spaces)
        # "Hello world" has 11 characters
        char_count = len(text)

        # replace(" ", "") removes all spaces, then we count
        # "Hello world" becomes "Helloworld" (10 characters)
        char_count_no_spaces = len(text.replace(" ", ""))

        # ----- Sentence Analysis -----
        # re.findall() finds all matches of a pattern
        # r'[.!?]+' means: find any sequence of periods, exclamation marks, or question marks
        # "Hello! How are you?" returns ['!', '?'] (2 sentences)
        # The "or 1" means: if no punctuation found, assume at least 1 sentence
        sentence_count = len(re.findall(r'[.!?]+', text)) or 1

        # ----- Paragraph Analysis -----
        # Paragraphs are separated by blank lines (two newlines: \n\n)
        # split('\n\n') breaks text at blank lines
        # We filter out empty paragraphs with "if p.strip()"
        # strip() removes whitespace - empty strings become "" which is False
        paragraph_count = len([p for p in text.split('\n\n') if p.strip()])

        # ----- Reading Time Calculation -----
        # Average reading speed is about 200 words per minute
        # round(x, 1) rounds to 1 decimal place
        # 100 words / 200 wpm = 0.5 minutes
        reading_time_minutes = round(word_count / 200, 1)

        # ----- Average Word Length -----
        # Total characters (no spaces) divided by number of words
        # We check "if word_count > 0" to avoid dividing by zero
        avg_word_length = round(char_count_no_spaces / word_count, 1) if word_count > 0 else 0

        # ----- Find Longest Word -----
        # max() finds the largest item
        # key=len means "compare words by their length"
        # max(["Hi", "Hello", "Hey"], key=len) returns "Hello"
        longest_word = max(words, key=len) if words else ""

        # =====================================================================
        # STEP 3: BUILD THE RESPONSE
        # =====================================================================
        # Create a Python dictionary with all our analysis results
        # This will be converted to JSON format
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
                # datetime.utcnow() gets current time, isoformat() converts to string
                # Example: "2024-01-15T14:30:00.000000"
                "analyzedAt": datetime.utcnow().isoformat(),

                # Show first 100 characters of text as a preview
                # The syntax: value_if_true if condition else value_if_false
                # This is called a "ternary operator" or "conditional expression"
                "textPreview": text[:100] + "..." if len(text) > 100 else text
            }
        }

        # Return a successful HTTP response
        # json.dumps() converts Python dictionary to JSON string
        # indent=2 makes the JSON nicely formatted (2 spaces per indent level)
        # mimetype tells the browser "this is JSON data"
        # status_code=200 means "OK - Success"
        return func.HttpResponse(
            json.dumps(response_data, indent=2),
            mimetype="application/json",
            status_code=200
        )

    # =========================================================================
    # STEP 4: HANDLE MISSING TEXT (Error Response)
    # =========================================================================
    else:
        # If no text was provided, return helpful instructions
        instructions = {
            "error": "No text provided",
            "howToUse": {
                "option1": "Add ?text=YourText to the URL",
                "option2": "Send a POST request with JSON body: {\"text\": \"Your text here\"}",
                "example": "https://your-function-url/api/TextAnalyzer?text=Hello world"
            }
        }

        # Return an error response
        # status_code=400 means "Bad Request - client made an error"
        return func.HttpResponse(
            json.dumps(instructions, indent=2),
            mimetype="application/json",
            status_code=400
        )
```

### 3.4 Code Explanation

Here's a breakdown of how the code works, section by section:

#### Imports Section

| Import            | What It Does                                                                                |
| ----------------- | ------------------------------------------------------------------------------------------- |
| `azure.functions` | The Azure Functions SDK - provides `HttpRequest`, `HttpResponse`, and `FunctionApp` classes |
| `logging`         | Lets us write messages to Azure's log system for debugging                                  |
| `json`            | Converts Python dictionaries to JSON strings (and vice versa)                               |
| `re`              | Regular expressions for pattern matching (used to count sentences)                          |
| `datetime`        | Gets the current date and time                                                              |

#### Function Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    HTTP Request Arrives                         │
│            (e.g., /api/TextAnalyzer?text=Hello)                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              Step 1: Try to Get Text Input                      │
│  • First check URL query string: ?text=...                      │
│  • If not found, check JSON body: {"text": "..."}               │
└─────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              │                               │
        Text Found?                     No Text Found
              │                               │
              ▼                               ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│   Step 2: Analyze       │     │   Step 4: Return        │
│   • Count words         │     │   Error Response        │
│   • Count characters    │     │   (400 Bad Request)     │
│   • Count sentences     │     │   with instructions     │
│   • Calculate metrics   │     └─────────────────────────┘
└─────────────────────────┘
              │
              ▼
┌─────────────────────────┐
│   Step 3: Build and     │
│   Return JSON Response  │
│   (200 OK)              │
└─────────────────────────┘
```

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

### 4.4 Test with the .http File

1. Open the `test-function.http` file in VS Code
2. Install the **REST Client** extension if not already installed
3. Update the `@baseUrl` variable with your function URL
4. Click **Send Request** above any test case to execute it

### 4.5 Test with the Web Client

First, enable CORS on your Function App:

1. In the Azure Portal, go to your Function App
2. In the left menu under **API**, click **CORS**
3. Add `*` to the Allowed Origins list
4. Click **Save**

Then use the client:

1. Open the `client.html` file in your browser
2. Enter your Function URL (from Step 4.2)
3. Type or paste text to analyze
4. Click **Analyze Text**

The client displays:
- Visual flow diagram showing the request lifecycle
- Analysis results in a dashboard format
- Raw JSON response from the function

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
