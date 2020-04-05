---
title: 利用Python实现httpServer
tags: python
categories: suricata
---


* TOC
{:toc}
---------------
## 背景介绍
  
项目需要编写一个中间件，对面提供Http接口，首先肯定想到的就是对 基于python3的简易http-server进行改造。
但是项目实际过程中，中间件的调用可能来自多方，多个用户都会对这个接口进行异步的调用，所以该接口就需要对异步的调用进行及时的处理。

ps： 直接运行的话可以 ```python3 -m http.server```

## 框架选择

首先可以想到的是http-server框架与flask框架。
- 首先联想到cuckoo-agent模块，[agent.py](https://github.com/cuckoosandbox/cuckoo/blob/master/cuckoo/data/agent/agent.py)。 他是基于python2的SimpleHTTPServer实现的。对其并发性能没有深入研究，不过该agent主要部署在虚拟机中，同时也只会处理一个任务。
- http-server比较简单，而flask则是python实现web服务器的通用框架。不管http-server、flask都是一个轻量级别的服务。
- 对于并发请求http-server，它可以有多个连接，但是同时只能处理一条请求任务。该方式就一定会产生阻塞。一开始想的是利用 超时函数，超过固定的秒数就直接返回，但是该方式在多用户的情况下，不是一个很好的选择。
- 对于flask 实现http服务，很显然更快捷，但是你会发现一些问题，就是你同一条指令、同一浏览器重复请求，会出现阻塞情况。尽管使用了多线程。
可以参考该连接：[python flask如何解决同时请求同一个请求的阻塞问题？](https://segmentfault.com/q/1010000012398202)。当然也有解决方法，不过先入为主了。

因此，问题就变为了，如何处理http-server的并发问题：
后来也找到了解决方式 就是使用ThreadingMixIn，参考[python3 HttpServer 实现多线程](https://blog.csdn.net/legend_x/article/details/14127675?depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-2&utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-2)

### http-server 基于flask demo

```python

#!/usr/bin/env python
# -*- coding: utf-8 -*-
# 参考： https://www.cnblogs.com/liangqihui/p/9139270.html
from flask import Flask, abort, request, jsonify

app = Flask(__name__)

# 测试数据暂时存放
tasks = []

@app.route('/add_task/', methods=['POST'])
def add_task():
    if not request.json or 'id' not in request.json or 'info' not in request.json:
        abort(400)
    task = {
        'id': request.json['id'],
        'info': request.json['info']
    }
    tasks.append(task)
    return jsonify({'result': 'success'})


@app.route('/get_task/', methods=['GET'])
def get_task():
    if not request.args or 'id' not in request.args:
        # 没有指定id则返回全部
        return jsonify(tasks)
    else:
        task_id = request.args['id']
        task = filter(lambda t: t['id'] == int(task_id), tasks)
        return jsonify(task) if task else jsonify({'result': 'not found'})


if __name__ == "__main__":
    # 将host设置为0.0.0.0，则外网用户也可以访问到这个服务
    app.run(host="0.0.0.0", port=8383, debug=True)

```

### http-server 基于http.server demo

这边用到了一个装饰器，用于判断函数运行超时，返回信息。
```python
from http.server import HTTPServer, BaseHTTPRequestHandler
import json
import subprocess
import os
import time
from threading import Timer
import getpass
import logging
import os.path
import sys
import signal
import codecs
# from signal import signal, SIGPIPE, SIG_DFL


sys.stdout = codecs.getwriter("utf-8")(sys.stdout.detach())

data = {'result': 'this is a agent!!'}
# 监听主机 & 端口
host = ('0.0.0.0', 4322)
# gvm指令请求超时时间，单位秒
gvm_order_timeout = 6


class Node():
    def __init__(self):
        self.connect_method = ""  # 连接方法
        self.host_name = "127.0.0.1"  # 主机名如：127.0.0.1


def timeout_exception():
    return (500, "run_gvm deal timeout")


def set_timeout(num, callback):
    def wrape(func):
        def handle(signum, frame):
            raise ("运行超时")

        def toDo(*args, **kwargs):
            try:
                signal.signal(signal.SIGALRM, handle)
                signal.alarm(num)  # 开启闹钟信号
                rs = func(*args, **kwargs)
                signal.alarm(0)  # 关闭闹钟信号
                return rs
            except:
                callback()

        return toDo

    return wrape


class Resquest(BaseHTTPRequestHandler):
    timeout = 10
    hostname = '127.0.0.1'
    port = 9390
    username = ''
    password = ''

    def handler(self, code, response):
        self.send_response(code)
        self.send_header('Content-type', 'application/txt')
        self.end_headers()
        self.wfile.write(response.encode())

    def timeout_callback(self, p):
        print('exe time out call back')
        print(p.id)
        try:
            p.kill()
        except Exception as error:
            print(error)

    def run_cmd(self, cmd):
        print('running:%s' % cmd)
        p = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE, cwd=os.getcwd(), shell=True)
        my_timer = Timer(self.timeout, self.timeout_callback, [p])
        my_timer.start()
        try:
            print("start to count timeout; timeout set to be %d \n" % (self.timeout,))
            out, stderr = p.communicate()
            # exit_code = p.returncode
        finally:
            my_timer.cancel()
            return out.decode()

    @set_timeout(gvm_order_timeout, timeout_exception)
    def run_app(self, node):
        return 200, "Parameters are missing"

    def do_GET(self):
        print(self.requestline)
        if self.path != '/hello':
            self.handler(404, "Parameters are missing")
            return
        self.send_response(200)
        self.send_header('Content-type', 'application/json')
        self.end_headers()
        self.wfile.write(json.dumps(data).encode())

    def do_POST(self):
        # print(self.headers)
        # print(self.command)
        # print(self.path, len(self.path), type(self.path))
        if self.headers['Referer'] != 'localHost':
            self.handler(404, "Parameters are missing")
            return
        if self.path == '/':
            self.path = '/tls'
        if self.path != '/tls' and self.path != '/ssh' and self.path != '/sock':
            self.handler(405, "Request method error")
            return
        # 读取请求体内容
        req_datas = self.rfile.read(int(self.headers['content-length']))
        # print(req_datas.decode())
        node = Node()
        node.gvm_cmd = req_datas.decode()
        node.host_name = self.headers['host_name']

        code, data = self.run_app(node)
        data_len = len(data)
        if data_len == 0 or data == "":
            self.handler(405, "response is null")
            return

        self.send_response(code)
        self.send_header('Content-type', 'application/xml')
        self.end_headers()
        # 返回信息量大 需分批写
        offset = 0
        n = data_len % 1024
        m = int(data_len / 1024)
        # print(data_len, m , n)
        for i in range(m):
            data_tmp = data[offset: offset + 1024]
            offset = offset + 1024
            self.wfile.write(data_tmp.encode())
        data_tmp = data[offset:]
        self.wfile.write(data_tmp.encode())


# 可以多线程，多队列的方式，
if __name__ == '__main__':
    server = HTTPServer(host, Resquest)
    server.timeout = 10
    print("Starting server, listen at: %s:%s" % host)
    # server.serve_forever()
    while True:
        server.handle_request()
```
### http-server 基于ThreadingMixIn实现多线程处理：
```python

from http.server import HTTPServer, BaseHTTPRequestHandler
import json
import subprocess
import os
import time
from threading import Timer
import getpass
import logging
import os.path
import sys
import signal
import codecs
from socketserver import ThreadingMixIn
# from signal import signal, SIGPIPE, SIG_DFL


sys.stdout = codecs.getwriter("utf-8")(sys.stdout.detach())

data = {'result': 'this is a agent!!'}
# 监听主机 & 端口
host = ('0.0.0.0', 4322)
# gvm指令请求超时时间，单位秒
gvm_order_timeout = 6


class Node():
    def __init__(self):
        self.connect_method = ""  # 连接方法
        self.host_name = "127.0.0.1"  # 主机名如：127.0.0.1


def timeout_exception():
    return (500, "run_gvm deal timeout")


def set_timeout(num, callback):
    def wrape(func):
        def handle(signum, frame):
            raise ("运行超时")

        def toDo(*args, **kwargs):
            try:
                signal.signal(signal.SIGALRM, handle)
                signal.alarm(num)  # 开启闹钟信号
                rs = func(*args, **kwargs)
                signal.alarm(0)  # 关闭闹钟信号
                return rs
            except:
                callback()

        return toDo

    return wrape


class Resquest(BaseHTTPRequestHandler):
    timeout = 10
    hostname = '127.0.0.1'
    port = 9390
    username = ''
    password = ''

    def handler(self, code, response):
        self.send_response(code)
        self.send_header('Content-type', 'application/txt')
        self.end_headers()
        self.wfile.write(response.encode())

    def timeout_callback(self, p):
        print('exe time out call back')
        print(p.id)
        try:
            p.kill()
        except Exception as error:
            print(error)

    def run_cmd(self, cmd):
        print('running:%s' % cmd)
        p = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE, cwd=os.getcwd(), shell=True)
        my_timer = Timer(self.timeout, self.timeout_callback, [p])
        my_timer.start()
        try:
            print("start to count timeout; timeout set to be %d \n" % (self.timeout,))
            out, stderr = p.communicate()
            # exit_code = p.returncode
        finally:
            my_timer.cancel()
            return out.decode()

    def run_app(self, node):
        print("demo1111")
        time.sleep(10)
        return 200, "Parameters are missing"

    def do_GET(self):
        print(self.requestline)
        if self.path != '/hello':
            self.handler(404, "Parameters are missing")
            return
        self.send_response(200)
        self.send_header('Content-type', 'application/json')
        self.end_headers()
        self.wfile.write(json.dumps(data).encode())

    def do_POST(self):
        # print(self.headers)
        # print(self.command)
        # print(self.path, len(self.path), type(self.path))
        if self.headers['Referer'] != 'localHost':
            self.handler(404, "Parameters are missing")
            return
        if self.path == '/':
            self.path = '/tls'
        if self.path != '/tls' and self.path != '/ssh' and self.path != '/sock':
            self.handler(405, "Request method error")
            return
        # 读取请求体内容
        req_datas = self.rfile.read(int(self.headers['content-length']))
        # print(req_datas.decode())
        node = Node()
        node.gvm_cmd = req_datas.decode()
        node.host_name = self.headers['host_name']

        code, data = self.run_app(node)
        data_len = len(data)
        if data_len == 0 or data == "":
            self.handler(405, "response is null")
            return

        self.send_response(code)
        self.send_header('Content-type', 'application/xml')
        self.end_headers()
        # 返回信息量大 需分批写
        offset = 0
        n = data_len % 1024
        m = int(data_len / 1024)
        # print(data_len, m , n)
        for i in range(m):
            data_tmp = data[offset: offset + 1024]
            offset = offset + 1024
            self.wfile.write(data_tmp.encode())
        data_tmp = data[offset:]
        self.wfile.write(data_tmp.encode())

class ThreadingHttpServer(ThreadingMixIn, HTTPServer):
    pass

# 可以多线程，多队列的方式，
if __name__ == '__main__':
    #server = HTTPServer(host, Resquest)
    server = ThreadingHttpServer(host, Resquest)
    server.timeout = 10
    print("Starting server, listen at: %s:%s" % host)
    server.serve_forever()
    
```

## client

```python
	
import  requests

url = 'http://10.25.40.30:4321'
header = {'User-Agent':'Mozilla/5.0 (Windows NT 6.1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.139 Safari/537.36',
          'Accept': 'text / html,application / xml',
          'Referer': 'localHost',
          'Accept-Encoding': 'gzip, deflate',
          'username':'sysadmin',
         'password':'122'}

#data = "<get_version/>"
r = requests.post(url, headers=header, data=data, timeout=100)

```
