# Vocareum OpenAI API Setup Guide

This guide provides detailed instructions for configuring your code to use Vocareum OpenAI API keys in the Udacity classroom environment.

## What Are Vocareum OpenAI API Keys?

Vocareum OpenAI API keys provide access to paid OpenAI services through Vocareum servers. Unlike standard OpenAI API keys, these keys:
- Start with the prefix `voc-`
- Must route through Vocareum servers at `https://openai.vocareum.com/v1`
- Have managed budgets set by Udacity
- Cannot be reissued if lost

## Finding Your Vocareum API Key

1. **Navigate to Cloud Resources**:
   - Click the **"Cloud Resources"** button in the left navigation pane
   - This is available in the Udacity classroom workspace

2. **Locate Your Key**:
   - Under the **"OpenAI Key"** header, you'll see:
     - Your API key (format: `voc-...`)
     - Budget information (e.g., "You have $4.97 left of your $5 budget")

3. **Monitor Your Budget**:
   - Check back regularly to monitor usage
   - Budgets are typically shared across the program
   - Avoid using models/endpoints beyond those indicated in course content

## Configuration Methods

### Method 1: Environment Variables (Recommended)

This is the cleanest approach and works well with the existing codebase.

**Step 1: Create/Update `.env` file**
```bash
OPENAI_API_KEY="voc-your-actual-key-here"
OPENAI_API_BASE="https://openai.vocareum.com/v1"
```

**Step 2: Load in your code**
```python
import os
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()

# OpenAI client will use OPENAI_API_BASE if set as environment variable
client = OpenAI(
    api_key=os.getenv("OPENAI_API_KEY"),
    base_url=os.getenv("OPENAI_API_BASE")
)
```

**Step 3: Use with LLM class**
```python
import os
from dotenv import load_dotenv
from lib.llm import LLM

load_dotenv()

# Set base_url in environment before creating LLM
os.environ["OPENAI_API_BASE"] = os.getenv("OPENAI_API_BASE", "https://openai.vocareum.com/v1")

llm = LLM(api_key=os.getenv("OPENAI_API_KEY"))
```

### Method 2: Direct Client Configuration

If you prefer to configure directly without environment variables:

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://openai.vocareum.com/v1",
    api_key="voc-your-actual-key-here"
)
```

**Note**: The current `LLM` class in `lib/llm.py` doesn't directly support `base_url` parameter. You'll need to either:
- Modify the `LLM` class to accept `base_url` parameter, OR
- Set `OPENAI_API_BASE` environment variable before initialization

### Method 3: Modifying LLM Class (Advanced)

If you want to add `base_url` support to the `LLM` class, modify `lib/llm.py`:

```python
class LLM:
    def __init__(
        self,
        model: str = "gpt-4o-mini",
        temperature: float = 0.0,
        tools: Optional[List[Tool]] = None,
        api_key: Optional[str] = None,
        base_url: Optional[str] = None  # Add this parameter
    ):
        self.model = model
        self.temperature = temperature
        
        # Configure client with base_url if provided
        client_kwargs = {}
        if api_key:
            client_kwargs["api_key"] = api_key
        if base_url:
            client_kwargs["base_url"] = base_url
            
        self.client = OpenAI(**client_kwargs) if client_kwargs else OpenAI()
        # ... rest of the code
```

Then use it:
```python
llm = LLM(
    api_key="voc-your-key",
    base_url="https://openai.vocareum.com/v1"
)
```

## Configuration for Different Components

### 1. Standard OpenAI Client

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://openai.vocareum.com/v1",
    api_key=os.getenv("OPENAI_API_KEY")
)
```

### 2. ChromaDB with OpenAI Embeddings

ChromaDB's `OpenAIEmbeddingFunction` may need special handling. Check if it supports `base_url`:

```python
from chromadb.utils import embedding_functions

# Check ChromaDB documentation for base_url support
# You may need to set environment variables:
os.environ["OPENAI_API_BASE"] = "https://openai.vocareum.com/v1"

embedding_fn = embedding_functions.OpenAIEmbeddingFunction(
    api_key=os.getenv("OPENAI_API_KEY")
)
```

### 3. REST API Calls

If making direct REST API calls (curl, requests, etc.), use:
- Base URL: `https://openai.vocareum.com/v1`
- Instead of: `https://api.openai.com/v1`

Example:
```python
import requests

response = requests.post(
    "https://openai.vocareum.com/v1/chat/completions",
    headers={
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json"
    },
    json={
        "model": "gpt-4o-mini",
        "messages": [{"role": "user", "content": "Hello!"}]
    }
)
```

## Troubleshooting

### Error: "Incorrect API key provided"

**Cause**: The API key is being sent to the wrong endpoint (standard OpenAI instead of Vocareum).

**Solution**: Ensure `base_url` is set to `https://openai.vocareum.com/v1`

### Error: "The api_key client option must be set"

**Cause**: API key is not being passed or loaded correctly.

**Solution**: 
- Verify `.env` file exists and contains `OPENAI_API_KEY`
- Ensure `load_dotenv()` is called before creating clients
- Check that the key starts with `voc-`

### ChromaDB Embeddings Not Working

**Cause**: ChromaDB's embedding function may not respect `base_url` parameter.

**Solution**:
- Set `OPENAI_API_BASE` environment variable before initializing ChromaDB
- Check ChromaDB version and documentation for Vocareum support
- Consider using environment variables globally

### Budget Exceeded

**Cause**: You've used up your allocated budget.

**Solution**:
- Check remaining budget in "Cloud Resources" tab
- Use your personal OpenAI API key for local development
- Contact Udacity support if you believe there's an error

## Using Personal OpenAI Key (Fallback)

If Vocareum keys don't work or you've exceeded your budget, you can use your personal OpenAI key:

1. Get your key from: https://platform.openai.com/api-keys
2. Update `.env`:
   ```
   OPENAI_API_KEY="sk-your-personal-key"
   OPENAI_API_BASE="https://api.openai.com/v1"  # Standard OpenAI endpoint
   ```

## Best Practices

1. **Never commit API keys**: Always use `.env` files and ensure they're in `.gitignore`
2. **Monitor budget**: Check "Cloud Resources" regularly
3. **Use environment variables**: Makes switching between Vocareum and personal keys easier
4. **Test configuration**: Verify your setup works before running expensive operations
5. **Handle errors gracefully**: Implement proper error handling for API failures

## Example: Complete Setup

```python
# setup.py or at the top of your notebook
import os
from dotenv import load_dotenv
from openai import OpenAI
from lib.llm import LLM

# Load environment variables
load_dotenv()

# Configure for Vocareum
VOCAREUM_BASE_URL = "https://openai.vocareum.com/v1"
api_key = os.getenv("OPENAI_API_KEY")
base_url = os.getenv("OPENAI_API_BASE", VOCAREUM_BASE_URL)

# Set environment variable for libraries that read it
os.environ["OPENAI_API_BASE"] = base_url

# Create OpenAI client
client = OpenAI(api_key=api_key, base_url=base_url)

# Create LLM instance
llm = LLM(api_key=api_key)

# Verify it works
try:
    response = llm.invoke("Hello!")
    print("✓ Configuration successful!")
except Exception as e:
    print(f"✗ Configuration error: {e}")
```

## Additional Resources

- [OpenAI Python SDK Documentation](https://github.com/openai/openai-python)
- [Vocareum Documentation](https://www.vocareum.com/) (if available)
- Udacity Course Materials and Forums
