---
title: Mybatis 实现物理分页
date: 2017-08-07 16:23:33
tags:
type: "categories"
categories: "Java"
---


<li>分页的分类
---
1.物理分页: 只从数据库中查询当前被查询页的数据,在实际的业务中基本都是使用物理分页.
<!-- more -->
2.逻辑分页: 将所有数据都先查询出来,然后会存储到内存中,再取出对应的数据,这种分页如果在数据量很大的情况下,内存会占用率就会非常高,比较适合少量数据的情况下的使用.

<li>查询语句
---
mysql: limit start, size 
>start表示起始的索引,比如说我要查询第五页的数据,那起始的索引就是40, size表示一页的数量,比如10的话,就是40-50的数据.

oracle: rownum

<li>实例
---
定义一个实体类

```java
//用户记录日志类
pulic class UserLog {
   private Integer id;
   
   private String module;
   
   private String function;
   
   private String ip;
   
   private String time;
   
   //Getter and Setter
}

```

一个简单的Page类

```java
pulic class Page {
   private int pageNum;
   
   private int pageSize = 10; //默认为10
   
   private int startIndex; //计算起始索引
   
   //Getter and Setter
   
   public void setPageNum(int pageNum) {
      this.pageNum = pageNum;
      setStartIndex(pageNum * getPageSize())
   }
   
    public void setPageSize(int pageSize) {
      this.pageNum = pageNum;
      setStartIndex(pageNum * getPageSize())
   }
   //其他的就省略了
}

```
mapper

```xml
mapper.xml配置文件
<select id="selectByPage" parameterType="com.ysq.util.Page"
          resultMap="BaseResultMap">
    select * from user_log
    limit #{page.startIndex}, #{page.pageSize}
  </select>
  
mapper
public List<UserLog> selectByPage(@Param("page") page)

```
dao

```java
public List<UserLog> selectByPage(Page page) {
   return this.userLogMapper.selectByPageSize(page);
}

```
测试一下

```java
public static void main(String[] args) {
    ApplicationContext context = new
                ClassPathXmlApplicationContext("spring/applicationContext.xml");

    UserLogService service = (UserLogService) context.getBean("userLogService");

    Page page = new Page();
    page.setPageSize(10);
    page.setPageNum(1);

    List<UserLog> logs = service.selectByPage(page);
    for (UserLog log: logs) {
        System.out.println("功能: " +log.getFunction() + " id: " + log.getId());
   }
}
```
![结果](/images/mybatisPage.png)

<li>带条件查询
---
也可以在语句中插入条件然后进行分页查询.
例如: 查询id>100的数据.

```xml
<select id="selectByPage" parameterType="com.ysq.util.Page"
          resultMap="BaseResultMap">
    select * from user_log where id > 100
    limit #{page.startIndex}, #{page.pageSize}
  </select>
```
<li>总结
---
从上面看来,通过mybatis实现分页查询还是较为简单的,希望这篇文章对大家有所帮助!谢谢.
