

# AWS Lambda with Google Gemini API

This repository provides a complete guide to setting up an AWS Lambda function that securely connects to the Google Gemini API. The setup includes secure API key storage using AWS Secrets Manager, a pre-packaged Lambda Layer for dependencies, a public-facing Function URL for easy invocation, and an optional EventBridge schedule for periodic triggers.

## Features

  - **Secure API Key Management**: Store your Google Gemini API key safely in AWS Secrets Manager.
  - **Serverless Architecture**: Run your code without provisioning or managing servers using AWS Lambda.
  - **Dependency Management**: Use a custom AWS Lambda Layer to package the `google-genai` SDK, ensuring a clean and efficient deployment.
  - **Multiple Invocation Methods**:
      - Trigger the function via a public **Function URL** (GET/POST).
      - Test directly from the **AWS Console**.
      - Invoke on a schedule using **Amazon EventBridge**.
  - **Efficient Code**: The Python handler includes caching for the API key and the Gemini client to optimize performance on warm starts.

-----

## Prerequisites

  * **Google AI Studio API Key**: You'll need an API key from [Google AI Studio](https://aistudio.google.com/app/apikey).
  * **AWS Account**: An active AWS account. If you don't have one, you can sign up at [aws.amazon.com](https://aws.amazon.com/).

-----

## Setup Instructions

Follow these steps to deploy the function. It is recommended to perform all steps in the same AWS region (e.g., `ap-south-1` Asia Pacific (Mumbai)).

### 1\. Store Your Gemini API Key in AWS Secrets Manager

First, we'll securely store the API key.

1.  Navigate to **AWS Secrets Manager** in the AWS Console.
2.  Click **Store a new secret**.
3.  For **Secret type**, select `Other type of secret`.
4.  Under **Secret value**, choose the `Plaintext` tab and paste your Gemini API key directly.
5.  Click **Next**.
6.  Set the **Secret name** to exactly `prod/gemini/api_key`.
7.  Click **Next** twice, then click **Store**.
8.  Open the secret you just created and **copy its ARN**. You will need this for the IAM policy later.

### 2\. Create the Lambda Function

Next, create the serverless function that will run the code.

1.  Navigate to **AWS Lambda** and click **Create function**.
2.  Configure the function with the following settings:
      * **Option**: `Author from scratch`
      * **Function name**: `gemini-ping`
      * **Runtime**: `Python 3.12`
      * **Architecture**: `x86_64`
      * **Permissions**: `Create a new role with basic Lambda permissions`
3.  Click **Create function**.
4.  Once created, go to the **Configuration** tab -\> **General configuration** -\> **Edit**.
5.  Set the following values and **Save**:
      * **Memory**: `256` MB
      * **Timeout**: `20` seconds

### 3\. Grant Lambda Permission to Read the Secret

Now, allow the Lambda function's execution role to access the secret you created.

1.  In the Lambda function's page, go to **Configuration** -\> **Permissions**.

2.  Click the **Role name** link to open the IAM console.

3.  In the IAM role view, click **Add permissions** -\> **Create inline policy**.

4.  Select the **JSON** tab and paste the following policy. **Remember to replace `YOUR_ACCOUNT_ID` and `<YOUR-REGION>` with your actual AWS Account ID and region.**

    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Sid": "ReadGeminiSecret",
          "Effect": "Allow",
          "Action": [
            "secretsmanager:GetSecretValue"
          ],
          "Resource": "arn:aws:secretsmanager:<YOUR-REGION>:<YOUR_ACCOUNT_ID>:secret:prod/gemini/api_key-*"
        }
      ]
    }
    ```

5.  Click **Review policy**, name it `AllowReadGeminiSecret`, and click **Create policy**.

### 4\. Create and Attach the Lambda Layer

The `google-genai` library needs to be available to our function. We'll build it as a Lambda Layer using AWS CloudShell to ensure it's compiled for the correct environment.

1.  Open **AWS CloudShell** from the top navigation bar in the AWS Console.

2.  Run the following commands in the terminal to build and publish the layer. **Replace `<YOUR-REGION>` with your selected region.**

    ```bash
    # Create directories and upgrade pip
    python3 -m pip install --upgrade pip
    mkdir -p genai-layer/python wheels

    # Download Python 3.12 compatible wheels for Amazon Linux
    python3 -m pip download \
      --only-binary=:all: \
      --platform manylinux2014_x86_64 \
      --implementation cp \
      --python-version 312 \
      -d wheels \
      "google-genai==0.6.0" "pydantic==2.8.2" "pydantic-core==2.20.1"

    # Unzip wheels into the layer folder structure
    for f in wheels/*.whl; do
      unzip -o "$f" -d genai-layer/python > /dev/null
    done

    # Zip the layer content and publish it
    cd genai-layer
    zip -r genai-layer.zip python

    aws lambda publish-layer-version \
      --layer-name google-genai-py312 \
      --region <YOUR-REGION> \
      --description "Google GenAI SDK layer (cp312, manylinux2014_x86_64)" \
      --zip-file fileb://genai-layer.zip \
      --compatible-runtimes python3.12
    ```

3.  From the JSON output of the last command, **copy the `LayerVersionArn`**.

4.  Navigate back to your `gemini-ping` Lambda function.

5.  Scroll down to the **Layers** section and click **Add a layer**.

6.  Select **Specify an ARN**, paste the copied `LayerVersionArn`, and click **Add**.

### 5\. Configure Environment Variables

Set environment variables to make the function configurable without changing code.

1.  In your Lambda function's page, go to **Configuration** -\> **Environment variables** -\> **Edit**.
2.  Add the following variables:
      * `GEMINI_SECRET_NAME`: `prod/gemini/api_key`
      * `MODEL_NAME`: `gemini-1.5-flash-latest`
3.  Click **Save**.

### 6\. Add the Lambda Function Code

Finally, add the Python code to the function.

1.  Go to the **Code** tab of your Lambda function.
2.  Replace the contents of `lambda_function.py` with the code below.
3.  Click the **Deploy** button to save your changes.

<!-- end list -->

```python
import json
import os
import base64
import boto3
import logging

# Provided by the Lambda Layer
import google.generativeai as genai

# Logger setup
logger = logging.getLogger()
logger.setLevel(logging.INFO)

# Environment variables
SECRET_NAME = os.environ.get("GEMINI_SECRET_NAME", "prod/gemini/api_key")
MODEL_NAME = os.environ.get("MODEL_NAME", "gemini-1.5-flash-latest")
REGION = os.environ.get("AWS_REGION") or os.environ.get("AWS_DEFAULT_REGION")

# Caching for warm starts
_API_KEY_CACHE = None
_GENERATIVE_MODEL_CACHE = None

def _get_api_key():
    """Fetch and cache the API key from Secrets Manager."""
    global _API_KEY_CACHE
    if _API_KEY_CACHE:
        return _API_KEY_CACHE

    try:
        sm = boto3.client("secretsmanager", region_name=REGION)
        sec = sm.get_secret_value(SecretId=SECRET_NAME)
        s = sec["SecretString"]
        # Support either a bare string or a simple JSON structure like {"api_key": "..."}
        try:
            data = json.loads(s)
            api_key = next(iter(data.values()))
        except (json.JSONDecodeError, StopIteration):
            api_key = s

        _API_KEY_CACHE = api_key.strip()
        return _API_KEY_CACHE
    except Exception as e:
        logger.error(f"Failed to retrieve API key from Secrets Manager: {e}")
        raise

def _get_model():
    """Initialize and cache the GenerativeModel client."""
    global _GENERATIVE_MODEL_CACHE
    if _GENERATIVE_MODEL_CACHE:
        return _GENERATIVE_MODEL_CACHE
    
    api_key = _get_api_key()
    genai.configure(api_key=api_key)
    _GENERATIVE_MODEL_CACHE = genai.GenerativeModel(MODEL_NAME)
    return _GENERATIVE_MODEL_CACHE

def _extract_prompt_from_event(event):
    """Extracts the prompt from various Lambda invocation sources."""
    # Direct invoke or EventBridge: {"prompt":"..."}
    if isinstance(event, dict):
        p = event.get("prompt")
        if p and isinstance(p, str):
            return p.strip()

        # API Gateway / Function URL GET: ?prompt=...
        q = event.get("queryStringParameters")
        if q and isinstance(q, dict):
            p = q.get("prompt")
            if p and isinstance(p, str):
                return p.strip()

        # API Gateway / Function URL POST (JSON body): {"prompt":"..."}
        body = event.get("body")
        if body:
            try:
                if event.get("isBase64Encoded"):
                    body = base64.b64decode(body).decode("utf-8")
                data = json.loads(body)
                p = data.get("prompt")
                if p and isinstance(p, str):
                    return p.strip()
            except (json.JSONDecodeError, UnicodeDecodeError):
                pass # Ignore malformed body
                
    return "Explain why many teams moved from TensorFlow to PyTorch in one paragraph."

def handler(event, context):
    try:
        prompt = _extract_prompt_from_event(event)
        model = _get_model()

        response = model.generate_content(prompt)
        text = response.text

        # Log a compact record for monitoring in CloudWatch
        log_output = {
            "aws_request_id": getattr(context, "aws_request_id", ""),
            "model": MODEL_NAME,
            "prompt": prompt,
            "response_preview": text[:250] + "..." if len(text) > 250 else text,
            "response_len": len(text),
        }
        logger.info(json.dumps(log_output, ensure_ascii=False))

        return {
            "statusCode": 200,
            "headers": {
                "Content-Type": "application/json",
                "Access-Control-Allow-Origin": "*", # CORS for public access
            },
            "body": json.dumps({"model": MODEL_NAME, "prompt": prompt, "response": text}, ensure_ascii=False),
        }

    except Exception as e:
        logger.exception("Lambda handler error")
        return {
            "statusCode": 500,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({"error": str(e)}),
        }

```

-----

## Deployment and Testing

Your function is now deployed. You can test it in three ways.

### 1\. Create and Test with a Function URL

1.  In your Lambda function's page, go to **Configuration** -\> **Function URL** -\> **Create function URL**.
2.  Set **Auth type** to `NONE`.
3.  Click **Save**. A public URL will be generated.
4.  **Copy the Function URL**.

#### Test with a GET request (in your browser):

Append a `prompt` query parameter to the URL:

```
<YOUR_FUNCTION_URL>?prompt=Tell%20me%20a%20short%20joke%20about%20AI
```

#### Test with a POST request (using cURL):

```bash
curl -X POST "<YOUR_FUNCTION_URL>" \
-H "Content-Type: application/json" \
-d '{"prompt": "Explain the attention mechanism in one paragraph."}'
```

### 2\. Test via the AWS Console

1.  In the **Code** tab of your Lambda function, click the **Test** button.

2.  Select **Configure test event**.

3.  Choose **Create a new event** and give it a name like `TestPrompt`.

4.  Use the following JSON as the event payload:

    ```json
    {
      "prompt": "Why is the sky blue?"
    }
    ```

5.  Click **Save**, then click the **Test** button again. You should see a successful execution result with the Gemini API's response.

-----

## (Optional) Schedule Periodic Invocations with EventBridge

You can set up a cron job to run your function automatically.

1.  Navigate to **Amazon EventBridge** in the AWS Console.
2.  Go to **Schedules** and click **Create schedule**.
3.  **Schedule name**: `gemini-ping-3m`
4.  **Schedule pattern**:
      * Select `Recurring schedule`.
      * Choose `Rate-based schedule`.
      * Set it to run every `3` `Minutes`.
5.  Click **Next**.
6.  **Select a target**:
      * Search for and select **AWS Lambda**.
      * **Lambda function**: Choose your `gemini-ping` function from the dropdown.
      * **Payload (optional)**: To send a custom prompt, use the `Constant (JSON text)` option and enter a payload like:
        ```json
        { "prompt": "Quick 1-line status: attention vs RNNs." }
        ```
7.  Click **Next**, review the details, and click **Create schedule**.

The schedule will now invoke your Lambda function every 3 minutes. You can monitor its execution in the **CloudWatch Logs** for your Lambda function.

-----

## Troubleshooting

  - **`No module named 'google'`**: The Lambda Layer is either not attached or was built incorrectly. Ensure it is attached under the "Layers" section. If the problem persists, delete the layer and recreate it following Step 4.
  - **`AccessDeniedException` for Secrets Manager**: The IAM policy is incorrect. Double-check that the `Resource` ARN in the IAM policy (Step 3) exactly matches the ARN of your secret.
  - **Task timed out**: The function took too long to execute. Increase the **Timeout** in the Lambda function's general configuration (Step 2) to 20-30 seconds.
  - **`403 Forbidden` on Function URL**: Ensure the **Auth type** for the Function URL is set to `NONE` for public access.
  - **No Internet Access**: If the function times out with no clear error, ensure it is **not** attached to a VPC, or if it is, that the VPC has a NAT Gateway configured for internet access. The default setting (no VPC) is correct.
