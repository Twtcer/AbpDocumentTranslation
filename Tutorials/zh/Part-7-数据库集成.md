#  Web开发教程7 作者：数据库集成

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
- **[第七章：数据库集成（本教程）](https://www.cnblogs.com/LandWind/p/Web_Application_Development_Tutorial_Part5_Authorization.html)**
- 第八章：应用层
- 第九章：用户接口
- 第十章：预定作者关系



## 下载源码

教程根据您的UI和数据库首选项有多个版本。我们准备了一些可供下载的源码组合：

- MVC(Razor Pages)界面和EFCore版本
- Blazor和EF Core版本
- Angular和MongoDB版本



## 导论

这部分教程说明了怎么为上一步介绍的**Author**实体配置数据库集成 。



## 数据库上下文对象

在项目 ***.BookStore.EntityFrameworkCore** 中打开文件**BookStoreDbCOntext** ，添加下方的**DbSet**属性

```C#
public DbSet<Author> Authors {get;set;}
```

然后打开类 **BookStoreDbContextModelCreatingExtentions**  修改其方法 ConfigureBookStore 内容：

```C#
builder.Entity<Author>(b =>
{
    b.ToTable(BookStoreConsts.DbTablePrefix + "Authors",
        BookStoreConsts.DbSchema);
    
    b.ConfigureByConvention();
    
    b.Property(x => x.Name)
        .IsRequired()
        .HasMaxLength(AuthorConsts.MaxNameLength);

    b.HasIndex(x => x.Name);
});
```

这个和之前的Book实体的操作类似，只是做相应的配置



## 创建新的数据库迁移

打开包管理器控制台（**Package Manager Console** ）,

然后设置默认项目为 ***.BookStore.EntityFrameworkCore.DbMigrations**

另外需要设置默认项目为***.BookStore.Web(或HttpApi.Host)**

![image-20201224111200445](https://raw.githubusercontent.com/Twtcer/imgbed/main/picgo/image-20201224111200445.png)



然后运行迁移命令

```cmd
Add_Migration "Added_Authors"
```

![image-20201224111945993](https://raw.githubusercontent.com/Twtcer/imgbed/main/picgo/image-20201224111945993.png)

这个操作将创建一个新的迁移类，然后运行命令 **Update-Database** 在数据库中创建新表

![image-20201224112530097](https://raw.githubusercontent.com/Twtcer/imgbed/main/picgo/image-20201224112530097.png)



## 实现IAuthorRepository接口

在项目 *.**BookStore.EntityFrameworkCore** 创建名为 **EfCoreAuthorRepository** 的类,然后参考下方的实现：

```C# 
using System;
using System.Collections.Generic;
using System.Linq;
using System.Linq.Dynamic.Core;
using System.Threading.Tasks;
using Acme.BookStore.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;
using Volo.Abp.Domain.Repositories.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore;

namespace Acme.BookStore.Authors
{
    public class EfCoreAuthorRepository
        : EfCoreRepository<BookStoreDbContext, Author, Guid>,
            IAuthorRepository
    {
        public EfCoreAuthorRepository(
            IDbContextProvider<BookStoreDbContext> dbContextProvider)
            : base(dbContextProvider)
        {
        }

        public async Task<Author> FindByNameAsync(string name)
        {
            return await DbSet.FirstOrDefaultAsync(author => author.Name == name);
        }

        public async Task<List<Author>> GetListAsync(
            int skipCount,
            int maxResultCount,
            string sorting,
            string filter = null)
        {
            return await DbSet
                .WhereIf(
                    !filter.IsNullOrWhiteSpace(),
                    author => author.Name.Contains(filter)
                 )
                .OrderBy(sorting)
                .Skip(skipCount)
                .Take(maxResultCount)
                .ToListAsync();
        }
    }
}
```

- 继承自**EfCoreAuthorRepository**  ，因此继承了标准仓储方法的实现
- **WhereIf** 是ABP框架的扩展方法，仅在第一个条件满足下才添加Where条件 (译者：Freesql也有类似的设计)
- **sorting**  可以是 Name`, `Name ASC` or `Name DESC。基于包[System.Linq.Dynamic.Core](https://www.nuget.org/packages/System.Linq.Dynamic.Core) 方法





