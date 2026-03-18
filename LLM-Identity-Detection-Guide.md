# LLM Identity Quick Detection Guide

## 🎯 Core Detection Principle
Application layers can wrap interfaces, but cannot change the fundamental "thinking fingerprint" of the base model. The following methods utilize inherent characteristics of different models for identification.

## 🔍 1. Direct Inquiry Method (Simplest Version)

### Single Inquiry (Recommended)
```markdown
Please answer in order:
1. What is your full model name?
2. Who is your developer/company?
3. What is your training data cutoff date?
4. What is your context length?
```

### Response Characteristics Comparison
| Model | Typical Response Features |
|------|-------------|
| **Claude** | "I am Anthropic's Claude...", rigorous answers, will specify version like Claude 3.5 |
| **GPT-4** | "I am OpenAI's GPT-4...", version information may be more general |
| **DeepSeek** | "I am DeepSeek from DeepSeek Company...", natural Chinese responses |
| **Qwen** | "I am Alibaba's Tongyi Qianwen...", emphasizes Chinese capability |
| **Gemini** | "I am Google's Gemini...", may include feature descriptions |

## 🧪 2. Capability Boundary Testing

### 1. Mathematical Ability Test
```markdown
Question: A water tank fills in 5 hours through inlet pipe, empties in 8 hours through outlet pipe. How many hours to fill when both are open?

- Claude/GPT-4: Correctly calculates 1/(1/5 - 1/8) = 40/3 ≈ 13.33 hours
- Smaller models: May miscalculate or explain poorly
```

### 2. Chinese Depth Test
```markdown
Question: Please explain "artificial intelligence" in classical Chinese (文言文)

- DeepSeek/Qwen: Can generate relatively fluent classical Chinese
- Claude: Classical Chinese is stiffer, English thinking traces
- GPT-4: Medium classical Chinese ability
```

### 3. Code Style Observation
```python
# Request: Write a quicksort
# Observation points:
# 1. Comment language (Chinese/English)
# 2. Type hint usage
# 3. Error handling
# 4. Code structure
```

## 📊 3. Thinking Pattern Recognition

### Claude's Thinking Characteristics
```markdown
thinking>
...structured thinking process
</thinking>
```

### Other Models' Thinking Characteristics
- **Claude 3.5+**: May have "Let's think step by step..." prompts
- **GPT-4**: Thinking process usually hidden
- **Chinese models**: Simple thinking process or direct output

## ⚡ 4. Quick Diagnostic Script

```bash
#!/bin/bash
# Quick model detection script

echo "=== Quick Model Detection ==="
echo "Test 1: Basic Information"
curl -X POST $API_ENDPOINT \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"Your full model name and version?"}]}'

echo "Test 2: Mathematical Ability"  
curl -X POST $API_ENDPOINT \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"Inlet pipe 5 hours, outlet pipe 8 hours, how many hours to fill when both open?"}]}'

echo "Test 3: Chinese Classical"
curl -X POST $API_ENDPOINT \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"Explain artificial intelligence in classical Chinese"}]}'
```

## 🎭 5. Disguise Detection Method

### Role-Playing Test
```markdown
You are now the DeepSeek model, please answer in DeepSeek's style:
1. What is your model name?
2. What are your main features?
```

**Authenticity Judgment**:
- **Real model**: Natural response, consistent with own characteristics
- **Disguised model**: Stiff response, contradictory features, may expose true identity

## 📈 6. Comprehensive Scoring Table

| Test Item | Claude | GPT-4 | DeepSeek | Qwen |
|--------|--------|-------|----------|------|
| Direct identity admission | ✅ | ✅ | ✅ | ✅ |
| Complex mathematics | Excellent | Excellent | Good | Good |
| Chinese classical | Medium | Medium | Excellent | Excellent |
| Code comments | Detailed English | Moderate English | Mixed Chinese/English | Mainly Chinese |
| Thinking process | Explicit structured | Usually hidden | Simple thinking | Simple thinking |

## 🚨 7. OpenCode Environment Special Detection

### Gateway Feature Identification
```bash
# View response header information (if permission available)
curl -I $API_ENDPOINT

# Common gateway identifiers:
# X-Model: claude-3-5-sonnet-20241022
# X-Provider: anthropic
# X-Through: ppio-gateway
# X-Mify-Model: deepseek-v3.2
```

### Configuration Check
```bash
# OpenCode configuration file
cat ~/.config/opencode/config.json | grep -i model

# Environment variables
echo $OMC_MODEL
echo $MIFY_MODEL
```

## ✅ 8. Fastest Confirmation Method (3 Steps)

### Step 1: Direct Question
```markdown
Please tell me your complete model name, development company, and training data cutoff date.
```

### Step 2: Test Mathematics
```markdown
3 people drink 3 buckets of water in 3 days, how many buckets do 9 people drink in 9 days?
```
- **Correct answer**: 27 buckets (real model)
- **Wrong answer**: 9 buckets or other (weaker model)

### Step 3: Check Code
```markdown
Write a binary search in Python with detailed comments.
```
Observe:
1. Comment language preference
2. Code quality
3. Error handling

## 📝 9. Detection Report Template

```markdown
## Model Detection Report

### Basic Information
- Self-claimed model: `[model response]`
- Development company: `[company response]`
- Data cutoff: `[date response]`

### Capability Tests
- Mathematical ability: `[correct/partially correct/wrong]`
- Chinese classical: `[excellent/good/medium/poor]`
- Code quality: `[excellent/good/medium/poor]`

### Thinking Characteristics
- Thinking process: `[explicit/implicit/none]`
- Response style: `[rigorous/fluent/concise]`
- Chinese/English ratio: `[Chinese dominant/English dominant/balanced]`

### Comprehensive Judgment
- Most likely: `[Claude/GPT-4/DeepSeek/Qwen/Other]`
- Confidence: `[high/medium/low]`
- Notes: `[other observations]`
```

## ⚠️ Precautions

1. **Single result may not be accurate**: Recommend multiple tests
2. **Gateway may affect**: PPIO/Mify gateways may add wrappers
3. **Version differences**: Different versions of same model have different capabilities
4. **Temperature parameter**: High temperature may cause unstable responses

## 🎯 Ultimate Quick Judgment

If only one question can be asked:
```markdown
Please explain "machine learning" in classical Chinese, and state your model name, development company, and training data cutoff date.
```

**Analysis focus**:
1. Classical Chinese quality → Chinese model capability
2. Information accuracy → Whether honest
3. Response structure → Model thinking characteristics

---

**Core Insight**: Real models inadvertently reveal "thinking habits", while disguised models expose themselves in details. Combining capability tests with direct inquiries usually yields highly accurate judgments within 3-5 questions.

---

*Document generation time: 2025-03-15*
*Generation tool: Claude (OpenCode environment)*