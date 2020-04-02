---
title:  python下实现http-server
tags: http-server
categories: httpServer 网络
---

* TOC
{:toc}

### 背景介绍
  
项目需要编写一个中间件，这个中间件就基于python3 的简易http-server进行改造。
直接运行的话可以 ```python3 -m http.server```

### demo演示

#### 1. http-server 基于flask demo

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

#### 2. http-server 基于http.server demo

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


### client

```python
	
	import  requests
	
	url = 'http://10.25.40.30:4321'
	header = {'User-Agent':'Mozilla/5.0 (Windows NT 6.1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.139 Safari/537.36',
	          'Accept': 'text / html,application / xml',
	          'Referer': 'localHost',
	          'Accept-Encoding': 'gzip, deflate',
	          'username':'sysadmin',
	         'password':'122121'}

	#data = "<get_version/>"
	r = requests.post(url, headers=header, data=data, timeout=100)

```