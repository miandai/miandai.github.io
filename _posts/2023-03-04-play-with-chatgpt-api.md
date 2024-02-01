---
layout: post
keywords: chatgpt
description: chatgpt
title: "Play with ChatGPT API"
categories: [Transformer]
tags: [Transformer]
---

* content
{:toc}

# Shell

```
curl https://api.openai.com/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer $OPENAI_API_KEY' \
  -d '{
  "model": "gpt-3.5-turbo",
  "messages": [{"role": "user", "content": "Hello!"}]
}'
```

# Python

这里介绍了三个角色，完整的用法是三个都用，正如 Assistant Role 代码所示。




## User Role

```python
import os
import openai
openai.api_key = os.getenv("OPENAI_API_KEY")

completion = openai.ChatCompletion.create(
  model="gpt-3.5-turbo",
  messages=[
    {"role": "user", "content": "Who won the world series in 2020?"}
  ]
)

print(completion.choices[0].message.content)
```

## System Role

```python
import openai

openai.api_key = os.getenv("OPENAI_API_KEY")

messages = [
 {"role": "system", "content" : "You’re a kind helpful assistant"}
]

content = input("User: ")
messages.append({"role": "user", "content": content})

completion = openai.ChatCompletion.create(
  model="gpt-3.5-turbo",
  messages=messages
)

chat_response = completion.choices[0].message.content
print(f'ChatGPT: {chat_response}')
```


## Assistant Role

```python
import openai

openai.api_key = os.getenv("OPENAI_API_KEY")

messages = [
 {"role": "system", "content" : "You’re a kind helpful assistant"}
]

while True:
    content = input("User: ")
    messages.append({"role": "user", "content": content})
    
    completion = openai.ChatCompletion.create(
      model="gpt-3.5-turbo",
      messages=messages
    )

    chat_response = completion.choices[0].message.content
    print(f'ChatGPT: {chat_response}')
    messages.append({"role": "assistant", "content": chat_response})
```

# 参考资料

- [API keys](https://platform.openai.com/account/api-keys)

- [A Simple Guide to The (New) ChatGPT API with Python](https://medium.com/geekculture/a-simple-guide-to-chatgpt-api-with-python-c147985ae28)