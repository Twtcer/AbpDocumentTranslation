# 原文档

地址：
 [Web Application Development Tutorial - Part 5: Authorization  ](https://docs.abp.io/en/abp/latest/Tutorials/Part-5?UI=MVC&DB=EF)

# 关于此教程

在这个教程系列中，您将构建一个基于ABP的Web应用程序。此应用程序用于管理书籍及其作者的列表。 它将使用以下技术开发:Acme.BookStore(译者:您创建项目名称)

- EntityFramework Core 作为ORM提供器
- MVC/Razor Pages 作为UI框架

这个教程全部由下面几个部分构成：

- 第一章：创建服务端
- 第二章：构建书籍列表页面
- 第三章：增删改图书
- 第四章：集成测试
- [第五章：授权（本教程）](https://www.cnblogs.com/LandWind/p/Web_Application_Development_Tutorial_Part5_Authorization.html)
- 第六章：领域层
- 第七章：数据库集成
- 第八章：应用层
- 第九章：用户接口
- 第十章：预定作者关系

## 下载源码

教程根据您的UI和数据库首选项有多个版本。我们准备了一些可供下载的源码组合：

- MVC(Razor Pages)界面和EFCore版本
- Blazor和EF Core版本
- Angular和MongoDB版本

## 视频教程

此教程亦提供视频教程 在YouTube(译者：稍后将转移到Bilibili)

# 权限

ABP框架提供基于Asp.Net Core权限授权基础的权限系统。在标准授权基础结构上添加一项主要功能就是权限系统，该系统允许定义权限并针对每个角色、用户或用户端启用/禁用。

## 权限名称

权限必须有唯一的名称，最好的方法是将其定义为我们可以重复使用的权限名称。

打开项目 *.Application.Contracts 文件夹Permissions 并更新内容如下:

```
namespace Acme.BookStore.Permissions
{
    public static class BookStorePermissions
    {
        public const string GroupName = "BookStore";

        public static class Books
        {
            public const string Default = GroupName + ".Books";
            public const string Create = Default + ".Create";
            public const string Edit = Default + ".Edit";
            public const string Delete = Default + ".Delete";
        }
    }
}

```

这个是定义权限名称的分层方法。举个例子，"create book" 权限名称将定义为创建图书。ABP不会强迫您使用这个规范，但是我们发现这个方法很有用。BookStore.Books.Create

## 权限定义

你需要先定义权限后再使用。

项目路径： BookStorePermissionDefinitionProviderAcme.BookStore.Application.ContractsPermissions
打开以上路径类，按照以下修改：

```
using Acme.BookStore.Localization;
using Volo.Abp.Authorization.Permissions;
using Volo.Abp.Localization;

namespace Acme.BookStore.Permissions
{
    public class BookStorePermissionDefinitionProvider : PermissionDefinitionProvider
    {
        public override void Define(IPermissionDefinitionContext context)
        {
            var bookStoreGroup = context.AddGroup(BookStorePermissions.GroupName, L("Permission:BookStore"));

            var booksPermission = bookStoreGroup.AddPermission(BookStorePermissions.Books.Default, L("Permission:Books"));
            booksPermission.AddChild(BookStorePermissions.Books.Create, L("Permission:Books.Create"));
            booksPermission.AddChild(BookStorePermissions.Books.Edit, L("Permission:Books.Edit"));
            booksPermission.AddChild(BookStorePermissions.Books.Delete, L("Permission:Books.Delete"));
        }

        private static LocalizableString L(string name)
        {
            return LocalizableString.Create<BookStoreResource>(name);
        }
    }
}

```

此类定义一个权限组(对UI上权限进行分组，如下所示)和该组的4个权限。同样的，创建，编辑和删除是权限的子集。 一个子权限只能归属于父权限。BookStorePermissions.Books.Default

最后，编辑国际化文件（在项目en.jsonLocalization/BookStoreAcme.BookStore.Domain.Shared 文件夹下）：

```
"Permission:BookStore": "Book Store",
"Permission:Books": "Book Management",
"Permission:Books.Create": "Creating new books",
"Permission:Books.Edit": "Editing the books",
"Permission:Books.Delete": "Deleting the books"

```

```
本地化关键字名称是任意的，并没有强制规定。但是我们还是更喜欢上面的约定模式
```

## 权限管理界面

定义权限后，您可以在权限管理模式下看到他们
定位到 Adninistrator -> Indentity -> Role 页面下，为角色“admin”选择编辑权限如下：

![权限设置](https://img2020.cnblogs.com/blog/630623/202012/630623-20201221150356967-1799851470.png)

授权所需的权限并保存
提示:新权限自动授予如果你运行application.Acme.BookStore.DbMigrator admin角色

# 授权

现在，您能使用权限授权图书管理。

## 应用层 和 Http API

打开类 BookAppService ，然后将策略名称设置为上面定义的权限名称

```
using System;
using Acme.BookStore.Permissions;
using Volo.Abp.Application.Dtos;
using Volo.Abp.Application.Services;
using Volo.Abp.Domain.Repositories;

namespace Acme.BookStore.Books
{
    public class BookAppService :
        CrudAppService<
            Book, //The Book entity
            BookDto, //Used to show books
            Guid, //Primary key of the book entity
            PagedAndSortedResultRequestDto, //Used for paging/sorting
            CreateUpdateBookDto>, //Used to create/update a book
        IBookAppService //implement the IBookAppService
    {
        public BookAppService(IRepository<Book, Guid> repository)
            : base(repository)
        {
            GetPolicyName = BookStorePermissions.Books.Default;
            GetListPolicyName = BookStorePermissions.Books.Default;
            CreatePolicyName = BookStorePermissions.Books.Create;
            UpdatePolicyName = BookStorePermissions.Books.Edit;
            DeletePolicyName = BookStorePermissions.Books.Delete;
        }
    }
}

```

文件：CrudAppService
在构造函数中添加代码。Base 自动都CRUD操作使用这些权限。这将使应用程序更加安全，同时也是HTTP API安全。因为上面所述，该服务自动用于HTTP API(详情见 [auto API controllers](https://docs.abp.io/en/abp/latest/API/Auto-API-Controllers))

在开发管理功能时候，稍后将该属性用于授权 [Authorize(...)]