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



如你所见，admin角色还没有没有**Aurthor Management** 权限。单击复选框，然后保存模式以授权必要的权限。刷新页面后，您将主菜单中的书店下看到“作者”菜单项：

![bookstore-authors-page](https://raw.githubusercontent.com/Twtcer/imgbed/main/picgo/bookstore-authors-page.png)

页面可以正常工作，除了“新作者”和操作“编辑“，我们尚未实现他们。

> 提示：如果定义新权限后运行**.DbMigrator** 控制台程序，它将自动向管理员角色授予这些新权限，并且您无需自己手动授予权限。



## 创建页面

在项目 ***.BookStore.Web** 文件夹下 **Pages/Authors** 创建一个新的页面CreateModal.cshtml ，然后更改内容如下：

###  CreateModal.cshtml

```c#
@page
@using Acme.BookStore.Localization
@using Acme.BookStore.Web.Pages.Authors
@using Microsoft.Extensions.Localization
@using Volo.Abp.AspNetCore.Mvc.UI.Bootstrap.TagHelpers.Modal
@model CreateModalModel
@inject IStringLocalizer<BookStoreResource> L
@{
    Layout = null;
}
<form asp-page="/Authors/CreateModal">
    <abp-modal>
        <abp-modal-header title="@L["NewAuthor"].Value"></abp-modal-header>
        <abp-modal-body>
            <abp-input asp-for="Author.Name" />
            <abp-input asp-for="Author.BirthDate" />
            <abp-input asp-for="Author.ShortBio" />
        </abp-modal-body>
        <abp-modal-footer buttons="@(AbpModalButtons.Cancel|AbpModalButtons.Save)"></abp-modal-footer>
    </abp-modal>
</form>
```

之前我们使用abp框架动态表单构建图书页面，我们可以在此处使用相同的方法，但是我们想展示如何手动进行。实际上，不是手动操作，因为在这种情况下，我们使用abp-input标签程序来简化表单元素的创建。



### CreateModel.cshtml.cs

```html
using System;
using System.ComponentModel.DataAnnotations;
using System.Threading.Tasks;
using Acme.BookStore.Authors;
using Microsoft.AspNetCore.Mvc;
using Volo.Abp.AspNetCore.Mvc.UI.Bootstrap.TagHelpers.Form;

namespace Acme.BookStore.Web.Pages.Authors
{
    public class CreateModalModel : BookStorePageModel
    {
        [BindProperty]
        public CreateAuthorViewModel Author { get; set; }

        private readonly IAuthorAppService _authorAppService;

        public CreateModalModel(IAuthorAppService authorAppService)
        {
            _authorAppService = authorAppService;
        }

        public void OnGet()
        {
            Author = new CreateAuthorViewModel();
        }

        public async Task<IActionResult> OnPostAsync()
        {
            var dto = ObjectMapper.Map<CreateAuthorViewModel, CreateAuthorDto>(Author);
            await _authorAppService.CreateAsync(dto);
            return NoContent();
        }

        public class CreateAuthorViewModel
        {
            [Required]
            [StringLength(AuthorConsts.MaxNameLength)]
            public string Name { get; set; }

            [Required]
            [DataType(DataType.Date)]
            public DateTime BirthDate { get; set; }

            [TextArea]
            public string ShortBio { get; set; }
        }
    }
}
```

此页面模型类仅注入并使用IAuthorAppService创建新作者。书籍创建模型类之间的主要区别在于该类为模型声明一个新类CreateAuthorViewModel,而不是重新使用CreateAuthorDto。

做出此决定的主要原因是向您展示如何在页面内使用其他的模型类。

但是还有一个好处：我们向成员类中添加两个属性，这些属性在CreateAuthorDto不存在：

- 添加 **[DataType(DataType.Date)]** 标签到属性**BirthDate**，该属性用于在UI上显示此属性的日期选择器。
- 添加**[TeatArea]**标签到属性**ShortBio**，该属性将展示一个多行的文本编辑器替代默认的文本框。 



这样，你可以基于UI需求来专门化视图模型类，而无需接触DTO。作为此决定的结果，我们使用ObjectMapper将**CreateAuthorViewModel**映射到**CreateAuthorDto**。为此，你选哟向BookStoreWebAutoMapperProfie构造函数添加新的映射代码：

```C#
using Acme.BookStore.Authors; // ADDED NAMESPACE IMPORT
using Acme.BookStore.Books;
using AutoMapper;

namespace Acme.BookStore.Web
{
    public class BookStoreWebAutoMapperProfile : Profile
    {
        public BookStoreWebAutoMapperProfile()
        {
            CreateMap<BookDto, CreateUpdateBookDto>();

            // ADD a NEW MAPPING
            CreateMap<Pages.Authors.CreateModalModel.CreateAuthorViewModel,
                      CreateAuthorDto>();
        }
    }
}
```



当你再次运行该应用程序时，"新建作者"按钮将按预期工作并打开一个新模型：

![bookstore-new-author-modal](https://raw.githubusercontent.com/Twtcer/imgbed/main/picgo/bookstore-new-author-modal.png)



## 编辑页面

在* .BookStore.Web项目Pages/Authors文件夹下创建新的页面EditModal.cshtml，然后更改内容，如下所示：

```razor
@page
@using Acme.BookStore.Localization
@using Acme.BookStore.Web.Pages.Authors
@using Microsoft.Extensions.Localization
@using Volo.Abp.AspNetCore.Mvc.UI.Bootstrap.TagHelpers.Modal
@model EditModalModel
@inject IStringLocalizer<BookStoreResource> L
@{
    Layout = null;
}
<form asp-page="/Authors/EditModal">
    <abp-modal>
        <abp-modal-header title="@L["Update"].Value"></abp-modal-header>
        <abp-modal-body>
            <abp-input asp-for="Author.Id" />
            <abp-input asp-for="Author.Name" />
            <abp-input asp-for="Author.BirthDate" />
            <abp-input asp-for="Author.ShortBio" />
        </abp-modal-body>
        <abp-modal-footer buttons="@(AbpModalButtons.Cancel|AbpModalButtons.Save)"></abp-modal-footer>
    </abp-modal>
</form>
```



###  EditModal.cshtml.cs

```c#
using System;
using System.ComponentModel.DataAnnotations;
using System.Threading.Tasks;
using Acme.BookStore.Authors;
using Microsoft.AspNetCore.Mvc;
using Volo.Abp.AspNetCore.Mvc.UI.Bootstrap.TagHelpers.Form;

namespace Acme.BookStore.Web.Pages.Authors
{
    public class EditModalModel : BookStorePageModel
    {
        [BindProperty]
        public EditAuthorViewModel Author { get; set; }

        private readonly IAuthorAppService _authorAppService;

        public EditModalModel(IAuthorAppService authorAppService)
        {
            _authorAppService = authorAppService;
        }

        public async Task OnGetAsync(Guid id)
        {
            var authorDto = await _authorAppService.GetAsync(id);
            Author = ObjectMapper.Map<AuthorDto, EditAuthorViewModel>(authorDto);
        }

        public async Task<IActionResult> OnPostAsync()
        {
            await _authorAppService.UpdateAsync(
                Author.Id,
                ObjectMapper.Map<EditAuthorViewModel, UpdateAuthorDto>(Author)
            );

            return NoContent();
        }

        public class EditAuthorViewModel
        {
            [HiddenInput]
            public Guid Id { get; set; }

            [Required]
            [StringLength(AuthorConsts.MaxNameLength)]
            public string Name { get; set; }

            [Required]
            [DataType(DataType.Date)]
            public DateTime BirthDate { get; set; }

            [TextArea]
            public string ShortBio { get; set; }
        }
    }
}
```

这个类似CreateModal.cshtml.cs，但是有一些区别：

- 使用IAuthorAppService.GetAsync()方法从应用程序层获取编辑作者。
- EditAurhorViewModal具有一个附加的Id属性，该属性用[HiddenInput]标记，该标签为此属性创建隐藏的输入。



此类要向BookStoreWebAutoMapperProfile类添加两个对象映射声明：

```c#
using Acme.BookStore.Authors;
using Acme.BookStore.Books;
using AutoMapper;

namespace Acme.BookStore.Web
{
    public class BookStoreWebAutoMapperProfile : Profile
    {
        public BookStoreWebAutoMapperProfile()
        {
            CreateMap<BookDto, CreateUpdateBookDto>();

            CreateMap<Pages.Authors.CreateModalModel.CreateAuthorViewModel,
                      CreateAuthorDto>();

            // ADD THESE NEW MAPPINGS
            CreateMap<AuthorDto, Pages.Authors.EditModalModel.EditAuthorViewModel>();
            CreateMap<Pages.Authors.EditModalModel.EditAuthorViewModel,
                      UpdateAuthorDto>();
        }
    }
}
```

通过以上你可以运行程序，然后尝试编辑作者。



## 下一部分

[详见下一部分教程]()