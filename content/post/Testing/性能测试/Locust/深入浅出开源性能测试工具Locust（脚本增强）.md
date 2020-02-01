---
title: 深入浅出开源性能测试工具 Locust（脚本增强）
url: /post/head-first-locust-advanced-script
date: 2017-02-22
categories:
  - Testing
tags:
  - 性能测试
  - Locust
---

在[《深入浅出开源性能测试工具Locust（使用篇）》](/post/head-first-locust-user-guide/)一文中，罗列了编写性能测试脚本时常用的几类脚本增强的场景，本文是对应的代码示例。

## 关联

在某些请求中，需要携带之前从Server端返回的参数，因此在构造请求时需要先从之前的Response中提取出所需的参数。

```python
from lxml import etree
from locust import TaskSet, task, HttpLocust

class UserBehavior(TaskSet):

    @staticmethod
    def get_session(html):
        tree = etree.HTML(html)
        return tree.xpath("//div[@class='btnbox']/input[@name='session']/@value")[0]

    @task(10)
    def test_login(self):
        html = self.client.get('/login').text
        username = 'user@compay.com'
        password = '123456'
        session = self.get_session(html)
        payload = {
            'username': username,
            'password': password,
            'session': session
        }
        self.client.post('/login', data=payload)

class WebsiteUser(HttpLocust):
    host = 'https://debugtalk.com'
    task_set = UserBehavior
    min_wait = 1000
    max_wait = 3000
```

## 参数化

### 循环取数据，数据可重复使用

> 所有并发虚拟用户共享同一份测试数据，各虚拟用户在数据列表中循环取值。
> 例如，模拟3用户并发请求网页，总共有100个URL地址，每个虚拟用户都会依次循环加载这100个URL地址；加载示例如下表所示。

| \ | vuser1 | vuser2 | vuser3 |
| --- | --- | --- | --- |
| iteration1 | url1 | url1 | url1 |
| iteration2 | url2 | url2 | url2 |
| ... | ... | ... | ... |
| iteration100 | url100 | url100 | url100 |

```python
from locust import TaskSet, task, HttpLocust

class UserBehavior(TaskSet):
    def on_start(self):
        self.index = 0

    @task
    def test_visit(self):
        url = self.locust.share_data[self.index]
        print('visit url: %s' % url)
        self.index = (self.index + 1) % len(self.locust.share_data)
        self.client.get(url)

class WebsiteUser(HttpLocust):
    host = 'https://debugtalk.com'
    task_set = UserBehavior
    share_data = ['url1', 'url2', 'url3', 'url4', 'url5']
    min_wait = 1000
    max_wait = 3000
```

### 保证并发测试数据唯一性，不循环取数据

> 所有并发虚拟用户共享同一份测试数据，并且保证虚拟用户使用的数据不重复。
> 例如，模拟3用户并发注册账号，总共有9个账号，要求注册账号不重复，注册完毕后结束测试；加载示例如下表所示。

| \ | vuser1 | vuser2 | vuser3 |
| --- | --- | --- | --- |
| iteration1 | account1 | account2 | account3 |
| iteration2 | account4 | account6 | account5 |
| iteration3 | account7 | account9 | account8 |
| N/A | - | — | -

```python
from locust import TaskSet, task, HttpLocust
import queue

class UserBehavior(TaskSet):

    @task
    def test_register(self):
        try:
            data = self.locust.user_data_queue.get()
        except queue.Empty:
            print('account data run out, test ended.')
            exit(0)

        print('register with user: {}, pwd: {}'\
            .format(data['username'], data['password']))
        payload = {
            'username': data['username'],
            'password': data['password']
        }
        self.client.post('/register', data=payload)

class WebsiteUser(HttpLocust):
    host = 'https://debugtalk.com'
    task_set = UserBehavior

    user_data_queue = queue.Queue()
    for index in range(100):
        data = {
            "username": "test%04d" % index,
            "password": "pwd%04d" % index,
            "email": "test%04d@debugtalk.test" % index,
            "phone": "186%08d" % index,
        }
        user_data_queue.put_nowait(data)

    min_wait = 1000
    max_wait = 3000
```

### 保证并发测试数据唯一性，循环取数据

> 所有并发虚拟用户共享同一份测试数据，保证并发虚拟用户使用的数据不重复，并且数据可循环重复使用。
> 例如，模拟3用户并发登录账号，总共有9个账号，要求并发登录账号不相同，但数据可循环使用；加载示例如下表所示。

| \ | vuser1 | vuser2 | vuser3 |
| --- | --- | --- | --- |
| iteration1 | account1 | account2 | account3 |
| iteration2 | account4 | account6 | account5 |
| iteration3 | account7 | account9 | account8 |
| iteration4 | account1 | account2 | account3 |
| iteration5 | account4 | account5 | account6 |
| ... | ... | ... | ... |

该种场景的实现方式与上一种场景基本相同，唯一的差异在于，每次使用完数据后，需要再将数据放入队列中。

```python
from locust import TaskSet, task, HttpLocust
import queue

class UserBehavior(TaskSet):

    @task
    def test_register(self):
        try:
            data = self.locust.user_data_queue.get()
        except queue.Empty:
            print('account data run out, test ended.')
            exit(0)

        print('register with user: {}, pwd: {}'\
            .format(data['username'], data['password']))
        payload = {
            'username': data['username'],
            'password': data['password']
        }
        self.client.post('/register', data=payload)
        self.locust.user_data_queue.put_nowait(data)

class WebsiteUser(HttpLocust):
    host = 'https://debugtalk.com'
    task_set = UserBehavior

    user_data_queue = queue.Queue()
    for index in range(100):
        data = {
            "username": "test%04d" % index,
            "password": "pwd%04d" % index,
            "email": "test%04d@debugtalk.test" % index,
            "phone": "186%08d" % index,
        }
        user_data_queue.put_nowait(data)

    min_wait = 1000
    max_wait = 3000
```
