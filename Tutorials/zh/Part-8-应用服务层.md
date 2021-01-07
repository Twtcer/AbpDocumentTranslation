#  Web开发教程8 作者：应用层

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
- 第六章：领域层
- 第七章：数据库集成
- **[第八章：应用层（本教程）]()**
- 第九章：用户接口
- 第十章：预定作者关系



## 下载源码

教程根据您的UI和数据库首选项有多个版本。我们准备了一些可供下载的源码组合：

- MVC(Razor Pages)界面和EFCore版本
- Blazor和EF Core版本
- Angular和MongoDB版本



## 导论

这部分教程说明了怎么为上一步介绍的**Author**实体创建应用层。



##  IAuthorAppService

我们将首先创建应用服务接口和相关的DTOs。在项目***.BookStore.Application.Contracts**创建一个名为**IAuthorAppService**新接口，代码如下：

```c#
using System;
using System.Threading.Tasks;
using Volo.Abp.Application.Dtos;
using Volo.Abp.Application.Services;

namespace Acme.BookStore.Authors
{
    public interface IAuthorAppService : IApplicationService
    {
        Task<AuthorDto> GetAsync(Guid id);

        Task<PagedResultDto<AuthorDto>> GetListAsync(GetAuthorListDto input);

        Task<AuthorDto> CreateAsync(CreateAuthorDto input);

        Task UpdateAsync(Guid id, UpdateAuthorDto input);

        Task DeleteAsync(Guid id);
    }
}
```

- **IApplicationService** 是由所有应用程序服务集成的常规接口，因此ABP框架可以识别该服务。
- 定义的标准方法以在**Author**实体上执行CRUD操作。
- **PageResultDto**  是ABP框架中预定义DTO类。它具有**Items** 集合和**TotalCount**属性即返回分页结果。
- 首先从**CreateAsync**方法返回AuthorDto（对于新建的作者），此应用程序不使用它-只是为了显示不同的方法。

该界面使用下面定义的DTO（为您的项目创建他们）



## AuthorDto

```c#
using System;
using Volo.Abp.Application.Dtos;

namespace Acme.BookStore.Authors
{
    public class AuthorDto : EntityDto<Guid>
    {
        public string Name { get; set; }

        public DateTime BirthDate { get; set; }

        public string ShortBio { get; set; }
    }
}
```

- **EntityDto<T>** 仅具有给定泛型参数的Id属性。您可以自己创建一个Id属性，而不是继承EntityDto。



## GetAuthorListDto

```c#
using Volo.Abp.Application.Dtos;

namespace Acme.BookStore.Authors
{
    public class GetAuthorListDto : PagedAndSortedResultRequestDto
    {
        public string Filter { get; set; }
    }
}
```

- **Filter**被用于搜索作者。它可为**null**对象（或者string空）以获得所有作者。
- **PagedAndSortedResultRequestDto** 具有标准的分页和排序属性：**int MaxResultCount, int SkipCount, String Sorting** 。



> ABP框架具有基本的DTO类，以简化和标准化您的DTO。有关信息，参考DTO文档。



## CreateAuthorDto

```c#
using System;
using System.ComponentModel.DataAnnotations;

namespace Acme.BookStore.Authors
{
    public class CreateAuthorDto
    {
        [Required]
        [StringLength(AuthorConsts.MaxNameLength)]
        public string Name { get; set; }

        [Required]
        public DateTime BirthDate { get; set; }
        
        public string ShortBio { get; set; }
    }
}
```

数据注解属性能够用于校验DTO对象。详见[校验文档](https://docs.abp.io/en/abp/latest/Validation)。



## UpdateAuthorDto

```c#
using System;
using System.ComponentModel.DataAnnotations;

namespace Acme.BookStore.Authors
{
    public class UpdateAuthorDto
    {
        [Required]
        [StringLength(AuthorConsts.MaxNameLength)]
        public string Name { get; set; }

        [Required]
        public DateTime BirthDate { get; set; }
        
        public string ShortBio { get; set; }
    }
}
```

> 我们可以在创建和更新操作之间共享(重用)相同的DTO。尽管可以做到，但是我们更愿意为这些操作创建不同的Dto，因为我们看它们通常会有有所不同。因此，与紧密耦合的设计相比，这里的代码复制是合理的。



## AuthorAppService

现在是实现IAuthorAppService接口的时候了。在***.BookStore.Application**项目的Authors文件夹下创建一个名为**AuthorAppService**新类：

```c#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Acme.BookStore.Permissions;
using Microsoft.AspNetCore.Authorization;
using Volo.Abp.Application.Dtos;

namespace Acme.BookStore.Authors
{
    [Authorize(BookStorePermissions.Authors.Default)]
    public class AuthorAppService : BookStoreAppService, IAuthorAppService
    {
        private readonly IAuthorRepository _authorRepository;
        private readonly AuthorManager _authorManager;

        public AuthorAppService(
            IAuthorRepository authorRepository,
            AuthorManager authorManager)
        {
            _authorRepository = authorRepository;
            _authorManager = authorManager;
        }

        //...SERVICE METHODS WILL COME HERE...
    }
}
```

- **[Authorize(BookStorePermission.Authors.Default)]**是检查权限(策略)以授权当前用户的声明方式。有关更多的信息，参考授权文档。BookStorePermission类将在下面更新，现在不用担心编译错误。
- 派生自**BookStoreAppService**,该模板是启动模板附带的一个简单基类。它是从标准**ApplcationService**类派生的。
- 现实上面定义的**IAuthorAppService**
- 注入**IAuthorRepository**和**AuthorManager**便于服务方法中使用



## GetAsync

```c#
public async Task<AuthorDto> GetAsync(Guid id)
{
    var author = await _authorRepository.GetAsync(id);
    return ObjectMapper.Map<Author, AuthorDto>(author);
}
```

这个方法仅通过其**Id**获取Author实体，并使用对象到对象的映射器将其转化为AuthorDto。这个需要提前配置AutoMapper，这将在后面说明。



## GetListAsync

```c#
public async Task<PagedResultDto<AuthorDto>> GetListAsync(GetAuthorListDto input)
{
    if (input.Sorting.IsNullOrWhiteSpace())
    {
        input.Sorting = nameof(Author.Name);
    }

    var authors = await _authorRepository.GetListAsync(
        input.SkipCount,
        input.MaxResultCount,
        input.Sorting,
        input.Filter
    );

    var totalCount = await AsyncExecuter.CountAsync(
        _authorRepository.WhereIf(
            !input.Filter.IsNullOrWhiteSpace(),
            author => author.Name.Contains(input.Filter)
        )
    );

    return new PagedResultDto<AuthorDto>(
        totalCount,
        ObjectMapper.Map<List<Author>, List<AuthorDto>>(authors)
    );
}
```

- 默认排序条件是"按作者名称",如果客户端未发送，则在方法的开头进行排序。
- 使用IAuthorRepository.GetListAsync从数据库中获取作者的分页，排序和筛选列表。我们已经在本教程的前一部分实现了它。同样，实际上并不需要创建这样的方法，因为我们可以直接查询存储库，但想演示如何创建自定义的存储方法。
- 直接从AuthorRepository查询，同时获取作者的人数。我们更喜欢使用AsyncExecuter服务，该服务允许我们执行异步查询而不需依赖EF Core。但是，您可以依赖EF Core包并直接使用_authorRepository.WhereIf(...).ToLIstAsync()方法。详情参阅存储库文档以阅读替代方法和讨论。
- 最后，我们通过Authors列表映射到AuthorDtos列表来返回分页结果。



## CreateAsync

```c#
[Authorize(BookStorePermissions.Authors.Create)]
public async Task<AuthorDto> CreateAsync(CreateAuthorDto input)
{
    var author = await _authorManager.CreateAsync(
        input.Name,
        input.BirthDate,
        input.ShortBio
    );

    await _authorRepository.InsertAsync(author);

    return ObjectMapper.Map<Author, AuthorDto>(author);
}
```

- **CreateAsync**需要BookStorePermissions.Authors.Create权限(除了为AuthorAppService类声明的BookStorePermissions.Authors.Default除外)。
-  使用**AuthorManager**（域服务）构建新作者。
- 使用**IAuthorRepository.InsertAsync**插入新作者到数据库中。
- 使用**ObjectManager**返回代表新创建作者的**AuthorDto**。

> DDD提示：一些开发人员可能会发现将新实体插入_authorManager.CreateAsync很有用。我们认为将其留个应用程序是一个和好的设计，因为它更好地知道何时将其插入数据库(也许在插入之前它需要对实体进行其他的工作，而如果我们在数据库中执行插入操作，则需要在域服务中进行其他更新。当然，这个取决于您。)



## UpdateAsync

```c#
[Authorize(BookStorePermissions.Authors.Edit)]
public async Task UpdateAsync(Guid id, UpdateAuthorDto input)
{
    var author = await _authorRepository.GetAsync(id);

    if (author.Name != input.Name)
    {
        await _authorManager.ChangeNameAsync(author, input.Name);
    }

    author.BirthDate = input.BirthDate;
    author.ShortBio = input.ShortBio;

    await _authorRepository.UpdateAsync(author);
}
```

- **UpdateAsync**需要BookStorePermissions.Authors.Edit权限。
-  使用**IAuthorRepository.GetAsync**从数据库中获取作者实体。如果不存在给定的ID的作者，则**GetAsync**引发**EntityNotFoundException**，这将在Web应用程序中导致404 HTTP状态码。始终使实体处于更新操作是一个好习惯。
- 如果客户端请求更改作者名称，则使用**AuthorManager.ChangeNameAsync**(域服务方法)更新作者名称。
- 由于没有更改这些熟悉那个的任何业务规则，因此直接更新了BirthDate和ShortBio，他们接受任何值。
- 最后，调用IAuthorRepositroy.UpdateAsync方法以更新数据库上的实体。

> EF Core提示：Entity Framework Core 具有变更跟踪系统，并在工作单元结束时自动将任何变更保存到实体（您可以简单地认为ABP Framework在方法结束时才会自动 调用**SaveChanges**）。因此，即使您未在方法末尾调用_authorRepository.UpdateAsync(...)，它也将按预期工作。如果您不考虑以后再更改EF Core，则只需要删除此行即可



## DeleteAsync

```c#
[Authorize(BookStorePermissions.Authors.Delete)]
public async Task DeleteAsync(Guid id)
{
    await _authorRepository.DeleteAsync(id);
}
```

- **DeleteAsync**要求添加**BookStorePermissions.Authors.Delete**权限。
- 它仅使用存储库的**DeleteAsync**方法。



## 权限定义

您无法编译代码，因为它需要在BookStorePermissions类中声明一些常量。



