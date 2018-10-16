---
title: 下载文件，header中Content-Disposition中文文件名乱码
tags: [开发日志]
date: 2018-10-16
---
## 前言
在用Java下在文件时，需要取到下载文件的名称，head中取Content-disposition，显示为乱码，在此记录下这个问题。


## 代码
```
	Map<String, List<String>> heads = connection.getHeaderFields();
	String description = heads.get("Content-disposition").get(0);
	//取到文件名
	String fileName = StringUtil.match(description, Pattern.compile("(?!\")\\S+(?<!\")"));
	//需要把iso8859-1转换成gbk。问题解决
	//转成gbk是因为在Windows环境下。
	fileName = new String(fileName.getBytes("iso8859-1"), "gbk");
	FileOutputStream fileOutputStream = new FileOutputStream("C:\\down\\" + fileName);
    FileCopyUtils.copy(connection.getInputStream(), fileOutputStream);
```
思路:
[这篇文章里面写了](https://blog.csdn.net/liuyaqi1993/article/details/78275396)，RFC 822（ Standard for ARPA Internet Text Messages）规定了文本消息只能为ASCII，所以在服务器下载文件时设置head，要把文件名转换成iso8859-1，故需要把head的文件名先转成iso8859-1，再转成目的需要的编码，即可解决乱码问题。