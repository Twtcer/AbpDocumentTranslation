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

在*.BookStore.Domain 项目中创建一个 Authors 文件夹  ，并添加添加类**Author**：

```C#
using System;
using JetBrains.Annotations;
using Volo.Abp;
using Volo.Abp.Domain.Entities.Auditing;

namespace Acme.BookStore.Authors
{
    public class Author : FullAuditedAggregateRoot<Guid>
    {
        public string Name { get; private set; }
        public DateTime BirthDate { get; set; }
        public string ShortBio { get; set; }

        private Author()
        {
            /* This constructor is for deserialization / ORM purpose */
        }

        internal Author(
            Guid id,
            [NotNull] string name,
            DateTime birthDate,
            [CanBeNull] string shortBio = null)
            : base(id)
        {
            SetName(name);
            BirthDate = birthDate;
            ShortBio = shortBio;
        }

        internal Author ChangeName([NotNull] string name)
        {
            SetName(name);
            return this;
        }

        private void SetName([NotNull] string name)
        {
            Name = Check.NotNullOrWhiteSpace(
                name, 
                nameof(name), 
                maxLength: AuthorConsts.MaxNameLength
            );
        }
    }
}
```

- 继承自 **FullAuditedAggregateaRoot<Guid>**,能够实现实体软删除（标记删除）等审计属性
- Name属性私有设置保证了无法从外部类修改。这有两个方法设置name名称
  - 在构造函数中
  - 使用方法**ChangeName**
- 构造方法和**ChangeName**方法标记为 internal 内部的，用于强制限制在此域AuthorManager中使用。
- **Check**类是ABP框架中的工具类，用于检查方法参数(若检查不过将抛错 ArgumentException)



**AuthorConsts**是一个简单类，位于项目**.BookStore.Domain.Shared**命名空间Authors 下 ：

```c#
namespace Acme.BookStore.Authors
{
    public static class AuthorConsts
    {
        public const int MaxNameLength = 64;
    }
}
```

创建此类将被DTOs（Data Transfer Object） 复用



## AuthorManager : 领域服务

Author类构造函数和**ChangeName** 方法是 internal 类型，所以只能被此域中使用。

在Author文件夹中创建类**AuthorManager** ，并参考下方实现：

```c#
using System;
using System.Threading.Tasks;
using JetBrains.Annotations;
using Volo.Abp;
using Volo.Abp.Domain.Services;

namespace Acme.BookStore.Authors
{
    public class AuthorManager : DomainService
    {
        private readonly IAuthorRepository _authorRepository;

        public AuthorManager(IAuthorRepository authorRepository)
        {
            _authorRepository = authorRepository;
        }

        public async Task<Author> CreateAsync(
            [NotNull] string name,
            DateTime birthDate,
            [CanBeNull] string shortBio = null)
        {
            Check.NotNullOrWhiteSpace(name, nameof(name));

            var existingAuthor = await _authorRepository.FindByNameAsync(name);
            if (existingAuthor != null)
            {
                throw new AuthorAlreadyExistsException(name);
            }

            return new Author(
                GuidGenerator.Create(),
                name,
                birthDate,
                shortBio
            );
        }

        public async Task ChangeNameAsync(
            [NotNull] Author author,
            [NotNull] string newName)
        {
            Check.NotNull(author, nameof(author));
            Check.NotNullOrWhiteSpace(newName, nameof(newName));

            var existingAuthor = await _authorRepository.FindByNameAsync(newName);
            if (existingAuthor != null && existingAuthor.Id != author.Id)
            {
                throw new AuthorAlreadyExistsException(newName);
            }

            author.ChangeName(newName);
        }
    }
}
```



AuthorManger 强制创建作者并以受控方式更改作者的名称，应用程序层后续将使用这些方法。

> DDD小贴士： 不要引入域服务方法，除非确实要它们并执行一些核心业务规则。对于这种情况，我们需要此服务能后强制唯一名称的约束。



两种方法都是检查是否存在具有给定名称的作者，并引发一个特俗的业务异常**AuthorAlreadyExistsException ** ，该错误位于项目*.BookStore.Domain中，如下所示：

```c#
using Volo.Abp;

namespace Acme.BookStore.Authors
{
    public class AuthorAlreadyExistsException : BusinessException
    {
        public AuthorAlreadyExistsException(string name)
            : base(BookStoreDomainErrorCodes.AuthorAlreadyExists)
        {
            WithData("name", name);
        }
    }
}
```

**BussinessException** 是一个特殊的错误类型。在需要时抛出与域相关的异常是一个好习惯。它将由ABP框架自动处理，并且可以轻松地进行本地化。**WithData(...)** 方法用于向异常对象提供其他数据，这些数据以后将在本地化消息上使用或用于其他目的。



在项目***.BookStore.Domain.Shared**中打开 BookStoreDomainErrorCodes,并作如下的更改：

```c#
namespace Acme.BookStore
{
    public static class BookStoreDomainErrorCodes
    {
        public const string AuthorAlreadyExists = "BookStore:00001";
    }
}
```

这有个唯一的字符串，用于表示您的应用程序引发的错误代码，并且可由客户端应用程序处理。对于用户，您可能想对其进行本地化。打开*.BookStore.Domain.Shared项目内的 **Localization/BootStore/en.json**  并添加以下条目：(译者：中文 zh.json中也需要添加)

```javascript
"BookStore:00001": "There is already an author with the same name: {name}"
```

每当您抛出AuthorAlreadyExistsException 时，最终用户将在UI看到此消息。



## IAuthorRepository

**AuthorManager** 注入了**IAuthorRepository** ,所以我们需要对其定义。在Authors文件夹下创建这个新的接口：

```c#
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using Volo.Abp.Domain.Repositories;

namespace Acme.BookStore.Authors
{
    public interface IAuthorRepository : IRepository<Author, Guid>
    {
        Task<Author> FindByNameAsync(string name);

        Task<List<Author>> GetListAsync(
            int skipCount,
            int maxResultCount,
            string sorting,
            string filter = null
        );
    }
}
```

- **IAuthorRepository**扩展自标准的**IRepository<Author,Guid>** 接口，因此所有标准的仓储方法也可以用于 **IAuthorRepository**。
- **AuthorManager**中使用**FindByNameAsync** 通过名称查询作者。
- GetListAsync将在应用层中使用，以获取排序、筛选后的作者列表，以显示在UI中。

我们将在下一部分中现实此仓储



> 由于标准仓储中已经是**IQueryable**,因此这两种方法似乎都是必要的，您可以直接使用他们，而不必定义此类自定义的方法。您说对了，就像在实际应用中一样。但是，对于学习本教程，很有必要解释如何在真正需要的时候创建自定义的仓储方法。



## 总结

这部分涵盖了书店应用程序作者功能的领域层。主要文件如下：

![bookstore-author-domain-layer](https://raw.githubusercontent.com/Twtcer/imgbed/main/picgo/bookstore-author-domain-layer.png)

