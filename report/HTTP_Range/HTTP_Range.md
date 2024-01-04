
## 概述

断点续传技术主要用于处理网络传输过程中的中断问题，允许传输任务在中断后从上次停止的地方继续，而不是重新开始。主流视频网站等应用均采用断点续传技术，用户可以从上次观看位置获取视频文件片段而无需重新下载整个视频，从而节省大量的网络带宽和服务器资源。

支持断点续传的协议包括：

- HTTP/HTTPS：用于网页内容的传输，支持断点续传的关键特性是HTTP头部中的Range请求和206 Partial Content响应。
- FTP：文件传输协议，早期用于文件下载，也支持断点续传。
- BitTorrent：一种点对点文件共享协议，天然支持断点续传，因为它从多个源下载文件的不同部分（例如迅雷等P2P技术）。

## 简要过程

断点续传的实现主要包括以下步骤：

1. 服务端对大型文件进行切片，将文件划分为多个片段。
2. 客户端发送带有范围的请求，通过设置HTTP头部的Range参数规定要获取的字节范围。
3. 服务器根据范围请求解析并返回相应的文件内容。
4. 客户端接收部分文件内容，并将其追加到已有文件的末尾。
5. 重复下载所有片段直到完成，客户端可以多次发送范围请求，直到获取完整的文件内容。
6. 客户端对下载完成的片段进行合并，生成完整的文件。

在实现中，可以选择两种思路：

1. 文件追加：客户端边下载边合并文件片段，适用于流媒体等需要边下载边使用的场景。但这种方法增加了文件合并的复杂性。
2. 并发下载：客户端先下载所有片段，然后按顺序进行文件合并，适用于非流媒体等不需要边下载边使用的场景。这种方法简化了合并逻辑。

## 现代浏览器及断点续传

现代浏览器如Chrome、Firefox等内置了对HTTP断点续传的支持，可以顺利下载文件片段。然而，在处理合并的逻辑过程中，由于网络中断、计算机重启等情况，已完整的片段已持久化到本地磁盘上。浏览器的安全机制不允许直接访问本地文件系统，因此处理合并逻辑需要用户手动选取文件，并由浏览器的JavaScript执行合并操作，这种方式不够优雅。

## 客户端应用及断点续传

为了解决浏览器限制的问题，可以开发一个独立的客户端应用程序，在用户的计算机上运行，并具备访问本地文件系统的权限。这样可以绕过浏览器的安全限制，实现对本地文件的读取和合并操作。

客户端应用程序可以实现以下功能：

1. 自动化下载过程：通过应用的任务调度来实现片段的自动下载，即使面对大量的用户下载需求，也可以一键启动下载队列。
2. 队列下载：通过队列机制解决浏览器中可能存在的高并发问题，利用浏览器的并发请求能力进行下载。
3. 自动化合并过程：当所有片段下载完成后，客户端应用程序可以自动检测并合并这些片段，生成完整的文件。这样可以将合并过程变为一个隐性的操作，提高产品的易用性和用户体验。

## Minio Share URL的配置

Minio是一个高性能的对象存储服务，支持断点续传。在生产系统中，产生的DLOC文件会进入Minio进行存储和管理。同时，借助Minio的Share URL方法，可以实现文件供外部客户临时访问。

以下是Minio Share URL的SDK源码示例：

```python
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

在上层逻辑中，可以实现身份验证、授权等活动，然后调用该函数生成一个预签名的URL地址，用户访问该地址即可实现下载。

对于Minio中的对象，目前是以芯片为维度的存储对象，每个芯片的24个DLOC都在同一个压缩包对象中。例如，如果一个客户的订单中包含200张芯片，那么可以生成200个Share URL，并通过客户端队列管理进行自动下载。（也可以调整存储桶逻辑，按照一个G的字节大小对每个订单进行多文件再次压缩后存储，以避免过多的Share URL）

## 客户端调用

Minio的Share URL已经封装了对HTTP头Range的处理过程，因此客户端在调用时，只需要设置Range参数的字节范围来决定获取存储对象的哪个片段。

调用示例：

```bash
curl -o output_file --header "Range: bytes=0-99" <shareurl>
```

## 完整实例

服务端示例：

```python
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

客户端示例：

```bash
curl -o output_file --header "Range: bytes=0-99" "http://192.168.110.222:9000/testbucket/test.txt?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=diaohui%2F20231213%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20231213T075359Z&X-Amz-Expires=604800&X-Amz-SignedHeaders=host&X-Amz-Signature=1721671bc7105fc8975a147b1f8d0e7e80e39c21efc6c14b8338d8cbeac12d32"
```
文件效果：
![ba3249e6-f980-4eb5-b7c0-c6b817d89e09.png](ba3249e6-f980-4eb5-b7c0-c6b817d89e09.png)