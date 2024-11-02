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
![An AI assistant helping with various tasks like scheduling, note-taking, and business management.](../assets/img/2024-11-02-assistant-function-call/assistant_img1_1102.jpg "generated with DALL·E 3")

Another scenario when I have my own online shop of coffee, I really need an assistant to help me manage my online shop. For example, at the end of the day, I need her to:
- Check my inventory and tell me to restock the items which are running out of stock
- Send me the total revenue and profit of the day
- Tell me the number of orders I have received during the day/ week/ month...
- Give me some analysis on those figures and maybe some suggestions to improve my business
- ...

![An AI Assistant help analysing the business performance.](../assets/img/2024-11-02-assistant-function-call/assistant_coffee_shop_seller.png "generated with DALL·E 3")

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



