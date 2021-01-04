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

> 我们能够共享(重用)相同的DTO

