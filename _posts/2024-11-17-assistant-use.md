---
layout: post
title: Quickly build an app to chat with your multi-function Assistant
date: 2024-11-17 12:55:00-0400
description: How to send and receive messages with your OpenAI Assistant
tags: openai assistant
categories: assistant
giscus_comments: true
related_posts: false
toc:
  beginning: true
---

# Overview
Hi. In my previous [post](https://vyhaophan.github.io/blog/2024/assistant-function-call/), but it is not enough. If you just have the Assistant on the OpenAI side, you might find it complicated and inconvenient to use. Everytime you want it to make an analysis on sales data, you have to go to the OpenAI side and copy the arguments from the chat history which then are used to put into tools like SQL and other visualisation tools, so you'll have the visualisation data you want.

Is the purpose of all technologies the most convenient for users? I think absolutely yes. 

Therefore, I think I need to build an app to chat with my assistant. The app must be able to send and receive messages with the assistant, display the conversation, and also be able to use the assistant's tools. And the most important thing is that it must be convenient.

Basically, there are two main problems to solve here:
1. **How to send and receive messages with the assistant** (the "backend" part)
2. **How to display the conversation on the UI** (the "frontend" part) 

Each problem requires a different approach, but it is not too difficult to implement. In this post, I will show you how to implement the "backend" part with the OpenAI Python client. For the "frontend" part, I will show in the next post.

# Send and receive messages with the assistant
There are some new concepts that you should understand to master this section of interacting with the assistant:
- **Thread**: A thread is a container for messages. You can think of it as a chat history.
- **Run**: A run is an execution of an assistant. It is created when you send a message to the assistant.
- **Message**: A message is a message sent to and by the assistant. There are HumanMessage and AIMessage.

So, to send a message to the assistant, you need to create a thread first, then create a run, and then send the message to the assistant. Below is the diagram showing how it works:

```typograms
    +---------------------------+
    |         Assistant         |
    +---------------------------+
                |
                V
    +---------------------------+
    |     Create a thread       |
    +---------------------------+
                |
                V
    +---------------------------+
    | User chat with assistant  |
    +---------------------------+
                |
                V
    +---------------------------+         +----------------------------+
    |   Add message to thread   | ------> | Create a run on the thread |
    +---------------------------+         +----------------------------+
                ^                                       |
                |                                       V
                |                       +---------------------------------+
                |                       | Receive response from assistant |
                |                       +---------------------------------+
                |                                       |
                |                                       V
    +---------------------------+         +----------------------------+
    | Add AI Message to thread  | <------ | Display the conversation   |
    +---------------------------+         +----------------------------+

```
Is there anything vague here? Please let me know if you have any questions.

We will work on each step one by one.

## Create a thread
To create a thread, we will use the `threads.create` method from the OpenAI Python client.

```python
from openai import OpenAI
client = OpenAI(api_key=os.getenv("YOUR_OPENAI_API_KEY"))
thread = client.beta.threads.create()
print("Thread created successfully with id: {}".format(thread.id))
```

If you run the code successfully, you should see the thread id printed out.
```
Thread created successfully with id: thread_Y6bDhi7KBBwdMwTLUAYuWPq8
```
## Add a message to the thread
You can add a message to the thread by using the `messages.create` method from the OpenAI Python client.
```python
message = client.beta.threads.messages.create(
    thread_id=thread.id,
    role="user",
    content="Hello, how are you?"
)
```

## Create a run and receive response from the assistant
To create a streaming run, you can use the `runs.stream` method from the OpenAI Python client. The streaming option helps you get the response from the assistant in real-time and much more faster than the non-streaming option.

```python
with client.beta.threads.runs.stream(
    thread_id=thread.id, assistant_id="Your_Assistant_ID"
) as stream:
    for event in stream:
        # do something with the event
    stream.until_done()
```
There are two types of events you can receive:
- `message`: A new message is created.
- `run.requires_action`: The assistant is calling a tool.

For the `message` type, you just need to yield the event content to the UI by accessing the `event.data.delta.content[0].text.value` attribute.

```python
with client.beta.threads.runs.stream(
    thread_id=thread.id, assistant_id="Your_Assistant_ID"
) as stream:
    for event in stream:
      if event.event == "thread.message.delta" and event.data.delta.content:
        yield event.data.delta.content[0].text.value
    stream.until_done()
```
It's great now, we can send and receive messages with the assistant. However, when you just run the code you won't receive the tokens returned from Assistant. The reason is that the `run` returns a **generator** object which is a kind of Python function and returns an iterator with using the `yield` keyword rather than `return`. Therefore, you need to treat it as a generator object, so what need to do here is to place the stream in a generator function with the `next` method or `for in` loop.

Here is the simple example of a generator function for the stream:

```python
def stream_response(user_input, thread_id, assistant_id):
  message = client.beta.threads.messages.create(
      thread_id=thread_id,
      role="user",
      content=user_input
  )
  with client.beta.threads.runs.stream(
      thread_id=thread_id, assistant_id=assistant_id
  ) as stream:
      for event in stream:
        if event.event == "thread.message.delta" and event.data.delta.content:
          yield event.data.delta.content[0].text.value
      stream.until_done()
```
To use this function, a `for in` loop is necessary.

```python
response = stream_response("Hi what can you do?", thread_id, assistant_id)
full_response='' # this variable is used to collect each chunk of response, create a full response
for chunk in response:
  if chunk:
      chunk_data = chunk.replace("data: ", "").strip()
      print(chunk_data, end=" ", flush=True)
      full_response += chunk_data + " "
```
Running the code above, you should see the response from the assistant.
```
Hi there ! Here's how I can assist you with the Xi Pho Coffee shop : - ** Inventory Management :** Check the inventory to identify products that are running out of stock and need to be rest ocked . - ** Financial Analysis :** Provide insights into daily revenue and profit for specific dates . - ** Business Operations Support :** Offer advice and information to improve management and operational efficiency . If there's anything specific you need , feel free to ask ! 
```

After getting the full response, remember to add the Assistant message to the history, similarly to the user message.

```python
message = client.beta.threads.messages.create(
    thread_id=thread.id,
    role="assistant",
    content=full_response
)
```

There is another way of retrieving the full response from the assistant while the `run` is still in progress. You can detect whether the run is completed and get the content of that event.
```python
from datetime import datetime
def stream_response(user_input, thread_id, assistant_id):
  ...
      for event in stream:
        ...
        if event.event == 'thread.message.completed' and event.data.content and event.data.completed_at:
          assistant_response = event.data.content[0].text.value # this is the full response from the assistant
          dt = datetime.fromtimestamp(event.data.completed_at)
          print("Message completed at {}".format(dt.strftime('%Y %b %d, %H:%M:%S')))
      stream.until_done()
```
You have stored the full response in the `assistant_response` variable.

## Create a run for the assistant's tool calls
However, how the assistant calls a tool is a different story.
- That's when the function call is triggered because the assistant needs to use a tool.
- The function with arguments is returned in the `event.data.required_action` attribute.
- An action must be taken to process the arguments. The action result is submitted back to the `run`.
- After having action result, the assistant will continue the conversation.

So what we need to do here is:
- Catch the function call event, get the function and arguments. Note that at a time, there could be multiple tools called.
- Run a tool with the arguments.
- Submit the action result back to the `run` event.

```python
import json
def stream_response(user_input, thread_id, assistant_id):
  ...
      for event in stream:
        ...
        if event.event == "thread.run.requires_action":
            # get the tool calls from the event
            tool_calls = event.data.required_action.submit_tool_outputs.tool_calls
            tool_results = []
            # process the tool calls
            for tool_call in tool_calls:
                tool_call_id = tool_call.id
                function_name = tool_call.function.name
                function_arguments = json.loads(tool_call.function.arguments)
                ...
      stream.until_done()
```
What have we done here?
- We have caught the function call event.
- We assigned all tool calls to a list named `tool_calls`.
- With each call, we have extracted the function ID, name and arguments.

Now we need to do something with the function to get the result and submit it back to the `run` event.

As discussed in my previous [post](https://vyhaophan.github.io/blog/2024/assistant-function-call/), we have defined 2 functions: `CheckOutOfStock` and `CheckDailyRevenue`. Now, with each call, we assign the appropriate function by the variable `function_name`. If there is no error occurred, we append the processed result to the `tool_results` list for later submission.

```python
if function_name == "CheckOutOfStock":
    function_result = check_out_of_stock() # this function requires no arguments
elif function_name == "CheckDailyRevenue":
    function_result = check_daily_revenue(function_arguments.date) # this function requires a date argument

if function_result != "An error occurred while connecting to the database.":
    print("Results retrieved from database successfully!")
    tool_results.append({
        "id": tool_call_id,
        "output": function_result
    })
```
Now we have successfully processed the tool calls and get the results. The last step is to submit the action result back to the `run` event and let the assistant continue the conversation.

```python
for tool_event in client.beta.threads.runs.submit_tool_outputs(
    thread_id=thread_id,
    run_id=event.data.id,
    tool_outputs=tool_results, 
    stream=True
):
    if tool_event.event == "thread.message.delta" and tool_event.data.delta.content:
        yield tool_event.data.delta.content[0].text.value
```
Good job! For tool calls, here is the full code:
```python
def stream_response(user_input, thread_id, assistant_id):
  ...
      for event in stream:
        ...
        if event.event == "thread.run.requires_action":
            # get the tool calls from the event
            tool_calls = event.data.required_action.submit_tool_outputs.tool_calls
            tool_results = []
            # process the tool calls
            for tool_call in tool_calls:
                tool_call_id = tool_call.id
                function_name = tool_call.function.name
                function_arguments = json.loads(tool_call.function.arguments)
                if function_name == "CheckOutOfStock":
                    function_result = check_out_of_stock()
                elif function_name == "CheckDailyRevenue":
                    function_result = check_daily_revenue(function_arguments.date)

                if function_result != "An error occurred while connecting to the database.":
                    print("Results retrieved from database successfully!")
                    tool_results.append({
                        "id": tool_call_id,
                        "output": function_result
                    })
            for tool_event in client.beta.threads.runs.submit_tool_outputs(
                thread_id=thread_id,
                run_id=event.data.id,
                tool_outputs=tool_results, 
                stream=True
            ):
                if tool_event.event == "thread.message.delta" and tool_event.data.delta.content:
                    yield tool_event.data.delta.content[0].text.value
      stream.until_done()
```
## Get the AI response and conversation from the thread
Due to the streaming nature of the `run`, the AI response is added to the thread immediately at the final step of `run`. The `thread` can be seen as the Assistant's memory, so both the user questions and the AI responses are stored there. Below is how you can get the latest AI response from the thread.

```python
# use the thread id to get the latest message
messages = client.beta.threads.messages.list(thread_id=thread_id)
# get the latest message
latest_message = messages.data[0].content[0].text.value
```
This may help if you want to retrieve and continue the conversation after ending it. For example, you end the chat, and return to continue the conversation after 10 minutes, you may not want to start a new chat but start with the previous thread.

I hope you find this post helpful. If you have any questions or feedback, please let me know. Thank you for reading!





