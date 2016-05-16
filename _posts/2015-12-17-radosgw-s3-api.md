---
layout: post
title: Radosgw S3 API
tags: ceph s3

---


# Radosgw S3 API

Ceph对象网关Radosgw支持Restful API，如Amazon S3 API，Switf API等。
现在测试Amazon S3 API的python API的相关情况。

Radosgw支持的API有：

|  feature         |           status  |
| -----------------|:-----------------:|
| List Buckets     | supported |
| Create Bucket   | supported |
| Delete Bucket   | supported |
| Bucket ACLs(Get, Put)   | supported |
| Bucket Location   | supported |
| Get Bucket Info(Head) | supported |
| Put Object      | supported |
| Get Object      | supported |
| Delete Object   | supported |
| Object ACLs(Get, Put)     | supported |
| Get Object info(Head) | supported |
| Post Object     | supported |
| Copy Object     | supported |
| Multipart Upload | supported |


## 创建连接
要通过S3连接Radosgw访问Ceph，需要先在Radosgw-admin创建S3账户，根据
返回的(AccessKey, SecretKey)和Radosgw的HOST进行连接。

```
boto.connect_s3(aws_access_key_id=None, aws_secret_access_key=None, **kwargs)
Parameters:	
aws_access_key_id (string) – Your AWS Access Key ID
aws_secret_access_key (string) – Your AWS Secret Access Key
Return type: boto.s3.connection.S3Connection
Returns:    A connection to Amazon’s S3
```

创建一个S3Connection对象，进行连接。也可以使用boto.s3.connecton.S3Connection
实例化。

Code:
```
access_key = 'FGOSHYBW8F3GXGAXNR28'
secret_key = 'k71RtJRKrMoAyH5IrI/jZ3q0xG+r2UMPhK1c1Bb/'

conn = boto.connect_s3(
aws_access_key_id = access_key,
aws_secret_access_key = secret_key,
host = 'phycles',
is_secure=False,
calling_format = boto.s3.connection.OrdinaryCallingFormat(),
)
```

## Bucket 操作

Bucket操作一些是S3Connection对象的方法，通过S3Connection进行调用。
一些是Bucket的属性和方法。

### Create Bucket
```
create_bucket(bucket_name, headers=None, location='', policy=None)
Parameters:	
* bucket_name (string) – The name of the new bucket
* headers (dict) – Additional headers to pass along with the request to AWS.
* location (str) – The location of the new bucket. You can use one of the
                   constants in boto.s3.connection.Location (e.g.Location.EU,
				   Location.USWest, etc.).
* policy (boto.s3.acl.CannedACLStrings) – A canned ACL policy that will be
	      applied to the new key in S3.
```
S3Connection对象的方法，每个Bucket都是全局唯一的，所以创建Bucket的时候如果
已经存在了，那么会创建失败。默认情况下只需要提供Bucket_name就可以了。

Code:
```
bucket = conn.create_bucket('test-bucket')
```

### List All Bucket
```
get_all_buckets(headers = None)
```
S3Connection对象的方法，返回S3账户中所有的bucket。

```
for bucket in conn.get_all_buckets():
        print "{name}\t{created}".format(
                name = bucket.name,
                created = bucket.creation_date,
)
```

### List Content in Bucket
```
list(prefix='', delimiter='', marker='', headers=None, encoding_type=None)
Return type:  BucketListResultSet
```
Bucket对象的方法，返回Bucket中所有Object的key链表， 返回的类型为BucketListResultSet.

```
for key in bucket.list():
        print "{name}\t{size}\t{modified}".format(
                name = key.name,
                size = key.size,
                modified = key.last_modified,
)
```

### Bucket ACLs(Put, Get)
获取和设置Object的ACL，有public-read, private等。
```
get_acl(key_name='', headers=None, version_id=None)
get_xml_acl(key_name='', headers=None, version_id=None)
```
Bucket对象的方法，获得Object的ACL。

```
set_acl(acl_or_str, key_name='', headers=None, version_id=None)
set_canned_acl(acl_str, key_name='', headers=None, version_id=None)
set_xml_acl(acl_str, key_name='', headers=None, version_id=None,
	query_args='acl')
```
Bucket对象的方法，设置Object的ACL。

Code:
```
hello_key = bucket.get_key('apple.txt')
hello_key.set_canned_acl('public-read')
```

### Delete Bucket
```
delete_bucket(bucket, headers=None)
Parameters:	
* bucket_name (string) – The name of the bucket
* headers (dict) – Additional headers to pass along with the
  request to AWS.
```

S3Connection对象的方法，要删除一个bucket，必须要确保bucket为空，
bucket不为空，则删除失败。

Code:
```
conn.delete_bucket(bucket.name)
```

## Object 操作

### Put Object
```
new_key(key_name=None)
Parameters:
* key_name (string) – The name of the key to create
* Return type:	boto.s3.key.Key or subclass
* Returns:	An instance of the newly created key object
```

上传Object要先Bucket里新建一个Key，然后再通过Key上传对象。
上传内容有多种方式：

*  文件名路径
*  文件指针
*  字符串
*  流

通过已打开的文件指针所指向的内容作为存储Object的内容。文件为EOF或者文件指针读取
数据到指定的Size时停止。

```
set_contents_from_file(fp, headers=None, replace=True, cb=None, num_cb=10,
	policy=None, md5=None, reduced_redundancy=False, query_args=None,
	encrypt_key=False, size=None, rewind=False)
Parameter:
* fp - file pointer whose content to upload
Return type: int
Returns: The number of bytes written to the key.
```

通过文件名上传Object，上传的Object的数据为文件名所有的文件。(文件路径)
```
set_contents_from_filename(filename, headers=None, replace=True, cb=None,
num_cb=10, policy=None, md5=None, reduced_redundancy=False, encrypt_key=False)
Parameters:	
* filename (string) – The name of the file that you want to put onto S3
Return type: int
Returns: The number of bytes written to the key.
```

通过字符串上传Object的内容，将字符串上传。 对于对象存储而言，没有文件夹的概念，
所有文件和文件夹都是看成一个Object。所以想要在对象存储表示文件夹，可以在Object
里有字符'/'来表示文件夹意义的标识符，但是利用前面的概念可以新建一个带有'/'的key，
这个key内容为空，来象征性的表示文件夹。这个文件夹里的Object都可以在新建的时候，
通过在key里面以这个文件的key最为前缀来表示，这样就可以表示为这个文件夹里面的文件
了。
```
set_contents_from_string(string_data, headers=None, replace=True, cb=None,
num_cb=10, policy=None, md5=None, reduced_redundancy=False, encrypt_key=False)
Parameter:
* string_data - the string to upload
```

通过文件指针所指向的数据流作为上传Object的内容。但是流对象是不可以移位的，数据
大小也是未知的，我们是无法在header里面指定内容的大小和内容的MD5。所以大文件上传
时可以避免计算MD5所带来的延迟，但是无法验证上传的数据。
```
set_contents_from_stream(fp, headers=None, replace=True, cb=None, num_cb=10,
policy=None, reduced_redundancy=False, query_args=None, size=None)
Parameters:	
* fp (file) – the file whose contents are to be uploaded
```
Code:
```
key = bucket.new_key('apple.txt')
key.set_contents_from_filename('./apple.txt')
```

### Get Object
获取对象与上传对象方式有些类似，通过向Bucket提供KeyName获得key对象，再过通Key
获取对象。调用这个方法之前要先确认这个Key是否已经那个创建了。此方法会使用HEAD
方法请求key的存在。

```
get_key(key_name, headers=None, version_id=None, response_headers=None,
	   validate=True)
Parameters:	
* key_name (string) – The name of the key to retrieve
* headers (dict) – The headers to send when retrieving the key
* version_id (string) – 
* response_headers (dict) – A dictionary containing HTTP headers/values
  that will override any headers associated with the stored object in
  the response.
* validate (bool) – Verifies whether the key exists. If False, this will
  not hit the service, constructing an in-memory object. Default is True.
Return type:  boto.s3.key.Key
Returns: A Key object from this bucket.
```

Key对象的方法，以字符串的形式获取Object的内容，Object的内容以字符串返回。
```
get_contents_as_string(headers=None, cb=None, num_cb=10, torrent=False,
	version_id=None, response_headers=None, encoding=None)
Return type: bytes or str
Returns: The contents of the file as bytes or a string
```

Key对象的方法，通过文件指针的形式获取Object的内容，将Object的内容下载到
已打开的文件指针所指向的文件中。
```
get_contents_to_file(fp, headers=None, cb=None, num_cb=10, torrent=False,
	version_id=None, res_download_handler=None, response_headers=None)
Parameter:
* fp - file pointer to put data in
Return type: None
```

Key对象的方法，将Object的内容下载到文件名(文件名路径)所在的位置。
```
get_contents_to_filename(filename, headers=None, cb=None, num_cb=10,torrent
	=False, version_id=None, res_download_handler=None, response_headers=None)
Parameter:
filename - file path to put data in
Return type: None
```

Key对象的方法，将Object的内容直接存储到文件指针所指向的文件。
```
get_file(fp, headers=None, cb=None, num_cb=10, torrent=False, version_id
=None,override_num_retries=None, response_headers=None)
Parameter:
 fp - file pointer to put the data in
Return type: None
```

Code:
```
get_key = bucket.get_key('apple.txt')
get_key.get_contents_to_filename('./get.txt')
```

### Delect Object
删除Bucket中的Object，可以指定Object的版本进行删除，如果指定了版本，
那么就只有指定的版本会被删除。

```
delete_key(key_name, headers=None, version_id=None, mfa_token=None)
Parameters:	
key_name (string) – The key name to delete
version_id (string) – The version ID (optional)
mfa_token (tuple or list of strings) – A tuple or list consisting of
	the serial number from the MFA device and the current value of the
	six-digit token associated with the device. This value is required
	anytime you are deleting versioned objects from a bucket that has
	the MFADelete option on the bucket.
Return type:  boto.s3.key.Key or subclass
Returns: A key object holding information on what was deleted. 
```

从Bucket中删除Object，只需要将Object的Key删除掉就可以了。
```
bucket.delete_key('apple.txt')
```

### Get Object ACLs(Put, Get)
Key对象的方法，获取Object的ACLs，一种以字符串的形式返回，另一种以xml的
形式返回。
```
get_acl(headers=None)
```
```
get_xml_acl(headers=None)
```

Key对象的方法，设置Object的ACLs，其中acl_str可以为public-read, private等。
通过设置ACL可以用来实现文件的共享。
```
set_acl(acl_str, headers=None)
set_canned_acl(acl_str, headers=None)
```

### Generate Download URL
Key的方法，生成访问Key的URL。生成Object的下载URL，可以选择有效时间(以秒为单位)。
可以用来实现外链下载。
```
generate_url(expires_in, method='GET', headers=None, query_auth=True,
force_http=False, response_headers=None, expires_in_absolute=False,
version_id=None, policy=None, reduced_redundancy=False, encrypt_key=False)
Parameters:	
expires_in (int) – How long the url is valid for, in seconds.
method (string) – The method to use for retrieving the file (default is GET).
headers (dict) – Any headers to pass along in the request.
query_auth (bool) – If True, signs the request in the URL.
Return type: string
Returns: The URL to access the key
```

Code:
```
hello_url = hello_key.generate_url(0, query_auth=False, force_http=True)
```
Bucket的方法，生成访问Bucket的URL。
```
generate_url(expires_in, method='GET', headers=None, force_http=False,
	response_headers=None, expires_in_absolute=False)
```


### Copy Object

Bucket的方法，通过复制现有的Key在Bucket新建Key，该方法可以用来实现文件的移动。
```
copy_key(new_key_name, src_bucket_name, src_key_name, metadata=None,
   src_version_id=None, storage_class='STANDARD', preserve_acl=False,
   encrypt_key=False, headers=None, query_args=None)
Parameters:	
new_key_name (string) – The name of the new key
src_bucket_name (string) – The name of the source bucket
src_key_name (string) – The name of the source key
Return type: boto.s3.key.Key or subclass
Returns: An instance of the newly created key object
```

Key的方法，将Key复制到其他Bucket中，该方法可以用来实现文件的移动。
```
copy(dst_bucket, dst_key, metadata=None, reduced_redundancy=False,
preserve_acl=False, encrypt_key=False, validate_dst_bucket=True)
Parameters:	
dst_bucket (string) – The name of the destination bucket
dst_key (string) – The name of the destination key
Return type: boto.s3.key.Key or subclass
Returns: An instance of the newly created key object
```


## 参考
[RADOSGW S3 API](http://docs.ceph.com/docs/master/radosgw/s3/)

[Intro to S3 API](http://boto.cloudhackers.com/en/latest/s3_tut.html)

[Boto S3](http://boto.cloudhackers.com/en/latest/ref/s3.html#module-boto.s3)
