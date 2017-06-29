---
title: Asprise OCR Java 二维码，条形码识别
date: 2017-06-29 17:02:42
tags:
- OCR
- Java
- 随笔
---
Asprise OCR Java 二维码，条形码识别。

 - 添加Asprise OCR Maven / Gradle依赖
```java
<dependency>
    <groupId>com.asprise.ocr</groupId>
    <artifactId>java-ocr-api</artifactId>
    <version>[15,)</version>
</dependency>
```
```java
compile group: 'com.asprise.ocr', name: 'java-ocr-api', version: '[15,)'
```

 - 基本语法

```java
import com.asprise.ocr.Ocr
...

Ocr.setUp(); // one time setup
Ocr ocr = new Ocr(); // create a new OCR engine
ocr.startEngine("eng", Ocr.SPEED_FASTEST); // English
String s = ocr.recognize(new File[] {new File("test-image.png")}, Ocr.RECOGNIZE_TYPE_ALL, Ocr.OUTPUT_FORMAT_PLAINTEXT);
System.out.println("Result: " + s);
// ocr more images here ...
ocr.stopEngine();
```

官网：http://asprise.com/ocr/docs/html/asprise-ocr-api-java.html
文件包： http://pan.baidu.com/s/1pLG2687 密码: ddra