---
layout: post
author: r3tree
title: "encrypt-labs靶场练习，前端加解密配合burp插件：JsRpc、Galaxy"
date: 2024-11-19
music-id: 
permalink: /archives/2024-11-19/1
description: "encrypt-labs靶场练习"
categories: "网络安全"
tags: [encrypt-labs, JsRpc, Galaxy]
---

## JsRpc

首先先熟悉下JsRpc怎么用的。

```python
cmd运行 JsRpc.exe
浏览器控制台输入 JsEnv_Dev.js 的文件内容
浏览器控制台输入 var demo = new Hlclient("ws://127.0.0.1:12080/ws?group=zzz"); 

执行 main.py 控制台可以看到输出test表示成功
也可以浏览器访问http://127.0.0.1:12080/go?group=zzz
```
另外自己安装burp插件Galaxy：https://github.com/outlaws-bai/Galaxy  
靶场地址：https://github.com/SwagXz/encrypt-labs  
在线靶场：http://82.156.57.228:43899/easy.php  

自己没有系统性学习过加解密，靶场是无混淆版，自己写的代码也非常烂。

### 第一关 AES固定KEY

参考第二关

### 第二关 AES服务端获取Key

```python
# coding: utf-8
# @description: 对应示例中的 AesCbc
import json
import base64
from fastapi import FastAPI
from typing import Dict, List
from Crypto.Cipher import AES
from pydantic import BaseModel
from Crypto.Util.Padding import pad, unpad

get_encrypt_text = lambda x: json.loads(x)["encryptedData"]
to_encrypt_body = lambda x: json.dumps({"encryptedData": x.decode()}).encode()

KEY = base64.b64decode(b"lhZ6bTFT04qXLhzkpNWeFg==")
IV = base64.b64decode(b"Oh1U8xTZ097ePh0wVmnALg==")


# AES加密函数
def aes_encrypt(data: bytes) -> bytes:
    cipher = AES.new(KEY, AES.MODE_CBC, IV)
    ct_bytes = cipher.encrypt(pad(data, AES.block_size))
    return base64.b64encode(ct_bytes)


# AES解密函数
def aes_decrypt(data: bytes) -> bytes:
    data1 = base64.b64decode(data)
    cipher = AES.new(KEY, AES.MODE_CBC, IV)
    pt = unpad(cipher.decrypt(data1), AES.block_size)
    return pt


class RequestModel(BaseModel):
    secure: bool
    host: str
    port: int
    version: str
    method: str
    path: str
    query: Dict[str, List[str]]
    headers: Dict[str, List[str]]
    contentBase64: str

    def get_content(self) -> bytes:
        return base64.b64decode(self.contentBase64)

    def set_content(self, content: bytes):
        self.contentBase64 = base64.b64encode(content).decode()


class ResponseModel(BaseModel):
    version: str
    statusCode: int
    reason: str
    headers: Dict[str, List[str]]
    contentBase64: str

    def get_content(self) -> bytes:
        return base64.b64decode(self.contentBase64)

    def set_content(self, content: bytes):
        self.contentBase64 = base64.b64encode(content).decode()


app = FastAPI()


@app.post("/hookRequestToBurp", response_model=RequestModel)
async def hookRequestToBurp(request: RequestModel):
    """HTTP请求从客户端到达Burp时被调用。在此处完成请求解密的代码就可以在Burp中看到明文的请求报文。"""
    request.set_content(aes_decrypt(get_encrypt_text(request.get_content())))
    return request


@app.post("/hookRequestToServer", response_model=RequestModel)
async def hookRequestToServer(request: RequestModel):
    """HTTP请求从Burp将要发送到Server时被调用。在此处完成请求加密的代码就可以将加密后的请求报文发送到Server。"""
    request.set_content(to_encrypt_body(aes_encrypt(request.get_content())))
    return request


@app.post("/hookResponseToBurp", response_model=ResponseModel)
async def hookResponseToBurp(response: ResponseModel):
    """HTTP响应从Server到达Burp时被调用。在此处完成响应解密的代码就可以在Burp中看到明文的响应报文。"""
    response.set_content(aes_decrypt(get_encrypt_text(response.get_content())))
    return response


@app.post("/hookResponseToClient", response_model=ResponseModel)
async def hookResponseToClient(response: ResponseModel):
    """HTTP响应从Burp将要发送到Client时被调用。在此处完成响应加密的代码就可以将加密后的响应报文返回给Client。"""
    response.set_content(to_encrypt_body(aes_encrypt(response.get_content())))
    return response


if __name__ == "__main__":
    # 多进程启动
    # uvicorn manager:app --host 0.0.0.0 --port 5000 --workers 4
    import uvicorn

    uvicorn.run(app, host="0.0.0.0", port=5000)
```

### 第三关 Rsa加密 无私钥版 JsRpc+Galaxy

浏览器控制台输入

```python
window.en=encryptor

demo.regAction("hello",function (resolve,param) {
    pKey="-----BEGIN PUBLIC KEY-----\nMIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDRvA7giwinEkaTYllDYCkzujvi\nNH+up0XAKXQot8RixKGpB7nr8AdidEvuo+wVCxZwDK3hlcRGrrqt0Gxqwc11btlM\nDSj92Mr3xSaJcshZU8kfj325L8DRh9jpruphHBfh955ihvbednGAvOHOrz3Qy3Cb\nocDbsNeCwNpRxwjIdQIDAQAB\n-----END PUBLIC KEY-----"
    en.setPublicKey(pKey)
    var data1 = {"username":"","password":""} 
    data1["username"] = param["username"]
    data1["password"] = param["password"]
    data = en.encrypt(JSON.stringify(data1))
    resolve(data);
})
```
访问测试http://127.0.0.1:12080/go?group=zzz&action=hello&param={"username":"admin","password":"123456"}

Edited request直接切换为明文，再Repeater明文即可，可以用于明文爆破

```python
# coding: utf-8
# Edited request直接切换为明文，再Repeater明文即可，可以用于明文爆破
import json
import base64
import requests
import urllib.parse
from fastapi import FastAPI
from typing import Dict, List
from pydantic import BaseModel
from Crypto.Util.Padding import pad, unpad

# 解密函数
def decrypt(data) -> bytes:
    # 判断是否能解析JSON，不能则是加密，Edited request功能时默认{"username":"admin","password":"12345"}
    try:
        json_data = json.loads(data)  # 尝试解析为 JSON
    except (json.JSONDecodeError, UnicodeDecodeError):
        data='{"username":"admin","password":"12345"}'
        json_data = json.loads(data)
    # 假设我们只取 JSON 数据中的 "password" 字段进行加密
    username = json_data.get("username", "")
    password = json_data.get("password", "")
    data = {"username":username,"password":password}
    return json.dumps(data).encode('utf-8')

# 加密函数
def encrypt(data) -> bytes:
    # 判断是否能解析JSON，不能则用固定发包{"username":"admin","password":"12345"}
    try:
        json_data = json.loads(data)  # 尝试解析为 JSON
    except (json.JSONDecodeError, UnicodeDecodeError):
        data='{"username":"admin","password":"12345"}'
        json_data = json.loads(data)
    # 假设我们只取 JSON 数据中的 "password" 字段进行加密
    username = json_data.get("username", "")
    password = json_data.get("password", "")
    
    response1 = requests.get('http://127.0.0.1:12080/go?group=zzz&action=hello&param={"username":"'+username+'","password":"'+password+'"}')
    json_data1 = response1.json()
    json_data2 = json_data1["data"]
    json_data2 = {"data": json_data2}
    json_data2=urllib.parse.urlencode(json_data2)
    return json_data2.encode('utf-8')

class RequestModel(BaseModel):
    secure: bool
    host: str
    port: int
    version: str
    method: str
    path: str
    query: Dict[str, List[str]]
    headers: Dict[str, List[str]]
    contentBase64: str

    def get_content(self) -> bytes:
        return base64.b64decode(self.contentBase64)

    def set_content(self, content: bytes):
        self.contentBase64 = base64.b64encode(content).decode()


class ResponseModel(BaseModel):
    version: str
    statusCode: int
    reason: str
    headers: Dict[str, List[str]]
    contentBase64: str

    def get_content(self) -> bytes:
        return base64.b64decode(self.contentBase64)

    def set_content(self, content: bytes):
        self.contentBase64 = base64.b64encode(content).decode()

app = FastAPI()

@app.post("/hookRequestToBurp", response_model=RequestModel)
async def hookRequestToBurp(request: RequestModel):
    """HTTP请求从客户端到达Burp时被调用。在此处完成请求解密的代码就可以在Burp中看到明文的请求报文。"""
    request.set_content(decrypt(request.get_content()))
    return request

@app.post("/hookRequestToServer", response_model=RequestModel)
async def hookRequestToServer(request: RequestModel):
    """HTTP请求从Burp将要发送到Server时被调用。在此处完成请求加密的代码就可以将加密后的请求报文发送到Server。"""
    request.set_content(encrypt(request.get_content()))
    return request

@app.post("/hookResponseToBurp", response_model=ResponseModel)
async def hookResponseToBurp(response: ResponseModel):
    """HTTP响应从Server到达Burp时被调用。在此处完成响应解密的代码就可以在Burp中看到明文的响应报文。"""
    #response.set_content(des_decrypt(get_encrypt_text(response.get_content())))
    return response


@app.post("/hookResponseToClient", response_model=ResponseModel)
async def hookResponseToClient(response: ResponseModel):
    """HTTP响应从Burp将要发送到Client时被调用。在此处完成响应加密的代码就可以将加密后的响应报文返回给Client。"""
    #response.set_content(to_encrypt_body(des_encrypt(response.get_content())))
    return response

if __name__ == "__main__":
    # 多进程启动
    # uvicorn manager:app --host 0.0.0.0 --port 5000 --workers 4
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=5000)
```

### 第四关 AES+Rsa加密 无私钥版 JsRpc+Galaxy

浏览器控制台输入

```python
window.keyy=CryptoJS.lib.WordArray
window.ivv=CryptoJS.lib.WordArray
window.aess=CryptoJS.AES.encrypt
window.modeCBC=CryptoJS.mode.CBC
window.padPkcs7=CryptoJS.pad.Pkcs7
window.RSAA=rsa
window.Base644=CryptoJS.enc.Base64

demo.regAction("hello",function (resolve,param) {
    var keyyy=keyy.random(16)
    var ivvv=ivv.random(16)
    var jsonn=JSON.stringify(param)
    RSAA.setPublicKey("-----BEGIN PUBLIC KEY-----\nMIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDRvA7giwinEkaTYllDYCkzujvi\nNH+up0XAKXQot8RixKGpB7nr8AdidEvuo+wVCxZwDK3hlcRGrrqt0Gxqwc11btlM\nDSj92Mr3xSaJcshZU8kfj325L8DRh9jpruphHBfh955ihvbednGAvOHOrz3Qy3Cb\nocDbsNeCwNpRxwjIdQIDAQAB\n-----END PUBLIC KEY-----")
    var encryptedKeyy=RSAA.encrypt(keyyy.toString(Base644))
    var encryptedIvv=RSAA.encrypt(ivvv.toString(Base644))
    var encryptedDataa=aess(jsonn, keyyy, {
			iv: ivvv,
			mode: modeCBC,
			padding: padPkcs7
		})
		.toString();
    var data1 = {"encryptedData":"","encryptedKey":"","encryptedIv":""} 
    data1["encryptedData"] = encryptedDataa
    data1["encryptedKey"] = encryptedKeyy
    data1["encryptedIv"] = encryptedIvv
    data = JSON.stringify(data1)
    resolve(data);
})
```
```python
访问测试http://127.0.0.1:12080/go?group=zzz&action=hello&param={"username":"admin","password":"123456"}
# coding: utf-8
import json
import base64
import requests
import urllib.parse
from fastapi import FastAPI
from typing import Dict, List
from pydantic import BaseModel
from Crypto.Util.Padding import pad, unpad


# 解密函数
def decrypt(data) -> bytes:
    # 将 JSON 字符串解析为 Python 字典
    json_data = json.loads(data)
    if json_data.get("encryptedData", "")!="":
        data='{"username":"admin","password":"12345"}'
        json_data = json.loads(data)
    # 假设我们只取 JSON 数据中的 "password" 字段进行加密
    username = json_data.get("username", "")
    password = json_data.get("password", "")
    data = {"username":username,"password":password}
    return json.dumps(data).encode('utf-8')

# 加密函数
def encrypt(data) -> bytes:
    # 将 JSON 字符串解析为 Python 字典
    # 尝试解析为 JSON
    json_data = json.loads(data)
    if json_data.get("encryptedData", "")!="":
        data='{"username":"admin","password":"12345"}'
        json_data = json.loads(data)
    # 假设我们只取 JSON 数据中的 "password" 字段进行加密
    username = json_data.get("username", "")
    password = json_data.get("password", "")
    
    response1 = requests.get('http://127.0.0.1:12080/go?group=zzz&action=hello&param={"username":"'+username+'","password":"'+password+'"}')
    json_data1 = response1.json()
    json_data2 = json_data1["data"]
    return json_data2.encode('utf-8')

class RequestModel(BaseModel):
    secure: bool
    host: str
    port: int
    version: str
    method: str
    path: str
    query: Dict[str, List[str]]
    headers: Dict[str, List[str]]
    contentBase64: str

    def get_content(self) -> bytes:
        return base64.b64decode(self.contentBase64)

    def set_content(self, content: bytes):
        self.contentBase64 = base64.b64encode(content).decode()


class ResponseModel(BaseModel):
    version: str
    statusCode: int
    reason: str
    headers: Dict[str, List[str]]
    contentBase64: str

    def get_content(self) -> bytes:
        return base64.b64decode(self.contentBase64)

    def set_content(self, content: bytes):
        self.contentBase64 = base64.b64encode(content).decode()


app = FastAPI()


@app.post("/hookRequestToBurp", response_model=RequestModel)
async def hookRequestToBurp(request: RequestModel):
    """HTTP请求从客户端到达Burp时被调用。在此处完成请求解密的代码就可以在Burp中看到明文的请求报文。"""
    request.set_content(decrypt(request.get_content()))
    return request


@app.post("/hookRequestToServer", response_model=RequestModel)
async def hookRequestToServer(request: RequestModel):
    """HTTP请求从Burp将要发送到Server时被调用。在此处完成请求加密的代码就可以将加密后的请求报文发送到Server。"""
    request.set_content(encrypt(request.get_content()))
    return request


@app.post("/hookResponseToBurp", response_model=ResponseModel)
async def hookResponseToBurp(response: ResponseModel):
    """HTTP响应从Server到达Burp时被调用。在此处完成响应解密的代码就可以在Burp中看到明文的响应报文。"""
    #response.set_content(des_decrypt(get_encrypt_text(response.get_content())))
    return response


@app.post("/hookResponseToClient", response_model=ResponseModel)
async def hookResponseToClient(response: ResponseModel):
    """HTTP响应从Burp将要发送到Client时被调用。在此处完成响应加密的代码就可以将加密后的响应报文返回给Client。"""
    #response.set_content(to_encrypt_body(des_encrypt(response.get_content())))
    return response


if __name__ == "__main__":
    # 多进程启动
    # uvicorn manager:app --host 0.0.0.0 --port 5000 --workers 4
    import uvicorn

    uvicorn.run(app, host="0.0.0.0", port=5000)
```

### 第五关 Des规律Key

```python
# coding: utf-8
import json
import base64
# import requests
from fastapi import FastAPI
from typing import Dict, List
from Crypto.Cipher import DES
from pydantic import BaseModel
from Crypto.Util.Padding import pad, unpad

get_encrypt_text = lambda x: json.loads(x)["password"]
to_encrypt_body = lambda x: json.dumps({"username":"admin","password": x}).encode()

#KEY = base64.b64decode(b"lhZ6bTFT04qXLhzkpNWeFg==")
#IV = base64.b64decode(b"Oh1U8xTZ097ePh0wVmnALg==")

def getKEY_IV():
    username = "admin"  # 这里替换为你的用户名
    KEY = (username[:8] + '6' * 8)[:8].encode('utf-8')
    IV =('9999' + username[:4] + '9' * 4)[:8].encode('utf-8')
    print(KEY)
    print(IV)
    return KEY, IV

# DES加密函数
def des_encrypt(data: bytes) -> bytes:
    KEY, IV = getKEY_IV()
    cipher = DES.new(KEY, DES.MODE_CBC, IV)
    ct_bytes = cipher.encrypt(pad(data, DES.block_size))
    return ct_bytes.hex()

# DES解密函数
def des_decrypt(data: bytes) -> bytes:
    KEY, IV = getKEY_IV()
    # 恢复
    encrypted_data = bytes.fromhex(data)
    # 创建 DES 解密器
    cipher = DES.new(KEY, DES.MODE_CBC, IV)
    # 解密并去除填充
    plaintext = unpad(cipher.decrypt(encrypted_data), DES.block_size)
    print(plaintext)
    return plaintext

class RequestModel(BaseModel):
    secure: bool
    host: str
    port: int
    version: str
    method: str
    path: str
    query: Dict[str, List[str]]
    headers: Dict[str, List[str]]
    contentBase64: str

    def get_content(self) -> bytes:
        return base64.b64decode(self.contentBase64)

    def set_content(self, content: bytes):
        print(content)
        self.contentBase64 = base64.b64encode(content).decode()


class ResponseModel(BaseModel):
    version: str
    statusCode: int
    reason: str
    headers: Dict[str, List[str]]
    contentBase64: str

    def get_content(self) -> bytes:
        return base64.b64decode(self.contentBase64)

    def set_content(self, content: bytes):
        self.contentBase64 = base64.b64encode(content).decode()


app = FastAPI()


@app.post("/hookRequestToBurp", response_model=RequestModel)
async def hookRequestToBurp(request: RequestModel):
    """HTTP请求从客户端到达Burp时被调用。在此处完成请求解密的代码就可以在Burp中看到明文的请求报文。"""
    request.set_content(des_decrypt(get_encrypt_text(request.get_content())))
    print(request)
    return request


@app.post("/hookRequestToServer", response_model=RequestModel)
async def hookRequestToServer(request: RequestModel):
    """HTTP请求从Burp将要发送到Server时被调用。在此处完成请求加密的代码就可以将加密后的请求报文发送到Server。"""
    request.set_content(to_encrypt_body(des_encrypt(request.get_content())))
    return request


@app.post("/hookResponseToBurp", response_model=ResponseModel)
async def hookResponseToBurp(response: ResponseModel):
    """HTTP响应从Server到达Burp时被调用。在此处完成响应解密的代码就可以在Burp中看到明文的响应报文。"""
    response.set_content(des_decrypt(get_encrypt_text(response.get_content())))
    return response


@app.post("/hookResponseToClient", response_model=ResponseModel)
async def hookResponseToClient(response: ResponseModel):
    """HTTP响应从Burp将要发送到Client时被调用。在此处完成响应加密的代码就可以将加密后的响应报文返回给Client。"""
    response.set_content(to_encrypt_body(des_encrypt(response.get_content())))
    return response


if __name__ == "__main__":
    # 多进程启动
    # uvicorn manager:app --host 0.0.0.0 --port 5000 --workers 4
    import uvicorn

    uvicorn.run(app, host="0.0.0.0", port=5000)
```

### 第六关 明文加签

```python
# coding: utf-8
import json
import base64
import hashlib
import hmac
import time
import random
import string
from fastapi import FastAPI
from typing import Dict, List
from pydantic import BaseModel
from Crypto.Util.Padding import pad, unpad

# 生成 nonce
nonce = ''.join(random.choices(string.ascii_lowercase + string.digits, k=16))
# 获取当前时间戳（秒）
timestamp = int(time.time())
# 密钥
secret_key = "be56e057f20f883e"

# 解密函数
def decrypt(data) -> bytes:
    # 将 JSON 字符串解析为 Python 字典
    json_data = json.loads(data)
    # 假设我们只取 JSON 数据中的 "password" 字段进行加密
    username = json_data.get("username", "")
    password = json_data.get("password", "")
    data = {"username":username,"password":password}
    return json.dumps(data).encode('utf-8')

# 加密函数
def encrypt(data) -> bytes:
    # 将 JSON 字符串解析为 Python 字典
    json_data = json.loads(data)
    # 假设我们只取 JSON 数据中的 "password" 字段进行加密
    username = json_data.get("username", "")
    password = json_data.get("password", "")
    data_to_sign = username + password + nonce + str(timestamp)
    # HMAC-SHA256 签名
    signature = hmac.new(
        bytes(secret_key, 'utf-8'), 
        bytes(data_to_sign, 'utf-8'), 
        hashlib.sha256
    ).hexdigest()
    data = {"username":username,"password":password,"nonce":nonce,"timestamp":timestamp,"signature": signature}
    return json.dumps(data).encode('utf-8')

class RequestModel(BaseModel):
    secure: bool
    host: str
    port: int
    version: str
    method: str
    path: str
    query: Dict[str, List[str]]
    headers: Dict[str, List[str]]
    contentBase64: str

    def get_content(self) -> bytes:
        return base64.b64decode(self.contentBase64)

    def set_content(self, content: bytes):
        self.contentBase64 = base64.b64encode(content).decode()


class ResponseModel(BaseModel):
    version: str
    statusCode: int
    reason: str
    headers: Dict[str, List[str]]
    contentBase64: str

    def get_content(self) -> bytes:
        return base64.b64decode(self.contentBase64)

    def set_content(self, content: bytes):
        self.contentBase64 = base64.b64encode(content).decode()


app = FastAPI()


@app.post("/hookRequestToBurp", response_model=RequestModel)
async def hookRequestToBurp(request: RequestModel):
    """HTTP请求从客户端到达Burp时被调用。在此处完成请求解密的代码就可以在Burp中看到明文的请求报文。"""
    request.set_content(decrypt(request.get_content()))
    return request


@app.post("/hookRequestToServer", response_model=RequestModel)
async def hookRequestToServer(request: RequestModel):
    """HTTP请求从Burp将要发送到Server时被调用。在此处完成请求加密的代码就可以将加密后的请求报文发送到Server。"""
    request.set_content(encrypt(request.get_content()))
    return request


@app.post("/hookResponseToBurp", response_model=ResponseModel)
async def hookResponseToBurp(response: ResponseModel):
    """HTTP响应从Server到达Burp时被调用。在此处完成响应解密的代码就可以在Burp中看到明文的响应报文。"""
    #response.set_content(des_decrypt(get_encrypt_text(response.get_content())))
    return response


@app.post("/hookResponseToClient", response_model=ResponseModel)
async def hookResponseToClient(response: ResponseModel):
    """HTTP响应从Burp将要发送到Client时被调用。在此处完成响应加密的代码就可以将加密后的响应报文返回给Client。"""
    #response.set_content(to_encrypt_body(des_encrypt(response.get_content())))
    return response


if __name__ == "__main__":
    # 多进程启动
    # uvicorn manager:app --host 0.0.0.0 --port 5000 --workers 4
    import uvicorn

    uvicorn.run(app, host="0.0.0.0", port=5000)
```

### 第七关 加签key在服务器端

```python
# 加密函数
def encrypt(data) -> bytes:
    # 将 JSON 字符串解析为 Python 字典
    json_data = json.loads(data)
    # 假设我们只取 JSON 数据中的 "password" 字段进行加密
    username = json_data.get("username", "")
    password = json_data.get("password", "")
    data1 = {
    "username": username,
    "password": password,
    "timestamp": timestamp
    }
    url = "http://82.156.57.228:43899/encrypt/get-signature.php"  # 这里替换为你实际的 API 地址
    # 发送 POST 请求
    response = requests.post(url, json=data1)
    response_data = response.json()
    signature = response_data.get('signature')
    print(signature)
    data = {"username":username,"password":password,"timestamp":timestamp,"signature": signature}
    return json.dumps(data).encode('utf-8')
```

### 第八关 禁止重放 公钥取random JsRpc+Galaxy

```python
window.rsa=rsaEncrypt

demo.regAction("hello",function (resolve,param) {
    var pKey = "-----BEGIN PUBLIC KEY-----\nMIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDRvA7giwinEkaTYllDYCkzujvi\nNH+up0XAKXQot8RixKGpB7nr8AdidEvuo+wVCxZwDK3hlcRGrrqt0Gxqwc11btlM\nDSj92Mr3xSaJcshZU8kfj325L8DRh9jpruphHBfh955ihvbednGAvOHOrz3Qy3Cb\nocDbsNeCwNpRxwjIdQIDAQAB\n-----END PUBLIC KEY-----"
    var time = Date.parse(new Date);
    random = rsa(time, pKey)
    var data = {"username":"","password":"","random":""} 
    data["username"] = param["username"]
    data["password"] = param["password"]
    data["random"] = random
    resolve(data);
})
```
```python
访问测试http://127.0.0.1:12080/go?group=zzz&action=hello&param={"username":"admin","password":"123456"}
# coding: utf-8
import json
import base64
import requests
from fastapi import FastAPI
from typing import Dict, List
from pydantic import BaseModel
from Crypto.Util.Padding import pad, unpad



# 解密函数
def decrypt(data) -> bytes:
    # 将 JSON 字符串解析为 Python 字典
    json_data = json.loads(data)
    # 假设我们只取 JSON 数据中的 "password" 字段进行加密
    username = json_data.get("username", "")
    password = json_data.get("password", "")
    data = {"username":username,"password":password}
    return json.dumps(data).encode('utf-8')

# 加密函数
def encrypt(data) -> bytes:
    # 将 JSON 字符串解析为 Python 字典
    json_data = json.loads(data)
    # 假设我们只取 JSON 数据中的 "password" 字段进行加密
    username = json_data.get("username", "")
    password = json_data.get("password", "")
    
    response1 = requests.get("http://127.0.0.1:12080/go?group=zzz&action=hello&param={"username":"'+username+'","password":"'+password+'"}')
    json_data1 = response1.json()
    json_data2 = json_data1["data"]
    get_random = json.loads(json_data2)["random"]
    data = {"username":username,"password":password,"random": get_random}
    return json.dumps(data).encode('utf-8')

class RequestModel(BaseModel):
    secure: bool
    host: str
    port: int
    version: str
    method: str
    path: str
    query: Dict[str, List[str]]
    headers: Dict[str, List[str]]
    contentBase64: str

    def get_content(self) -> bytes:
        return base64.b64decode(self.contentBase64)

    def set_content(self, content: bytes):
        self.contentBase64 = base64.b64encode(content).decode()


class ResponseModel(BaseModel):
    version: str
    statusCode: int
    reason: str
    headers: Dict[str, List[str]]
    contentBase64: str

    def get_content(self) -> bytes:
        return base64.b64decode(self.contentBase64)

    def set_content(self, content: bytes):
        self.contentBase64 = base64.b64encode(content).decode()


app = FastAPI()


@app.post("/hookRequestToBurp", response_model=RequestModel)
async def hookRequestToBurp(request: RequestModel):
    """HTTP请求从客户端到达Burp时被调用。在此处完成请求解密的代码就可以在Burp中看到明文的请求报文。"""
    request.set_content(decrypt(request.get_content()))
    return request


@app.post("/hookRequestToServer", response_model=RequestModel)
async def hookRequestToServer(request: RequestModel):
    """HTTP请求从Burp将要发送到Server时被调用。在此处完成请求加密的代码就可以将加密后的请求报文发送到Server。"""
    request.set_content(encrypt(request.get_content()))
    return request


@app.post("/hookResponseToBurp", response_model=ResponseModel)
async def hookResponseToBurp(response: ResponseModel):
    """HTTP响应从Server到达Burp时被调用。在此处完成响应解密的代码就可以在Burp中看到明文的响应报文。"""
    #response.set_content(des_decrypt(get_encrypt_text(response.get_content())))
    return response


@app.post("/hookResponseToClient", response_model=ResponseModel)
async def hookResponseToClient(response: ResponseModel):
    """HTTP响应从Burp将要发送到Client时被调用。在此处完成响应加密的代码就可以将加密后的响应报文返回给Client。"""
    #response.set_content(to_encrypt_body(des_encrypt(response.get_content())))
    return response


if __name__ == "__main__":
    # 多进程启动
    # uvicorn manager:app --host 0.0.0.0 --port 5000 --workers 4
    import uvicorn

    uvicorn.run(app, host="0.0.0.0", port=5000)
```

## Galaxy联动sqlmap

在已解密请求右键找到 `HTTP HOOK -> Send Decrypted Request To Sqlmap`，点击即可

`python3 sqlmap.py -r C:\x\.galaxy\tmp\28ff6666-ca66-40cf-ac1c-29162772da75.txt --risk=3 --level=3 --proxy=http://127.0.0.1:8080`

## Galaxy联动Xray

配置xray的上游代理为 burp`Network -> Connections`

开启 `Auto Scan Decrypted Request` 或 在已解密请求右键找到 `Send Decrypted Request To Scanner` 点击

## YAKIT 热加载

```python
POST /api/user/login HTTP/1.1
Host: 39.98.108.20:8085
Content-Type: application/json;charset=utf-8
X-Requested-With: XMLHttpRequest
requestId: afda9bc9b9d10b4b55b8aedbcc27e29a
Priority: u=0
Sec-GPC: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:132.0) Gecko/20100101 Firefox/132.0
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
timestamp: 1731917319000
Origin: http://39.98.108.20:8085
DNT: 1
Accept: application/json, text/plain, */*
Referer: http://39.98.108.20:8085/
Accept-Encoding: gzip, deflate
sign: cfcfc45b5dd24a8b1c0d83129d752b6e
Content-Length: 88

{"password":"{{payload(pass_top25)}}","username":"test","validCode":"3psj"}
// 定义加密函数
func getEnc(data){
    rsp,rep,err = poc.Post("http://127.0.0.1:12080/go",poc.replaceBody("group=zzz&action=hello&param="+data, false),poc.appendHeader("content-type", "application/x-www-form-urlencoded"))
    if(err){
        return(err)
    }

    return json.loads(rsp.GetBody())["data"]
}

// beforeRequest 允许发送数据包前再做一次处理，定义为 func(origin []byte) []byte
beforeRequest = func(req) {
    //获取请求体
    req_body = poc.GetHTTPPacketBody(req)
    //加密
    res = getEnc(string(req_body))
    //获取其他的参数
    res = json.loads(res)

    //修改其他的请求头
    req = poc.ReplaceHTTPPacketHeader(req, "requestId", res["id"])
    req = poc.ReplaceHTTPPacketHeader(req, "timestamp", res["time"])
    req = poc.ReplaceHTTPPacketHeader(req, "sign", res["sign"])

    //修改请求体
    req = poc.ReplaceHTTPPacketBody(req, res["encstr"])


    return []byte(req)
}

// afterRequest 允许对每一个请求的响应做处理，定义为 func(origin []byte) []byte
afterRequest = func(rsp) {
    return []byte(rsp)
}

// mirrorHTTPFlow 允许对每一个请求的响应做处理，定义为 func(req []byte, rsp []byte, params map[string]any) map[string]any
// 返回值回作为下一个请求的参数，或者提取的数据，如果你需要解密响应内容，在这里操作是最合适的
mirrorHTTPFlow = func(req, rsp, params) {
    return params
}
```

## autoDecoder接口加解密

```python
# -*- coding:utf-8 -*-  
# aes加密后，外面套了一层base64  
# 明文为  
# {"username":"admin","password":"123"}
# 数据包的入参为  
# {"encryptedData":"igI6RB3HXfZhRt4zncoBYJN2tCTZJ9+dqwQL9OTCOOeV6zqDF7+I2SYrIOQa6Ydt"} 
  
from Crypto.Cipher import AES  
import base64,json  
  
from Crypto.Util.Padding import pad  
  
def aes_encrypt(text):  
    password = base64.b64decode("lhZ6bTFT04qXLhzkpNWeFg==") #秘钥，b就是表示为bytes类型  
    iv = base64.b64decode("Oh1U8xTZ097ePh0wVmnALg==")
    text = text.encode() #需要加密的内容，bytes类型  
    aes = AES.new(password,AES.MODE_CBC,iv) #创建一个aes对象  
    # AES.MODE_ECB 表示模式是ECB模式    
    text = pad(text, 16)  
    en_text = aes.encrypt(text) #加密明文  
    print(en_text)
    out = base64.b64encode(en_text)  
    return out.decode() #加密明文，bytes类型  
  
  
def aes_decrypt(text):  
    password = base64.b64decode("lhZ6bTFT04qXLhzkpNWeFg==") #秘钥，b就是表示为bytes类型  
    iv = base64.b64decode("Oh1U8xTZ097ePh0wVmnALg==")
    text = base64.b64decode(text) #需要加密的内容，bytes类型  
    aes = AES.new(password,AES.MODE_CBC,iv) #创建一个aes对象  
    # AES.MODE_ECB 表示模式是ECB模式    
    en_text = aes.decrypt(text) #加密明文  
    return en_text.decode()  
  
  
from flask import Flask,Response,request  
import base64  
app = Flask(__name__)  
  
@app.route('/encode',methods=["POST"])  
def encrypt():  
    body = request.form.get('dataBody')  # 获取  post 参数 必需  
    headers = request.form.get('dataHeaders')  # 获取  post 参数  可选  
    print(body)  
    if headers != None: # 开启了请求头加密  
        headers = headers + "aaaa:bbbb\r\n"  
        headers = headers + "xxx:test"  
        print(headers + "\r\n\r\n\r\n\r\n" + body)  
        return headers + "\r\n\r\n\r\n\r\n" + body # 返回值为固定格式，不可更改  
    body = aes_encrypt(body)  
    body = '{"encryptedData":"' + body + '"}'  
    print(body) 
    return  body  
  
@app.route('/decode',methods=["POST"])  
def decrypt():  
    body = request.form.get('dataBody')  # 获取  post 参数 必需  
    headers = request.form.get('dataHeaders')  # 获取  post 参数  可选  
    print(body)  
    if headers != None: # 开启了响应头加密  
        print(headers + "\r\n\r\n\r\n\r\n" + body)  
        headers = headers + "yyyy:zzzz\r\n"  
        headers = headers + "xxx:onlysecurity"  
        return headers + "\r\n\r\n\r\n\r\n" + body # 返回值为固定格式，不可更改  
  
    if "encryptedData" in body:  
        body = json.loads(body)['encryptedData']  
        body = aes_decrypt(body)
        print(body)  
        return body.strip()  
    else:  
        return body.strip()  
  
# print(aes_encrypt('{"userName":"admin","userPwd":"123456"}'))  
if __name__ == '__main__':  
    app.debug = True # 设置调试模式，生产模式的时候要关掉debug  
    app.run(host="0.0.0.0",port="8888")
```

## jsEncrypter取Des规律key

靶场：[encrypt靶场](http://82.156.57.228:43899/easy.php)第五关 Des规律key

```javascript
function js_encrypt(payload){
  var newpayload;
  /**********在这里编写调用加密函数进行加密的代码************/
  const username = "admin"
  const password = payload
  const key = CryptoJS.enc.Utf8.parse(padEndPolyfill(username.slice(0, 8), 8, '6'));
  const iv = CryptoJS.enc.Utf8.parse('9999' + padEndPolyfill(username.slice(0, 8), 4, '9'));
  const encryptedPassword = CryptoJS.DES.encrypt(password, key, {
    iv: iv,
    mode: CryptoJS.mode.CBC,
    padding: CryptoJS.pad.Pkcs7
  });

  const encryptedHex = encryptedPassword.ciphertext.toString(CryptoJS.enc.Hex);
  newpayload = encryptedHex
  /**********************************************************/
  return newpayload;
}

/**********自己写的padEndPolyfill函数************/
function padEndPolyfill(str, targetLength, padString) {
  targetLength = targetLength >> 0; // 转为整数
  padString = typeof padString === 'string' ? padString : ' '; // 确保是字符串
  if (str.length > targetLength) {
    return String(str);
  }
  targetLength = targetLength - str.length;
  if (padString.length > 0) {
    while (padString.length < targetLength) {
      padString += padString; // 自行扩展字符串长度
    }
  }
  return String(str) + padString.slice(0, targetLength);
}
```