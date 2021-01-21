#  Web开发教程9 作者：用户界面

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
- 第八章：应用层
- [**第九章：用户界面（本教程）**]()
- 第十章：预定作者关系



### 下载源码

教程根据您的UI和数据库首选项有多个版本。我们准备了一些可供下载的源码组合：

- MVC(Razor Pages)界面和EFCore版本
- Blazor和EF Core版本
- Angular和MongoDB版本



## 导论

这部分教程说明了怎么为上一步介绍的**Author**实体创建CURD页面。



##  作者列表页面

在项目***.BookStore.Web**文件夹**Pages/Authors**下创建新的razor页面**Index.cshtml**,内容如下：

```c#
@page
@using Acme.BookStore.Localization
@using Acme.BookStore.Permissions
@using Acme.BookStore.Web.Pages.Authors
@using Microsoft.AspNetCore.Authorization
@using Microsoft.Extensions.Localization
@inject IStringLocalizer<BookStoreResource> L
@inject IAuthorizationService AuthorizationService
@model IndexModel

@section scripts
{
    <abp-script src="/Pages/Authors/Index.js"/>
}

<abp-card>
    <abp-card-header>
        <abp-row>
            <abp-column size-md="_6">
                <abp-card-title>@L["Authors"]</abp-card-title>
            </abp-column>
            <abp-column size-md="_6" class="text-right">
                @if (await AuthorizationService
                    .IsGrantedAsync(BookStorePermissions.Authors.Create))
                {
                    <abp-button id="NewAuthorButton"
                                text="@L["NewAuthor"].Value"
                                icon="plus"
                                button-type="Primary"/>
                }
            </abp-column>
        </abp-row>
    </abp-card-header>
    <abp-card-body>
        <abp-table striped-rows="true" id="AuthorsTable"></abp-table>
    </abp-card-body>
</abp-card>
```

这个简单的页面类似于我们之前创建的图书页面。它将导入一个JavaScript文件，下面将对其进行介绍。



### IndexModel.cshtml.cs

```c#
using Microsoft.AspNetCore.Mvc.RazorPages;

namespace Acme.BookStore.Web.Pages.Authors
{
    public class IndexModel : PageModel
    {
        public void OnGet()
        {

        }
    }
}
```



### Index.js

```c#
$(function () {
    var l = abp.localization.getResource('BookStore');
    var createModal = new abp.ModalManager(abp.appPath + 'Authors/CreateModal');
    var editModal = new abp.ModalManager(abp.appPath + 'Authors/EditModal');

    var dataTable = $('#AuthorsTable').DataTable(
        abp.libs.datatables.normalizeConfiguration({
            serverSide: true,
            paging: true,
            order: [[1, "asc"]],
            searching: false,
            scrollX: true,
            ajax: abp.libs.datatables.createAjax(acme.bookStore.authors.author.getList),
            columnDefs: [
                {
                    title: l('Actions'),
                    rowAction: {
                        items:
                            [
                                {
                                    text: l('Edit'),
                                    visible: 
                                        abp.auth.isGranted('BookStore.Authors.Edit'),
                                    action: function (data) {
                                        editModal.open({ id: data.record.id });
                                    }
                                },
                                {
                                    text: l('Delete'),
                                    visible: 
                                        abp.auth.isGranted('BookStore.Authors.Delete'),
                                    confirmMessage: function (data) {
                                        return l(
                                            'AuthorDeletionConfirmationMessage',
                                            data.record.name
                                        );
                                    },
                                    action: function (data) {
                                        acme.bookStore.authors.author
                                            .delete(data.record.id)
                                            .then(function() {
                                                abp.notify.info(
                                                    l('SuccessfullyDeleted')
                                                );
                                                dataTable.ajax.reload();
                                            });
                                    }
                                }
                            ]
                    }
                },
                {
                    title: l('Name'),
                    data: "name"
                },
                {
                    title: l('BirthDate'),
                    data: "birthDate",
                    render: function (data) {
                        return luxon
                            .DateTime
                            .fromISO(data, {
                                locale: abp.localization.currentCulture.name
                            }).toLocaleString();
                    }
                }
            ]
        })
    );

    createModal.onResult(function () {
        dataTable.ajax.reload();
    });

    editModal.onResult(function () {
        dataTable.ajax.reload();
    });

    $('#NewAuthorButton').click(function (e) {
        e.preventDefault();
        createModal.open();
    });
});
```

简而言之，这个JavaScript页面：

- 为**操作，名称**和**出生生日**列创建一个数据表
  - **操作**列被用于添加编辑和删除操作
  - 生日提供了一个渲染函数，用于使用luxon库格式化DateTime值
- 使用**abp.ModalManager**打开**创建**和**编辑**模态表单

该代码与之前创建的图书页面非常相似。因此我们将不再对其进行详细的说明。



### 本地化

该页面使用一些我们需要先定义本地化关键字。在项目***.BootStore.Domain.Shared**文件夹**Localization/BookStore**打开**en.json**文件，并添加已下条目：

```c#
"Menu:Authors": "Authors",
"Authors": "Authors",
"AuthorDeletionConfirmationMessage": "Are you sure to delete the author '{0}'?",
"BirthDate": "Birth date",
"NewAuthor": "New author"
```

注意。我们添加更多键。它将在下一部分中使用。



### 添加作者主菜单

在项目***.BookStore.Web**文件夹**Menu**下打开文件**BookStoreMenuContributor.cs**，然后添加以下代码到方法**ConfigureMainMenuAsync**结尾部分：

```c#
if (await context.IsGrantedAsync(BookStorePermissions.Authors.Default))
{
    bookStoreMenu.AddItem(new ApplicationMenuItem(
        "BooksStore.Authors",
        l["Menu:Authors"],
        url: "/Authors"
    ));
}
```



### 运行应用程序

运行并登录应用程序。您尚未获得权限，因此无法看到菜单项。撞到**身份/角色**页面，单价**操作**按钮，然后为管理员角色选择**权限**操作：

![bookstore-author-permissions](https://raw.githubusercontent.com/Twtcer/imgbed/main/picgo/bookstore-author-permissions.png)

如你所见，admin角色还没有没有**Aurthor Management** 权限。单击复选框，然后保存模式以授权必要的权限。刷新页面后，您将主菜单中的书店下看到



### AuthorAppService

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
- 使用**AuthorManager**（域服务）构建新作者。
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
- 使用**IAuthorRepository.GetAsync**从数据库中获取作者实体。如果不存在给定的ID的作者，则**GetAsync**引发**EntityNotFoundException**，这将在Web应用程序中导致404 HTTP状态码。始终使实体处于更新操作是一个好习惯。
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

在项目***.BookStore.Application.Contracts**文件夹**Permissions**下修改类BookStorePermissions如下：

```c#
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
        
        // *** ADDED a NEW NESTED CLASS ***
        public static class Authors
        {
            public const string Default = GroupName + ".Authors";
            public const string Create = Default + ".Create";
            public const string Edit = Default + ".Edit";
            public const string Delete = Default + ".Delete";
        }
    }
}
```

然后在同项目中打开**BookStorePermissionDefinitionProvider** ，在结尾处添加方法**Define**:

```c#
var authorsPermission = bookStoreGroup.AddPermission(
    BookStorePermissions.Authors.Default, L("Permission:Authors"));

authorsPermission.AddChild(
    BookStorePermissions.Authors.Create, L("Permission:Authors.Create"));

authorsPermission.AddChild(
    BookStorePermissions.Authors.Edit, L("Permission:Authors.Edit"));

authorsPermission.AddChild(
    BookStorePermissions.Authors.Delete, L("Permission:Authors.Delete"));
```

最后，在项目***.BookStore.Domian.Share**找到文件 **Localization/BookStore/en.json**,添加如下权限条目：

```
"Permission:Authors": "Author Management",
"Permission:Authors.Create": "Creating new authors",
"Permission:Authors.Edit": "Editing the authors",
"Permission:Authors.Delete": "Deleting the authors"
```



## 对象到对象映射

**AuthorAppService ** 使用**ObjectManager**将**Author**对象转化为**AuthoDto**对象。所以我们需要在**AutoMapper**中定义映射关系。

```
CreateMap<Author, AuthorDto>();
```



## Data Seeder (数据初始种子)

就像之前的书籍一样，最好在数据库中有一些初始作者实体。第一次运行该应用程序时候，这会很好，但对于自动化测试而言，它非常有用。

打开项目***.BootStore.Domian**中**BookStoreDataSeederContributor**,作如下更改：

```c#
using System;
using System.Threading.Tasks;
using Acme.BookStore.Authors;
using Acme.BookStore.Books;
using Volo.Abp.Data;
using Volo.Abp.DependencyInjection;
using Volo.Abp.Domain.Repositories;

namespace Acme.BookStore
{
    public class BookStoreDataSeederContributor
        : IDataSeedContributor, ITransientDependency
    {
        private readonly IRepository<Book, Guid> _bookRepository;
        private readonly IAuthorRepository _authorRepository;
        private readonly AuthorManager _authorManager;

        public BookStoreDataSeederContributor(
            IRepository<Book, Guid> bookRepository,
            IAuthorRepository authorRepository,
            AuthorManager authorManager)
        {
            _bookRepository = bookRepository;
            _authorRepository = authorRepository;
            _authorManager = authorManager;
        }

        public async Task SeedAsync(DataSeedContext context)
        {
            if (await _bookRepository.GetCountAsync() <= 0)
            {
                await _bookRepository.InsertAsync(
                    new Book
                    {
                        Name = "1984",
                        Type = BookType.Dystopia,
                        PublishDate = new DateTime(1949, 6, 8),
                        Price = 19.84f
                    },
                    autoSave: true
                );

                await _bookRepository.InsertAsync(
                    new Book
                    {
                        Name = "The Hitchhiker's Guide to the Galaxy",
                        Type = BookType.ScienceFiction,
                        PublishDate = new DateTime(1995, 9, 27),
                        Price = 42.0f
                    },
                    autoSave: true
                );
            }

            // ADDED SEED DATA FOR AUTHORS

            if (await _authorRepository.GetCountAsync() <= 0)
            {
                await _authorRepository.InsertAsync(
                    await _authorManager.CreateAsync(
                        "George Orwell",
                        new DateTime(1903, 06, 25),
                        "Orwell produced literary criticism and poetry, fiction and polemical journalism; and is best known for the allegorical novella Animal Farm (1945) and the dystopian novel Nineteen Eighty-Four (1949)."
                    )
                );

                await _authorRepository.InsertAsync(
                    await _authorManager.CreateAsync(
                        "Douglas Adams",
                        new DateTime(1952, 03, 11),
                        "Douglas Adams was an English author, screenwriter, essayist, humorist, satirist and dramatist. Adams was an advocate for environmentalism and conservation, a lover of fast cars, technological innovation and the Apple Macintosh, and a self-proclaimed 'radical atheist'."
                    )
                );
            }
        }
    }
}
```

你现在可以运行**.DbMigrator**控制台程序以迁移数据库架构并未初始数据添加种子。



## 测试Author服务

最后，我们将为**IAuthorAppService**写一些测试。在项目***.BookStore.Applcation.Tests**文件夹**Authors**下添加新类并命名为**AuthorAppService_Tests**：

```c#
using System;
using System.Threading.Tasks;
using Shouldly;
using Xunit;

namespace Acme.BookStore.Authors
{ 
    public class AuthorAppService_Tests : BookStoreApplicationTestBase
    {
        private readonly IAuthorAppService _authorAppService;

        public AuthorAppService_Tests()
        {
            _authorAppService = GetRequiredService<IAuthorAppService>();
        }

        [Fact]
        public async Task Should_Get_All_Authors_Without_Any_Filter()
        {
            var result = await _authorAppService.GetListAsync(new GetAuthorListDto());

            result.TotalCount.ShouldBeGreaterThanOrEqualTo(2);
            result.Items.ShouldContain(author => author.Name == "George Orwell");
            result.Items.ShouldContain(author => author.Name == "Douglas Adams");
        }

        [Fact]
        public async Task Should_Get_Filtered_Authors()
        {
            var result = await _authorAppService.GetListAsync(
                new GetAuthorListDto {Filter = "George"});

            result.TotalCount.ShouldBeGreaterThanOrEqualTo(1);
            result.Items.ShouldContain(author => author.Name == "George Orwell");
            result.Items.ShouldNotContain(author => author.Name == "Douglas Adams");
        }

        [Fact]
        public async Task Should_Create_A_New_Author()
        {
            var authorDto = await _authorAppService.CreateAsync(
                new CreateAuthorDto
                {
                    Name = "Edward Bellamy",
                    BirthDate = new DateTime(1850, 05, 22),
                    ShortBio = "Edward Bellamy was an American author..."
                }
            );
            
            authorDto.Id.ShouldNotBe(Guid.Empty);
            authorDto.Name.ShouldBe("Edward Bellamy");
        }

        [Fact]
        public async Task Should_Not_Allow_To_Create_Duplicate_Author()
        {
            await Assert.ThrowsAsync<AuthorAlreadyExistsException>(async () =>
            {
                await _authorAppService.CreateAsync(
                    new CreateAuthorDto
                    {
                        Name = "Douglas Adams",
                        BirthDate = DateTime.Now,
                        ShortBio = "..."
                    }
                );
            });
        }

        //TODO: Test other methods...
    }
}
```

为应用程序服务方法创建一些测试，这些测试应该很容易理解。

