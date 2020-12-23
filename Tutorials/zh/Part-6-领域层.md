#  Web开发教程6 作者：领域层

## 关于此教程

在这个教程系列中，你将要构建一个基于ABP框架的应用程序 Acme.BookStore。这个应用程序被用于甘丽图书页面机器作者。它将用以下开发技术：

- Entity Framework Core 作为数据提供器
- MVC / Razor Pages 作为UI框架



这个教程全部由下面几个部分构成：

- 第一章：创建服务端
- 第二章：构建书籍列表页面
- 第三章：增删改图书
- 第四章：集成测试
- 第五章：授权
- **[第六章：领域层（本教程）](https://www.cnblogs.com/LandWind/p/Web_Application_Development_Tutorial_Part5_Authorization.html)**
- 第七章：数据库集成
- 第八章：应用层
- 第九章：用户接口
- 第十章：预定作者关系



## 下载源码

教程根据您的UI和数据库首选项有多个版本。我们准备了一些可供下载的源码组合：

- MVC(Razor Pages)界面和EFCore版本
- Blazor和EF Core版本
- Angular和MongoDB版本



## 导论

上一教程中，我们使用ABP框架很容易的构建了一些服务：

- 使用CrudAppService 基类替代基础的方法实现，而不是按部就班的创建、读取、更新和删除操作。
- 使用通用的仓储自动化实现数据访问层

对于**”作者“**部分：

- 我们将手动执行某些操作，以显示需要的时候是如何执行的操作
- 我们将实现一些DDD的最佳实践



## 作者实体

在*.BookStore.Domain 项目中创建一个 Authors 文件夹  ，并添加添加类