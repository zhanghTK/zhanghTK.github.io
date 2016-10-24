---
title: 使用Builder模式改进多参数方法
date: 2016-10-16 11:58:39
tags: 
  - 设计模式
  - Java
category: 设计模式
---

## 概述

记一次工作当中对多参数方法重构。

1. 使用对象封装对多参数，简化方法调用
2. 使用Builder（创建者）模式简化多属性对象的创建

## 问题

业务系统中统一的邮件发送服务接口在改造前大概长着个样子：

```java
	/**
     * 不帶附件的邮件发送
     * 未使用建造者模式的原始方法（不良代码）
     * @param template                      模板
     * @param subjects                      主题
     * @param contents                      内容
     * @param toPersons                     收件人
     * @param ccPersons                     抄送人
     * @param bccPersons                    暗送人
     */
    public static void sendEmail(String template, List<String> subjects, List<String> contents, List<String> toPersons,
                                  List<String> ccPersons, List<String> bccPersons){
        logger.info("send email");
    }


    /**
     * 带附件的邮件发送
     * 未使用建造者模式的原始方法（不良）
     * @param template                      模板
     * @param subjects                      主题
     * @param contents                      内容
     * @param toPersons                     收件人
     * @param ccPersons                     抄送人
     * @param bccPersons                    暗送人
     * @param docName                       文档名称
     * @param fileName                      文件名称(单个文件)
     */
    public static void sendEmail(String template, List<String> subjects, List<String> contents, List<String> toPersons,
                                  List<String> ccPersons, List<String> bccPersons, String docName, String fileName){
        logger.info("send email");
    }

    /**
     * 带附件的邮件发送
     * 未使用建造者模式的原始方法（不良代码）
     * @param template                      模板
     * @param subjects                      主题
     * @param contents                      内容
     * @param toPersons                     收件人
     * @param ccPersons                     抄送人
     * @param bccPersons                    暗送人
     * @param docName                       文档名称
     * @param fileNames                     文件名称(多个文件,文件名称列表)
     */
    public static void sendEmail(String template, List<String> subjects, List<String> contents, List<String> toPersons,
                                  List<String> ccPersons, List<String> bccPersons, String docName, List<String> fileNames){
        logger.info("send email");
    }
```

接口有个重要的内容注释中并没有说明：参数中“模板”和“收件人”是必填的，而其他参数是非必填的。

接口的可配置程度还是不错的，但是调用的过程就比较痛苦了。

一大堆String和List接口暴露出来，同时又使用了不同的参数个数来进行重载。

调用的时候光是创建这些参数就够麻烦的了，还要考虑哪些参数是必填的以及参数的正确位置。更糟糕的是参数传错位置你会发现很有可能并没有显式的暴露出问题，邮件还是发送了只是发送到错误的相关人员那里。

其他人看到方法调用也无法清晰知道这个到底是要发什么邮件，给哪些人。

这应该就是坏代码的味道吧。

## 改进

简单的改进思路就是把参数做成一个modle封装起来，以后传递model给方法。就像这样：

```java
    /**
     * 邮件发送通用接口
     * @param email 邮件发送参数对象
     */
    public static void sendEmail(EmailSendMain email){
    	logger.info("send email:" + email);
    }
```

但是问题还是没有根本解决，对象的构造还是需要多个参数的构造方法，可能还需要重载。

比较容易想到的改进是使用Java Bean模式。简化构造方法字段，构造方法只传入必要的字段，使用setter方法设置其他值。我就是这么肤浅的想到这个地步了。

乍一看问题是解决了，其实不然。

1. 在对象创建过程中Java Bean可能处于不一致状态
2. 使用Java Bean就将不能创建不可变对象

读了《Effective Java》只是第二章就有了更好的解决思路——Builder模式。

先看改进后的代码：

```java
/**
 * 复杂类型构建接口
 *
 * 建造者模式中的抽象构建者
 * Created by ZhangHao on 2016/10/15.
 */
public interface Builder<T> {
    T build();
}

/**
 * 邮件发送参数对象
 * 包含多个字段的复杂类型，使用内部类实现Builder接口创建对象
 *
 * 建造者模式中的产品类
 * Created by ZhangHao on 2016/10/15.
 */
public final class EmailSendMain {
    private final String template;  // 模板名称
    private final List<String> subjects;  // 主题参数列表
    private final List<String> contents;  // 内容参数列表
    private final List<String> toPersons;  // 收件人列表
    private final List<String> ccPersons;  // 抄送人列表
    private final List<String> bccPersons;  // 暗送人列表
    private final String docName;  // 文档名称
    private final List<String> fileNames;  // 文件名称列表

    private EmailSendMain(Builder builder) {
        this.template = builder.template;
        this.subjects = builder.subjects;
        this.contents = builder.contents;
        this.toPersons = builder.toPersons;
        this.ccPersons = builder.ccPersons;
        this.bccPersons = builder.bccPersons;
        this.docName = builder.docName;
        this.fileNames = builder.fileNames;
    }

    /**
     * 实现Builder接口的构建类，用于创建EmailSendMain
     *
     * 建造者模式中的建造类
     */
    public static class Builder implements tk.zhangh.pattern.create.builder.demo1.Builder<EmailSendMain> {
        private String template;  // 模板名称
        private List<String> subjects;  // 主题参数列表
        private List<String> contents;  // 内容参数列表
        private List<String> toPersons;  // 收件人列表
        private List<String> ccPersons;  // 抄送人列表
        private List<String> bccPersons;  // 暗送人列表
        private String docName;  // 文档名称
        private List<String> fileNames;  // 文件名称列表

        public Builder(String template, List<String> toPersons) {
            this.template = template;
            this.toPersons = toPersons;
        }

        @Override
        public EmailSendMain build() {
            return new EmailSendMain(this);
        }

        public Builder subjects(List<String> subjects) {
            this.subjects = subjects;
            return this;
        }

        public Builder contents(List<String> contents) {
            this.contents = contents;
            return this;
        }

        public Builder ccPersons(List<String> ccPersons) {
            this.ccPersons = ccPersons;
            return this;
        }

        public Builder bccPersons(List<String> bccPersons) {
            this.bccPersons = bccPersons;
            return this;
        }

        public Builder docName(String docName) {
            this.docName = docName;
            return this;
        }

        public Builder fileNames(List<String> fileNames) {
            this.fileNames = fileNames;
            return this;
        }
    }

    // getter,toString方法省略
}
```

重写做的接口方法封装：

```java
    /**
     * 邮件发送通用接口
     * @param email 邮件发送参数对象
     */
    public static void sendEmail(EmailSendMain email){
        logger.info("send email:" + email);
        if ((email.getDocName() == null || email.getDocName().equals("")) ||
                (email.getFileNames() == null || email.getFileNames().size() == 0)) {
            sendEmail(email.getTemplate(), email.getSubjects(), email.getContents(), email.getToPersons(),
                    email.getCcPersons(), email.getBccPersons());
        }else {
            sendEmail(email.getTemplate(), email.getSubjects(), email.getContents(), email.getToPersons(),
                    email.getCcPersons(), email.getBccPersons(), email.getDocName(), email.getFileNames());
        }
    }
```

客户端调用：

```java
    @Test
    public void testSendEmail() throws Exception {
        EmailSendMain email =
                new EmailSendMain.Builder("邮件模版名",toPersons).
                        subjects(subjects).
                        contents(contents).
                        ccPersons(ccPersons).
                        bccPersons(bccPersons).build();
        SendEmailUtil.sendEmail(email);
    }
```

问题圆满解决，支持可选参数的链式结构调用，创建过程也确保了一致性，使用不可变类也没问题。

如果说缺点，其实不难看出EmailSendMain的字段和它的内部类Builder字段完全重复了。为了创建EmailSendMain的对象将必须先创建Builder也会带来轻微的性能问题。创建的调用过程虽然看起来更清晰，但也更加冗长。

但是Builder模式还是创建多参数类的不错选择，尤其是大多数参数是可选。

## 扩展

这篇文章的思路是根据《Effective Java》得来的，文章只提到书中建议的第二条，实际关于上面使用到的内部类，泛型，在书中的建议都让我有了更多的认识。我就不赘述了，连上8天班我要去偷懒了。

写这篇文章的时候看到有个系列专门讲Java方法参数太多的问题

传送门：https://dzone.com/articles/too-many-parameters-java

以及翻译：http://www.importnew.com/6518.html

代码我放在了学习设计模式的项目下：

传送门：https://github.com/zhanghTK/HelloDesignPattern
