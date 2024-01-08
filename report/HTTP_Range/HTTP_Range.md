# HTTP断点续传技术及其在minio中的实践
## 概述
断点续传技术主要用于处理网络传输过程中的中断问题，允许传输任务在中断后从上次停止的地方继续，而不是重新开始。
比较代表的案例是：主流视频网站均采用断点续传，用户可以从上次观看位置获取视频文件片段而无需下载整个视频。从而节省大量的网络带宽和服务器资源。
## 支持断点续传的协议
- **HTTP/HTTPS:** 最常见的协议，用于网页内容的传输。支持断点续传的关键特性是HTTP头部中的`Range`请求和`206 Partial Content`响应。
- FTP: 文件传输协议，早期用于文件下载，也支持断点续传。
- BitTorrent: 一种点对点文件共享协议，天然支持断点续传，因为它从多个源下载文件的不同部分。（迅雷等P2P技术）
## 简要过程
服务端对大型文件进行切片，如1000字节的文件每100个字节切成10个片段，客户端请求时通过设置HTTP头部的`Range`参数规定要索取的字节范围(0,99),(100,199),(200,299)...(900,999)
10个片段下载完成后，客户端对片段进行合并，将10个片段文件合成一个完整文件。（这个过程需要做文件检查，补充下载缺失片段）
## 断点续传实现步骤
### 前置条件,协议支持
HTTP/HTTPS:从HTTP1.1开始支持，断点续传的关键特性是利用HTTP头部中的`Range : bytes = 0-100`和`Response Status= 206 Partial Conten` 参数。
### 步骤
1. 服务端对大型文件先进行长度length，然后进行切片（片段大小需要考虑，过大的片段会导致断点续传重复下载的数据量，过小的片段增加服务端资源消耗，即增加服务端连接次数和网络带宽并发量）
2. 客户端发送带有范围的请求：客户端在继续传输时，发送一个带有Range头字段的HTTP请求。该字段指定了客户端希望获取的文件范围，例如`Range: bytes=100-199`，表示从字节流文件中获取100-199范围内字节。
3. 服务器处理范围请求：服务器接收到带有范围的请求后，需要解析范围参数并返回相应的文件内容。服务器可以使用文件操作函数来定位到指定的字节索引，并从该位置开始读取文件内容。
### 至此，接下来两种实现思路：
1. 文件追加，边下载边合并 
   - 客户端接收部分文件内容：客户端接收到服务器返回的部分文件内容后，将其写入到本地文件的相应位置。如果之前已经下载了一部分文件，客户端需要将新获取的文件内容追加到已有文件的末尾。 
   - 重复下载所有片段直到完成：客户端可以根据需要多次发送范围请求，直到获取完整的文件内容。客户端需要跟踪已经下载的文件部分，并在每次请求时更新范围参数。
2. 并发下载，最后进行文件合并 
3. 所有片段下载完成后，按片段的顺序对文件进行合并。
### 两种思路比较：
文件追加的方法适合于视频流，文件顺序下载一边下载一边被用户使用。但是增加了文件合并的复杂性，追加时对于多个片段与以字节顺序对文件进行合并。  
最后合并的思路适合非流媒体，同时合并直接以片段顺序进行合并，而无需记录字节的顺序。大大减少客户端合并逻辑。
### 结论：
就我们的业务场景我更倾向于后者
## 现代浏览器及断点续传
现代浏览器如Chrome, Firefox等，内置了对HTTP断点续传的支持，可以顺利下载片段。
### 但是:
处理合并的逻辑过程中，由于网络中断、计算机重启等场景下已完整的片段已持久化到本地磁盘上，浏览器的安全机制不允许直接访问本地文件系统，(浏览器访问本地文件需要通过浏览器的FILE API取得用户授权，才能read文件)，这种场景下处理合并逻辑需要用户进行手动选取文件，由浏览器JS执行合并逻辑，意味着由一个隐性的过程变成显性的行为过程，作为一个商业化软件，产品设计上很难接受这种笨拙表现。
### 客户端应用及断点续传
1. 客户端应用程序开发：开发一个独立的客户端应用程序，该应用程序可以在用户的计算机上运行，并具备访问本地文件系统的权限。这样可以绕过浏览器的安全限制，实现对本地文件的读取和合并操作。
2. 自动化下载过程，通过应用的任务调度来实现片段的自动下载，面对下载量再多的用户也可一键启动下载队列。
3. 队列下载，通过队列机制解决用户在浏览器中可能存在的高并发问题。（浏览器中的HTTP请求可以是并发请求）
4. 自动化合并过程，当所有片段下载完成后，客户端应用程序可以自动检测并合并这些片段，生成完整的文件。这样可以将合并过程变为一个隐性的操作，提高产品的易用性和用户体验。
### 结论
每张芯片24个DLOC使用7Z压缩后约占磁盘100-110MB，10张芯片1个G，目前大文件的传输靠U盘，小文件的传输靠HTTP，但使用U盘并不意味着杜绝客户使用HTTP进行下载，从上面可以看出使用客户端可以极大的减轻服务器负载压力，减少错误发生，提升整体业务的可靠性行和稳定性，并提供更流畅和稳定的用户体验。
Minio Share URL的配置
## MINIO SDK源码与调用
### MINIO SDK源码
Minio是一个高性能的对象存储服务，支持断点续传。生产系统产生的DLOC文件进入Minio进行存储和管理。同时借助Minio share URL方法，可以实现文件供外部客户临时访问。  
以下是Minio share URL SDK源码：  

```


    def presigned_get_object(self, bucket_name, object_name,
                             expires=timedelta(days=7),
                             response_headers=None,
                             request_date=None,
                             version_id=None,
                             extra_query_params=None):
        """
        Get presigned URL of an object to download its data with expiry time
        and custom request parameters.

        :param bucket_name: Name of the bucket.
        :param object_name: Object name in the bucket.
        :param expires: Expiry in seconds; defaults to 7 days.
        :param response_headers: Optional response_headers argument to
                                  specify response fields like date, size,
                                  type of file, data about server, etc.
        :param request_date: Optional request_date argument to
                              specify a different request date. Default is
                              current date.
        :param version_id: Version ID of the object.
        :param extra_query_params: Extra query parameters for advanced usage.
        :return: URL string.

        Example::
            # Get presigned URL string to download 'my-object' in
            # 'my-bucket' with default expiry (i.e. 7 days).
            url = client.presigned_get_object("my-bucket", "my-object")
            print(url)

            # Get presigned URL string to download 'my-object' in
            # 'my-bucket' with two hours expiry.
            url = client.presigned_get_object(
                "my-bucket", "my-object", expires=timedelta(hours=2),
            )
            print(url)
        """
                return self.get_presigned_url(
            "GET",
            bucket_name,
            object_name,
            expires,
            response_headers=response_headers,
            request_date=request_date,
            version_id=version_id,
            extra_query_params=extra_query_params,
        )
```
在上层逻辑中可以实现身份验证，授权等活动，下层逻辑可以调用该函数生成一个预签名的URL地址，用户访问该地址即可实现下载。  
由于Minio中的对象目前是是以芯片为维度的存储对象，每个芯片24个DLOC都在这个压缩包对象中。一个客户一张订单中如包含200张芯片，即可生成200个SHARE_URL，通过客户端队列管理进行自动下载。（也可调整存储桶逻辑，如倾向于一个URL的大小 可在每个订单中按一个G的字节大小进行多文件再次压缩后存储，避免过多SHARE_URL）

### 客户端调用
Minio的SHARE_URL已经封装了对HTTP头Range头的处理过程，所以客户端在调用时，可直接设置Range参数的字节范围决定取该存储对象的哪个片段。

#### 调用示例：
获取该存储对象前100个字节片段  
`curl -o output_file --header "Range: bytes=0-99" <shareurl>`

## 完整实例
### 服务端实现：
```

import minio
from minio import Minio
from minio.error import S3Error
import os
import time

endpoint = '192.168.110.222:9000'
access_key = 'diaohui'
secret_key = 'laso123456'
bucket_name = 'testbucket'
def upload_file_and_get_share_url(file_path):

    client = Minio(
        endpoint=endpoint,
        access_key=access_key,
        secret_key=secret_key,
        secure=False
    )

    try:
        length = os.path.getsize(file_path)
        start_time = time.time()
        client.put_object(bucket_name=bucket_name, object_name=file_path, length=length, data=open(file_path, 'rb'))
        end_time = time.time()
        upload_time = end_time - start_time

        share_url = client.presigned_get_object(bucket_name, object_name=file_path)
        return share_url, length, upload_time

    except S3Error as err:
        print('文件上传失败', err)
        return None, None, None



share_url, file_size, upload_time = upload_file_and_get_share_url(file_path='318677224273.zip')

if share_url is not None:
    print('共享链接:', share_url)
    print('文件大小:', file_size)
    print('上传时间:', upload_time)
```
### 客户端实现： 

`curl -o output_file --header "Range: bytes=0-99" "http://192.168.110.222:9000/testbucket/test.txt?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=diaohui%2F20231213%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20231213T075359Z&X-Amz-Expires=604800&X-Amz-SignedHeaders=host&X-Amz-Signature=1721671bc7105fc8975a147b1f8d0e7e80e39c21efc6c14b8338d8cbeac12d32"`  
### 文件效果：  
![01.png](/01.png)
