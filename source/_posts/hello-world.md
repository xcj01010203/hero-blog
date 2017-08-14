---
title: Office转PDF
date: 2013-07-14 17:01:34
tags:
- JodConverter
categories:
- JAVA
description: 介绍使用JodConverter把文件转换为pdf文件的java方法。本文使用了jodconverter2.2.1和jodconverter3.0两个版本的api
---

![cmd-markdown-logo](/images/bg.jpg)

#### 利用jodconverter2.2.1
-  缺点：openoffice一旦启动，便一直存在于内存中，占用内存
- 优点：如果经常需要执行转换操作，则省去了大量启动/关闭openoffice服务的时间，而且适用于多并发访问的情况
- 使用场景：在经常需要转换的时候使用
- 补充：此种解决方式有一个致命的弊端，自己手动启动的openoffice服务会无故关闭，暂时找不到很好的解决办法
```
<dependency>
<groupId>com.artofsolving</groupId>
<artifactId>jodconverter</artifactId>
<version>2.2.1</version>
</dependency>
```
- 操作流程
1. 安装openoffice，自己百度
2. 启动openoffice服务
- window下：
soffice -headless -accept="socket,port=2017;urp;" -nofirststartwizard
//socket,host=localhost,port=2017,tcpNoDelay=1

- linux下：
export DISPLAY=:0.0
soffice --headless --accept="socket,port=2017;urp;" --nofirststartwizard&

3. java代码
```java
	/**
	 * 将Office文档转换为其他格式. 运行该函数需要用到OpenOffice
	 * 通过指定outputFilePath文件后缀，该方法亦可实现将Office文档转换为TXT、PDF等格式.
	 * 运行该函数需要用到OpenOffice,需要在服务器上安装OpenOffice 文件转换成功与否以异常的形式抛出
	 *
	 * @param inputFilePath
	 *            源文件,绝对路径. 可以是Office2003-2007全部格式的文档, Office2010的没测试. 包括.doc,
	 *            .docx, .xls, .xlsx, .ppt, .pptx等.
	 * @param outputFilePath
	 *            转换后文件输出路径
	 *
	 * @throws IOException
	 * @throws FileNotFoundException
	 */
	public static String word2Format(String inputFilePath, String outputFilePath)
			throws FileNotFoundException, IOException {

		OpenOfficeConnection connection = new SocketOpenOfficeConnection(2017);
		try {
			File inputFile = new File(inputFilePath);
			if (!inputFile.exists()) {
				throw new IllegalArgumentException("找不到需要转换的文件");
			}

			File outputFile = new File(outputFilePath);
			if (!outputFile.getParentFile().exists()) { // 假如目标路径不存在, 则新建该路径
				outputFile.getParentFile().mkdirs();
			}

			connection.connect();

			DocumentConverter converter = new OpenOfficeDocumentConverter(connection);
			converter.convert(inputFile, outputFile);
		} catch (Exception e) {
			throw new IllegalArgumentException(e);
		} finally {
			if (connection.isConnected()) {
				connection.disconnect();
				connection = null;
			}
		}

		return outputFilePath;
	}
```

#### 利用jodconverter3.0
```
<dependency>
<groupId>com.github.livesense</groupId>
<artifactId>org.liveSense.framework.jodconverter-osgi</artifactId>
<version>1.0.5</version>
</dependency>
```
- 优点：只有在使用的时候才去开启openoffice服务，而且使用过后会关闭服务，节省内存
- 缺点：多线程并发问题无法解决，开启/关闭服务很消耗时间
- 使用场景：在很少使用转换功能的情况下使用
- 补充：多线程并发时，虽然会产生重复启动的bug，但是通过把连接动作封装到单例模式中，并且进行线程安全控制，也是可以支持多并发的情况的。（jodconverter自己可能进行了任务队列控制操作）

转换代码：
```java
	/**
	 * 将Office文档转换为其他格式. 运行该函数需要用到OpenOffice
	 * 通过指定outputFilePath文件后缀，该方法亦可实现将Office文档转换为TXT、PDF等格式.
	 * 运行该函数需要用到OpenOffice,需要在服务器上安装OpenOffice 文件转换成功与否以异常的形式抛出
	 *
	 * @param inputFilePath
	 *            源文件,绝对路径. 可以是Office2003-2007全部格式的文档, Office2010的没测试. 包括.doc,
	 *            .docx, .xls, .xlsx, .ppt, .pptx等.
	 * @param outputFilePath
	 *            转换后文件输出路径
	 *
	 * @throws IOException
	 * @throws FileNotFoundException
	 */
	public static String word2Format(String inputFilePath, String outputFilePath)
			throws FileNotFoundException, IOException {

		OfficeManager officeManager = MyOfficeManager.getInstance();
		try {
			OfficeDocumentConverter converter = new OfficeDocumentConverter(officeManager);
			File inputFile = new File(inputFilePath);
			if (inputFile.exists()) {// 找不到源文件, 则返回
				File outputFile = new File(outputFilePath);
				if (!outputFile.getParentFile().exists()) { // 假如目标路径不存在, 则新建该路径
					outputFile.getParentFile().mkdirs();
				}
				converter.convert(inputFile, outputFile);
			} else {
				throw new IllegalArgumentException("找不到需要转换的文件");
			}
		} catch (Exception e) {
			throw new IllegalArgumentException(e);
		}

		return outputFilePath;
	}
```
单例模式代码：
```java
public class MyOfficeManager {

	private MyOfficeManager() {}

	private static OfficeManager officeManager = null;

	public static synchronized OfficeManager getInstance() throws FileNotFoundException, IOException {
		if (officeManager == null) {
			Properties properties = PropertiesUitls.fetchProperties("/config.properties");
			String officeHome = properties.getProperty("openInstallPath");

			DefaultOfficeManagerConfiguration config = new DefaultOfficeManagerConfiguration();
			config.setOfficeHome(officeHome);

			officeManager = config.buildOfficeManager();
			officeManager.start();
		}
		return officeManager;
	}
}
```
