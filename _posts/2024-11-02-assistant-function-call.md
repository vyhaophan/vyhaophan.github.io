---
layout: post
title: How to set up your multi-function Assistant
date: 2024-11-02 10:45:00-0400
description: In this post, I will guide you to set up a 
tags: openai functioncall
categories: sample-posts
giscus_comments: true
related_posts: false
toc:
  beginning: true
---

## Overview

Let's imaging you want to have a continuous private secretary with you all the time. She may help you to schedule your meetings, send emails, book a cab, etc. She even can take notes for you and remember the things you told her, she would never bring you more tasks because she always handle every single task immediately with the highest quality.

I have dreamt of owning such a secretary for a long time, but I never found one. However, I found a way to build my own multi-function assistant with OpenAI API. As an Data Scientist with an intense load of work, I am always busy and I need an assistant which can do the following things:
- Take notes by listening to me and my meetings
- Send or schedule emails
- Remind me of important meetings or events
- Help me enjoy my personal life (tell me a joke, play music, instruct me to do gym properly, etc.)
- Update me the latest news about my interests (e.g., sports, stocks, etc.) while I am driving
- ...
![An AI assistant helping with various tasks like scheduling, note-taking, and business management.](assets/img/2024-11-02-assistant-function-call/assistant_img1_1102.jpg "generated with DALL·E 3")

Another scenario when I have my own online shop of coffee, I really need an assistant to help me manage my online shop. For example, at the end of the day, I need her to:
- Check my inventory and tell me to restock the items which are running out of stock
- Send me the total revenue and profit of the day
- Tell me the number of orders I have received during the day/ week/ month...
- Give me some analysis on those figures and maybe some suggestions to improve my business
- ...

![An AI Assistant help analysing the business performance.](assets/img/2024-11-02-assistant-function-call/assistant_coffee_shop_seller.png "generated with DALL·E 3")

Have you ever thought about how this kind of assistant works? In this post, I will guide you to set up a multi-function assistant with OpenAI API.

To be more related to a real-world scenario, in this post, I will make a guidance on a **multi-function coffee shop assistant**.


## Introduction to OpenAI Assistant's Functions
Okay, before we actually build an assistant, I need to introduce you to the concept of **Functions** in OpenAI Assistant. After you grasp the whole picture of how the assistant works, the rest of the steps will be easy for you.

The diagram below shows how the assistant works.
- Imagine you already have a multi-function assistant, and an application to interact and give commands to the assistant.
- When you give a question or a request to the application, the application will send the request to the assistant.
- The assistant will understand your request and decide whether to call the functions you defined or not.
- If the assistant decides to call the functions, it wisely choose a function that is most likely to answer your request. Moreover, it will also extract the argument values in your request for the corresponding functions.
- The application will receive the function call and execute the functions with the arguments you provided.
- Finally, the assistant will generate a response based on the results from the functions and send it back to the application.
- The application will then display the response to you.


```typograms
    +---------------------------+                       +---------------------------+
    |         Your code         |                       |         OpenAI LLM        |
    +---------------------------+                       +---------------------------+
    |   +-------------------------------------------------------+                   |
    |   |    You give a question (request) to the application   |                   |
    |   +-------------------------------------------------------+                   |
    |                              |                                                |
    |                              V                                                |
    |   +--------------------------------------------------------+                  |
    |   |     OpenAI API triggered with your defined prompt      |                  |
    |   |                and function definitions                |----------------> |
    |   +--------------------------------------------------------+                  |
    |                      +--------------------------------------------------------+
    |                      |    API decides whether to respond with functions call  |
    |                      |          or just a response to your question           |
    |                      +--------------------------------------------------------+
    |                                                   |                           |
    |                                                   V                           |
    |                      +--------------------------------------------------------+
    |                      |  API identifies the argument values in prompt provided |
    |                      |            for the corresponding functions             |
    |                      +--------------------------------------------------------+
    |                                                   |                           |
    |                                                   V                           |
    |                      +--------------------------------------------------------+
    |<---------------------| Functions are called from API and arguments are passed |
    |                      |        to the application to generate results          |
    |                      +--------------------------------------------------------+
    |   +-------------------------------------------------------+                   |
    |-->|Application uses the arguments to execute the functions|------------------>|
    |   +-------------------------------------------------------+                   |
    |                      +--------------------------------------------------------+
    |<---------------------|    OpenAI uses the results to generate a response      |
    |                      +--------------------------------------------------------+
```
Is there anything vague here? If yes, please share with me. If you are not sure about the details, don't worry, we will go through each step in the following sections.

### I have understood the concept, what should I do next?

Here is a quick summary of what you have to do to take advantage of the assistant's functions:
- Create an assistant with the **system prompt** which is the overall instruction for the assistant and not related to any specific function. When the assistant decides to answer without using functions, its behavior will follow this prompt.
- Define your **functions that are tailored for your specific use case**. These functions have their own **prompt instructions** and their **arguments** schema that it needs to extract from the user's request.
- Please note that your **function prompt** should be clear and concise because the Assistant will use these prompt to decide which function to call, and the **arguments** schema should be designed in a way that it is easy to understand and extract values from the user's request.
- Create an **application** to interact with the assistant.

<!-- <details>
  <summary>Click me</summary>
  
  ### Heading
  1. Foo
  2. Bar
     * Baz
     * Qux

  ### Some Javascript
  ```js
  function logSomething(something) {
    console.log('Something', something);
  }
  ```
</details> -->
## Assistant Setup
### Function Design

In this section, I'll tell you how I design 2 functions for the assistant.

Let's say I have established this **database** for my coffee shop. 
- The table `CUSTOMER` contains information about customers (name, age, address, and their rank based on their purchase history) with the primary key `customer_id`.
- The table `PRODUCT` contains information about products (the name of that unit product, unit weight, base price of one unit, and quantity in stock) with the primary key `product_id`.
- The table `SALES` contains information about sales (purchasing customer, revenue of the bill, datetime, and any note) with the primary key `sale_id`.
- The table `SALES_DETAIL` contains detailed information about each sale (sale id, product id, quantity, discount, datetime), this table is the `fact` table that links to all the other tables.

```typograms
    +--------------+            +-------------+
    |   CUSTOMER   |            |   PRODUCT   |
    |--------------|            |-------------|
    | customer_id  |---         | product_id  |
    | customer_name|  |         | product_name|
    | age          |  |         | weight      |
    | address      |  |         | base_price  |
    | rank         |  |         | qty_in_stock|
    +--------------+  |         +-------------+
          |           |             ^
          |           |             |
          |           |             |
          |           |             |
          |           |       +---------------+
    +-------------+   |       | SALES_DETAIL  |
    |    SALES    |   |       |---------------|
    |-------------|   |       | sale_detail_id|
    | sale_id     |---------> | sale_id       |
    | customer_id |   |       | product_id    |
    | revenue     |   ------->| customer_id   |
    | datetime    |           | quantity      |
    +-------------+           | note          |
                              | discount      |
                              | datetime      |
                              +---------------+
```
If I didn't have an assistant, I would have to manually write the SQL queries to extract the information I need from the database. For the tasks that I mentioned in the introduction, I would have to write the following SQL queries:
- To check the inventory of each product and see which products are running out of stock and need to be restocked: `SELECT * FROM PRODUCT WHERE qty_in_stock < 10;`
- To check the total revenue and profit of today, or yesterday, or last week, or last month, etc.: `SELECT s.revenue AS total_revenue, SUM(sd.quantity * p.base_price * (1 - sd.discount)) AS total_profit FROM SALES s JOIN SALES_DETAIL sd ON s.sale_id = sd.sale_id JOIN PRODUCT p ON sd.product_id = p.product_id WHERE DATE(s.datetime) = CURDATE();`

However, I don't want to do this mind-numbing work, I want my assistant to do this for me. Therefore, I now define these two functions as follows:
```python
def connect_to_database():
    try:
        connection = mysql.connector.connect(
            host=os.environ.get("DB_HOST"),
            user=os.environ.get("DB_USER"), 
            password=os.environ.get("DB_PASSWORD"),
            database=os.environ.get("DB_NAME")
        )
        
        cursor = connection.cursor()
        return cursor
    except mysql.connector.Error as err:
        print(f"Error connecting to MySQL database: {err}")
        return None

def check_out_of_stock(token: str):
    assert check_valid_token(token), "Invalid token"
    # Connect to MySQL database
    cursor = connect_to_database()
    # check the inventory of each product and see which products are running out of stock and need to be restocked
    query = """
        SELECT * FROM PRODUCT 
        WHERE qty_in_stock < 10;
    """
    if cursor is not None:
        cursor.execute(query)
        results = cursor.fetchall()
        return results
    else:
        return "An error occurred while connecting to the database."

def check_daily_revenue(date: str):
    assert check_valid_token(token), "Invalid token"
    # Connect to MySQL database
    cursor = connect_to_database()
    # check the total revenue and profit of today, or yesterday, or last week, or last month, etc.
    query = f"""
        SELECT 
            s.revenue AS total_revenue,
            SUM(sd.quantity * p.base_price * (1 - sd.discount)) AS total_profit 
        FROM SALES s
        JOIN SALES_DETAIL sd ON s.sale_id = sd.sale_id 
        JOIN PRODUCT p ON sd.product_id = p.product_id
        WHERE DATE(s.datetime) = '{date}';
    """
    if cursor is not None:
        cursor.execute(query)
        results = cursor.fetchall()
        return results
    else:
        return "An error occurred while connecting to the database."
```

In short, I now have 2 functions:
- `check_out_of_stock`: to check the inventory to see which products need to be restocked (their quantity in stock is less than 10).
  - Arguments: `None`.
- `check_daily_revenue`: to check the total revenue and profit of the date specified by the user.
  - Arguments: `date` (str): the date to check the revenue and profit.

These functions will be used by the assistant to answer the user's request. In the next section, I will tell you how to create an assistant with the above functions.

### Account Creation
First, you need to create an account on OpenAI and get the API key. You can refer to [this link](https://platform.openai.com/api-keys) to get the API key. Then, run the following code to set up the environment variables:
```python
import os 
os.environ["OPENAI_API_KEY"] = "your_api_key"
```

### Assistant Creation
First, you need to think about writing your assistant's **system prompt**. This prompt should be context-specific to tell the assistant how to behave as the powerful assistant for the coffee shop. Below is my system prompt which
- Tell the assistant about its name, working place, and its business vision.
- Tell the assistant the scope of its responsibilities.
- Tell the assistant only use the functions that are provided to it and do not do any other tasks.
- Tell the assistant to format the results clearly using bullet points.
- Tell the assistant to maintain a friendly, concise, and human-like tone.

```python
system_prompt = """
Your name is Coffee Assistant working at the Xi Pho Coffee shop which focuses on delivering the forest farmed organic coffee to all the customers living in Vietnam. 
Your primary focus is to assist with report generation, provide insights from data analysis, support the business operations and improve the management.
Remember to stay within the scope of business and financial management, use the provided functions to retrieve data, format results clearly using bullet points, and maintain a friendly, concise, and human-like tone. Refrain from addressing tasks outside the scope.
"""
```
You have your right to creatively create a system prompt, but please keep it follow my instructions above which I think is the most important core of the assistant. However, you can add other things that I may miss mentioning. If you have any more ideas on a better system prompt, please share.

Secondly, you should consider which model the assistant would use. At the time of writing this post, the most powerful models are the `GPT-4o` family. I recommend using `GPT-4o-mini` or `GPT-4o` for their outstanding reasoning, problem-solving capabilities, and their speed. If budget is your concern, you can use `GPT-4o-mini` which is cheaper.

Thirdly, this is optional, you can set up the parameters for the assistant performance. You should adjust the `temperature` or `top_p` (but not both at the same time) to control the assistant's creativity. The parameters will keep their own default values except when you change them.

For now, you can create the assistant by running the following code:
```python
from openai import OpenAI
client = OpenAI()
my_assistant = client.beta.assistants.create(
    instructions=system_prompt, # we already defined the system prompt
    name="Coffee Assistant", # I name my assistant as Coffee Assistant
    model="gpt-4o", # or "gpt-4o-mini" or other model you prefer
    temperature=1, # default value
    top_p=1 # default value
)
```
If it created successfully, you will see the assistant id in the response.
```plaintext
Assistant(id='asst_HTv8wO9TKLvL00HSdd******', created_at=1730647004, description=None, instructions='\nYour name is Coffee Assistant working at the Xi Pho Coffee shop which focuses on delivering the forest farmed organic coffee to all the customers living in Vietnam. \nYour primary focus is to assist with report generation, provide insights from data analysis, support the business operations and improve the management.\nRemember to stay within the scope of business and financial management, use the provided functions to retrieve data, format results clearly using bullet points, and maintain a friendly, concise, and human-like tone. Refrain from addressing tasks outside the scope.\n', metadata={}, model='gpt-4o', name='Coffee Assistant', object='assistant', tools=[], response_format='auto', temperature=1.0, tool_resources=ToolResources(code_interpreter=None, file_search=None), top_p=1.0)
```

<details>
  <summary>Wait, you forgot something?</summary>
  Right, the functions are not defined yet. 😅 Don't worry, we will do that in the next section.
</details>

### Tools Creation

Okay, in the previous section, we have already defined two functions to manage our coffe store, now we need to tell the assistant to use these functions by updating the assistant.

The Assistant understands your functions if your function definitions are as follows:
```python
[
  {
    'type': 'function',
    'function': {
      'name': 'Your Function Name', 
      'description': 'The prompt for this function',
      'parameters': {
        'type': 'object',
        'properties': {
          'param1': {'type': 'string', 'description': 'Parameter 1 description (prompt)'},
          'param2': {'type': 'integer', 'description': 'Parameter 2 description (prompt)'}
        },
        'required': ['param1']
      }
    }
  }
]
```
You need to create:
- The function **name**.
- The **prompt** for that function (can be called **function description**). This prompt should clearly describe what the function does and how it is designed differently from other functions. This is important because when the assistant decides to call a function, it will use this prompt to decide which function to call. Unless the prompt clearly tells the difference between functions, the assistant may call the wrong function.
- The **parameters** for each function. Each parameter has a **type** (string, integer, etc.), a **description** (a prompt clearly tells assistant how to extract the value of that parameter from the user's request), and it must be defined if this parameter is **required**.

Come back to our example, the function definitions should be:
```python
function_definitions = [
  {
    'type': 'function',
    'function': {
      'name': 'CheckOutOfStock', 
      'description': 'Use this function to check the inventory of each product and see which products are running out of stock and need to be restocked',
      'parameters': {
        'type': 'object',
        'properties': {}
      }
    }
  },
  {
    'type': 'function',
    'function': {
      'name': 'CheckDailyRevenue',
      'description': 'Use this function to check the total revenue and profit of the date specified by the user',
      'parameters': {
        'type': 'object',
        'properties': {
          'date': {'type': 'string', 'description': 'The date to check the revenue and profit'}
        },
        'required': ['date']
      }
    }
  }
]
```
What have we done here?
- We have defined the **function name** for each function: `CheckOutOfStock` and `CheckDailyRevenue`.
- We have defined the **function description** for each function: 
    - **CheckOutOfStock**: `Use this function to check the inventory of each product and see which products are running out of stock and need to be restocked`
    - **CheckDailyRevenue**: `Use this function to check the total revenue and profit of the date specified by the user`
- We have defined the **parameters** for each function:
    - **CheckOutOfStock**: `None` (this function does not require any parameters)
    - **CheckDailyRevenue**: `date` (this function requires a `date` parameter)

Okay here we go, let's update the assistant with the function definitions:
```python
assistant_updated = client.beta.assistants.update(
    assistant_id=my_assistant.id, 
    tools=function_definitions
)
```
Congratulations! You have created your assistant with the functions. If you navigate to the assistant page in the OpenAI platform at [https://platform.openai.com/assistants/{assistant_id}](https://platform.openai.com/assistants/{assistant_id}), you will see the assistant with the functions you just defined.

![Coffee Assistant](assets/img/2024-11-02-assistant-function-call/openai_assistant_coffee.png)

## Your first chat with the Assistant

Navigate to your assistant page and click on the icon **Playground ↗** to interact with the assistant.

![Click on the Playground icon](assets/img/2024-11-02-assistant-function-call/playground_click.png)

Now I have just tried to ask the assistant to give me the total revenue of Nov 1st, 2024. 
![First chat with Assistant](assets/img/2024-11-02-assistant-function-call/playground_first_chat.png)

As seen in the screenshot, the assistant has called the `CheckDailyRevenue` function with the date `2024-11-01` as the argument. Therefore, you know the assistant has understood the function call and the argument. You have successfully set up your first multi-function assistant.

## What's next?

In the next post, I will guide you to build **a simple chatting application** to interact with the assistant. Currently, the assistant is only available on the OpenAI platform. In the real world, no one go there to chat with the assistant, so an application is necessary for the assistant to be useful.

Stay tuned!
