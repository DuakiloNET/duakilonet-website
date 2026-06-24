---
title: "Claude API vs OpenAI API: A Python Developer's Honest Comparison"
date: 2026-06-16
description: "Perbandingan langsung antara Claude API dan OpenAI API untuk developer Python — dari setup, streaming, hingga tool use dan pricing."
tags: ["claude", "openai", "llm-api", "python", "ai-tools"]
categories: ["ai-dev"]
draft: false
showHero: false
showTableOfContents: true
---

Kalau kamu developer Python yang baru mulai eksplorasi LLM API, pertanyaan pertama biasanya sama: **Claude atau OpenAI?** Artikel ini bukan marketing piece — ini perbandingan teknis berdasarkan penggunaan nyata.

## Setup Awal

Keduanya straightforward, tapi ada perbedaan kecil yang penting.

**OpenAI:**
```bash
pip install openai
```

```python
from openai import OpenAI

client = OpenAI(api_key="sk-...")
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello"}]
)
print(response.choices[0].message.content)
```

**Claude (Anthropic):**
```bash
pip install anthropic
```

```python
import anthropic

client = anthropic.Anthropic(api_key="sk-ant-...")
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}]
)
print(response.content[0].text)
```

Perbedaan pertama yang langsung terasa: Claude menggunakan `max_tokens` sebagai **required parameter**. OpenAI tidak. Ini bukan bug, ini design choice — Claude lebih eksplisit soal batas output.

## System Prompt

Keduanya support system prompt, tapi caranya berbeda.

**OpenAI** — system prompt masuk sebagai message pertama dengan role `system`:
```python
messages = [
    {"role": "system", "content": "Kamu adalah code reviewer senior."},
    {"role": "user", "content": "Review kode ini..."}
]
```

**Claude** — ada parameter terpisah:
```python
client.messages.create(
    model="claude-opus-4-8",
    max_tokens=2048,
    system="Kamu adalah code reviewer senior.",
    messages=[{"role": "user", "content": "Review kode ini..."}]
)
```

Pendekatan Claude lebih clean secara arsitektur — system prompt tidak bercampur dengan conversation history.

## Streaming

Kedua API support streaming, dan pattern-nya mirip:

**OpenAI:**
```python
with client.chat.completions.stream(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Tulis essay panjang..."}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

**Claude:**
```python
with client.messages.stream(
    model="claude-opus-4-8",
    max_tokens=4096,
    messages=[{"role": "user", "content": "Tulis essay panjang..."}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

Quasimirip. Tapi Claude punya `stream.get_final_message()` yang berguna kalau kamu perlu akses usage stats setelah streaming selesai.

## Tool Use / Function Calling

Ini salah satu area di mana keduanya punya filosofi yang sedikit berbeda.

**OpenAI tools:**
```python
tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "Ambil cuaca saat ini",
        "parameters": {
            "type": "object",
            "properties": {
                "location": {"type": "string"}
            },
            "required": ["location"]
        }
    }
}]

response = client.chat.completions.create(
    model="gpt-4o",
    tools=tools,
    messages=[{"role": "user", "content": "Cuaca Jakarta?"}]
)

# Cek apakah ada tool call
if response.choices[0].message.tool_calls:
    tool_call = response.choices[0].message.tool_calls[0]
    args = json.loads(tool_call.function.arguments)
```

**Claude tools:**
```python
tools = [{
    "name": "get_weather",
    "description": "Ambil cuaca saat ini",
    "input_schema": {
        "type": "object",
        "properties": {
            "location": {"type": "string"}
        },
        "required": ["location"]
    }
}]

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "Cuaca Jakarta?"}]
)

# Cek tool use
for block in response.content:
    if block.type == "tool_use":
        print(block.name, block.input)
```

Claude menggunakan `input_schema` bukan `parameters`, dan response-nya berupa list of content blocks — lebih konsisten dengan arsitektur message-nya secara keseluruhan.

## Context Window

Ini salah satu keunggulan terbesar Claude saat ini:

| Model | Context Window |
|-------|---------------|
| GPT-4o | 128K tokens |
| GPT-4o mini | 128K tokens |
| Claude Opus 4.8 | **1M tokens** |
| Claude Sonnet 4.6 | **1M tokens** |
| Claude Haiku 4.5 | 200K tokens |

1 juta tokens itu cukup untuk membaca beberapa novel sekaligus, atau seluruh codebase medium-sized project. Ini game changer untuk use case seperti:
- Analisis seluruh repository
- Processing dokumen panjang
- Long-running conversation dengan banyak context

## Error Handling

**OpenAI** menggunakan exception hierarchy yang straightforward:
```python
from openai import RateLimitError, APIError

try:
    response = client.chat.completions.create(...)
except RateLimitError as e:
    time.sleep(60)
    retry()
except APIError as e:
    print(f"Error: {e.status_code}")
```

**Claude** punya pattern serupa:
```python
import anthropic

try:
    response = client.messages.create(...)
except anthropic.RateLimitError as e:
    time.sleep(60)
    retry()
except anthropic.APIError as e:
    print(f"Error: {e.status_code}")
```

Hampir identik. SDK keduanya mature dan well-designed.

## Pricing (Juni 2026)

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|-------|----------------------|------------------------|
| GPT-4o | ~$5 | ~$15 |
| GPT-4o mini | ~$0.15 | ~$0.60 |
| Claude Opus 4.8 | $5 | $25 |
| Claude Sonnet 4.6 | $3 | $15 |
| Claude Haiku 4.5 | $1 | $5 |

Untuk production workload yang budget-sensitive, GPT-4o mini dan Claude Haiku 4.5 adalah pilihan ekonomis dengan kualitas yang masih solid untuk banyak task.

## Kapan Pilih Mana?

**Pilih Claude kalau:**
- Butuh context window sangat besar (analisis codebase, dokumen panjang)
- Nuance dalam instruction following sangat penting
- Kerjaan memerlukan reasoning yang lebih dalam
- Budget untuk tier Opus/Sonnet

**Pilih OpenAI kalau:**
- Ekosistem dan tooling yang lebih mature (fine-tuning, DALL-E, Whisper, dll)
- Butuh multimodal yang lebih kaya (vision + generation)
- Tim sudah familiar dengan OpenAI API
- Deployment di Azure OpenAI Service

**Pakai keduanya kalau:**
- Butuh redundancy dan fallback
- A/B testing model untuk use case spesifik
- Different models untuk different tasks dalam pipeline yang sama

## Kesimpulan

Tidak ada yang "lebih baik" secara mutlak. Keduanya excellent API dengan SDK Python yang mature dan well-documented.

Dari pengalaman saya: Claude unggul dalam task yang butuh nuanced understanding dan long context, sementara OpenAI punya ekosistem yang lebih luas. Untuk pure coding assistant atau API integration, keduanya sangat kompetitif.

Satu tip praktis: coba keduanya untuk use case spesifik kamu dengan dataset yang sama. Hasilnya seringkali mengejutkan.

---

*Tertarik dengan satu aspek spesifik yang ingin dibahas lebih dalam? Drop comment di bawah.*
