# Blazor Server 快速開發專案樣板 6 - 動態功能表

![Blazor Server](../Images/x052.png)

[Blazor Server 快速開發專案樣板 - 相關系列文章清單](https://csharpkh.blogspot.com/2021/06/Blazor-Backend-project-template-syncfusion-NET5.html)

上一篇的文章 : [Blazor Server 快速開發專案樣板 5 - 使用者身分驗證與授權](https://csharpkh.blogspot.com/2021/06/Blazor-Backend-project-template-syncfusion-NET5-Part5-Cookie-Authentication-Authorization-CAPTCHA-Claim-Identity-SignInAsync.html)

這一篇文章所討論到的需求，是大多數 Blazor 開發者想要知道開發技能，但是，當使用 Visual Studio 建立 [Blazor Server](https://docs.microsoft.com/zh-tw/aspnet/core/blazor/hosting-models?view=aspnetcore-5.0&WT.mc_id=DT-MVP-5002220) 專案的時候，這個預設專案範本僅提供的靜態功能表的設計方式，因此，在這篇文章將會說明如何設計出這樣的功能。

在這個 [Blazor Server 快速開發專案樣板] 中，將會設計三種功能表角色 (當然，開發者可以根據本身不同需求，自行擴增)，分別是 [開發者角色]、[系統管理員角色]、[使用者角色]，若使用不同的使用者登入 ( god , admin , user1 而密碼皆為 123 )，將會看到不同的功能表清單。

* 具有 [開發者角色] 可以看到的功能表清單

  ![](../Images/x087.png)

* 具有 [系統管理員角色] 可以看到的功能表清單

  ![](../Images/x088.png)

* 具有 [使用者角色] 可以看到的功能表清單
 
  ![](../Images/x089.png)

## 資料庫需要設計的資料表

對於想要設計這樣的功能，會將不同功能表角色與相關功能表項目清單資訊儲存在資料表內，在這個 [Blazor Server 快速開發專案樣板] 內，將會設計底下資料表關係

![](../Images/x090.png)

其中

* MyUser

  這裡儲存每個使用者的紀錄

* MenuRole

  這裡是儲存每個使用者可以綁定的功能表角色紀錄

* MenuData

  每個功能表角色紀錄所對應的功能表項目，都會儲存在這個資料表內

上面的三個資料表的 SQL 定義如下

* MyUser

```SQL
CREATE TABLE [dbo].[MyUser] (
    [Id]         INT            IDENTITY (1, 1) NOT NULL,
    [Account]    NVARCHAR (MAX) NOT NULL,
    [Password]   NVARCHAR (MAX) NOT NULL,
    [Name]       NVARCHAR (MAX) NOT NULL,
    [Salt]       NVARCHAR (MAX) NULL,
    [Status]     BIT            NOT NULL,
    [MenuRoleId] INT            NOT NULL,
    CONSTRAINT [PK_MyUser] PRIMARY KEY CLUSTERED ([Id] ASC),
    CONSTRAINT [FK_MyUser_MenuRole_MenuRoleId] FOREIGN KEY ([MenuRoleId]) REFERENCES [dbo].[MenuRole] ([Id])
);


GO
CREATE NONCLUSTERED INDEX [IX_MyUser_MenuRoleId]
    ON [dbo].[MyUser]([MenuRoleId] ASC);
```

* MenuRole

```SQL
CREATE TABLE [dbo].[MenuRole] (
    [Id]     INT            IDENTITY (1, 1) NOT NULL,
    [Name]   NVARCHAR (MAX) NOT NULL,
    [Remark] NVARCHAR (MAX) NULL,
    CONSTRAINT [PK_MenuRole] PRIMARY KEY CLUSTERED ([Id] ASC)
);
```

* MenuData

```SQL
CREATE TABLE [dbo].[MenuData] (
    [Id]         INT            IDENTITY (1, 1) NOT NULL,
    [Name]       NVARCHAR (MAX) NOT NULL,
    [Level]      INT            NOT NULL,
    [IsGroup]    BIT            NOT NULL,
    [Sequence]   INT            NOT NULL,
    [Icon]       NVARCHAR (MAX) NULL,
    [CodeName]   NVARCHAR (MAX) NOT NULL,
    [Enable]     BIT            NOT NULL,
    [ForceLoad]  BIT            NOT NULL,
    [MenuRoleId] INT            NOT NULL,
    CONSTRAINT [PK_MenuData] PRIMARY KEY CLUSTERED ([Id] ASC),
    CONSTRAINT [FK_MenuData_MenuRole_MenuRoleId] FOREIGN KEY ([MenuRoleId]) REFERENCES [dbo].[MenuRole] ([Id])
);


GO
CREATE NONCLUSTERED INDEX [IX_MenuData_MenuRoleId]
    ON [dbo].[MenuData]([MenuRoleId] ASC);
```

## 設計功能表清單 UI

Visual Studio 預設的 Blazor 專案範本會在 [Shared] 資料夾內建立一個 [NavMenu.razor] 檔案，這是一個 [Razor 元件]，這裡將會使用靜態的方式來定義這個系統可以使用的功能表項目。

而在這個 [NavMenu.razor] 元件將會在 [Shared] > [MainLayout.razor] 元件內來使用，底下將會是 [MainLayout.razor] 元件的程式碼

```html
@inherits LayoutComponentBase

<div class="page">
    <div class="sidebar">
        <NavMenu />
    </div>

    <div class="main">
        <div class="top-row px-4">
            <a href="https://docs.microsoft.com/aspnet/" target="_blank">About</a>
        </div>

        <div class="content px-4">
            @Body
        </div>
    </div>
</div>
```

因此，在這裡將會額外在 [Shared] 設計一個 [NavDynamicMenu.razor] 動態功能表清單元件，這個元件同樣會在 [MainLayout.razor] 元件內來參考使用。

在這個 [NavDynamicMenu.razor] 動態功能表清單元件內，使用 `List<MainMenu> mainMenus = new List<MainMenu>();` 屬性來儲存要顯示的功能表清單項目。

對於 [mainMenus] 這個屬性的集合物件值，將會在 Blazor 生命週期的 [OnAfterRenderAsync] 事件內，使用 `menuHelper.MakeMenuObjectAsync` 方法來取得，不過，首先需要使用 `AuthenticationStateProvider.GetAuthenticationStateAsync().User` 這個物件來取得當前登入的使用者是哪個使用者，不過，先要檢查這個 `user.Identity.IsAuthenticated` 屬性來確認現在開啟的網頁，是否已經成功登入，完成身分驗證的程序。

若使用者已經成功登入這個系統，則會使用 `int menuRoleId = Convert.ToInt32(user.FindFirst(c => c.Type == MagicHelper.MenuRoleClaim)?.Value);` 方法來取得這名使用者所配置的功能表角色紀錄 Id，並且透過 `var foo = user.FindFirst(c => c.Type == System.Security.Claims.ClaimTypes.Role && c.Value == MagicHelper.開發者的角色聲明); // 確認這個使用者為系統管理者` 敘述來判斷這個使用者是否具有 開發者的角色聲明 ，因為若這個使用者為 開發者 專用的帳號，則接下來將會額外產生的功能表項目。

最後，將會使用 `mainMenus = await menuHelper.MakeMenuObjectAsync(foo == null ? false : true);` 敘述來取得該使用者可以看得到的功能表項目清單設定到 mainMenus 屬性內，之後就會透過 [Blazor 的 資料綁定 Data Binding](https://docs.microsoft.com/zh-tw/aspnet/core/blazor/components/data-binding?view=aspnetcore-5.0&WT.mc_id=DT-MVP-5002220) 技術，將這名使用者可以看到的功能表項目清單顯示在網頁的最左方。

## 從資料庫取得使用者可以用的功能表項目清單

想要做到從資料庫取得使用者可以用的功能表項目清單，這樣的需求將會設計在 [Helpers] 資料夾內 [MenuHelper.cs] 檔案，底下將會是這個方法內的原始碼。

```csharp
public async Task<List<MainMenu>> MakeMenuObjectAsync(bool sHttc)
{
    #region 讀取該使用者角色的所有功能表清單
    var dataRequest = new DataRequest()
    {
        Sorted = new SortCondition()
        {
            Id = (int)MenuDataSortEnum.Default,
            Title = "預設"
        },
        Skip = 0,
        Take = int.MaxValue
    };
    var allMenuData = await MenuDataService.GetByHeaderIDAsync(RoleId, dataRequest);
    var menuDatas = allMenuData.Result;
    menuDatas = menuDatas
        .Where(x => x.Enable == true)
        .OrderBy(x => x.Sequence).ToList();
    #endregion
    List<MainMenu> mainMenus = new();
    MainMenu mainMenu = new();
    SubMenu subMenu = new();
    #region 依據資料庫內的紀錄，產生要顯示的功能表物件
    foreach (var item in menuDatas)
    {
        if (item.Level == 0)
        {
            #region 第一層功能清單
            mainMenu = new MainMenu()
            {
                IsSubMenu = item.IsGroup,
                IsExpand = false,
                MenuData = item
            };
            mainMenus.Add(mainMenu);
            #endregion
        }
        else
        {
            #region 第二層功能清單
            subMenu = new SubMenu()
            {
                MenuData = item
            };
            mainMenu.SubMenus.Add(subMenu);
            #endregion
        }
    }
    #endregion
    #region 加入鴻才管理者可以使用的功能表清單
    if (isHttc == true)
    {
        mainMenu = new MainMenu()
        {
            IsSubMenu = true,
            IsExpand = false,
            MenuData = new()
            {
                Level = 0,
                Name = "系統管理者",
                Enable = true,
                ForceLoad = false,
                Icon = "mdi-apple-icloud",
                CodeName = "mdi-apple-icloud",
                IsGroup = true,
            }
        };
        mainMenus.Add(mainMenu);
        #region 第二層功能清單
        subMenu = new SubMenu()
        {
            MenuData = new MenuDataAdapterModel()
            {
                Name = ShareBusiness.Helpers.MagicHelper.功能表角色功能名稱,
                CodeName = "MenuRole",
                Enable = true,
                Icon = "mdi-menu",
                IsGroup = false,
                Level = 1,
            }
        };
        mainMenu.SubMenus.Add(subMenu);
        subMenu = new SubMenu()
        {
            MenuData = new MenuDataAdapterModel()
            {
                Name = ShareBusiness.Helpers.MagicHelper.系統日誌功能名稱,
                CodeName = "SystemLog",
                Enable = true,
                Icon = "mdi-message-processing",
                IsGroup = false,
                Level = 1,
            }
        };
        mainMenu.SubMenus.Add(subMenu);
        subMenu = new SubMenu()
        {
            MenuData = new MenuDataAdapterModel()
            {
                Name = ShareBusiness.Helpers.MagicHelper.Excel匯入功能名稱,
                CodeName = "Import",
                Enable = true,
                Icon = "mdi-database-import",
                IsGroup = false,
                Level = 1,
            }
        };
        mainMenu.SubMenus.Add(subMenu);
        #endregion
    }
    #endregion  
    #region 加入登出按鈕
    if (mainMenus.Count > 0)
    {
        mainMenu = new MainMenu()
        {
            IsSubMenu = false,
            IsExpand = false,
            MenuData = new()
            {
                Level = 0,
                Name = "登出",
                Enable = true,
                ForceLoad = true,
                Icon = "mdi-logout",
                CodeName = "/Logout",
                IsGroup = false,
            }
        };
        mainMenus.Add(mainMenu);
    }
    #endregion
    return mainMenus;
}
```

在這個 `#region 讀取該使用者角色的所有功能表清單 ... #endregion` 區段內，根據該使用者的功能表角色 Id ，使用 `MenuDataService.GetByHeaderIDAsync` 方法，取得這個功能表角色內擁有的功能表項目紀錄；這裡將會取得一個 [MenuData] 集合物件，接著，把排除尚未啟用的 MenuData 紀錄。

在接下來的 `#region 依據資料庫內的紀錄，產生要顯示的功能表物件 ... #endregion` 區段內，則會將從資料庫取得的紀錄，轉換成為要顯示在畫面上的集合物件。

最重要的是，若這個使用者為 開發者 帳號，則在 `#region 加入鴻才管理者可以使用的功能表清單 ... #endregion` 區段內，額外加入專屬於開發者帳號才能夠看到的功能項目，也就是說這些功能項目不會定義在資料庫內，完全定義在程式碼內；在這裡的程式碼首先會加入一個功能表項目節點 [系統管理者]，這是一個子功能表項目，也就是說，若點選這個 [系統管理者] 功能表項目，不會導航到任何的頁面，而是會展開一個子功能表清單，這裡將會出現 [功能表角色] 、 [系統日誌] 、 [Excel匯入]；因此，日後若需要擴充 開發者 帳號才能夠看到的功能表選項，便可以要在這裡擴充相關程式碼。

另外，對於這些專屬於 開發者 帳號才能夠看到的功能表項目，在設計這些頁面的時候，需要宣告要具備 `developer` 這樣的聲明在該頁面做前面，例如，底下將會是位於 [Pages] > [MenuRolePage.razor] 檔案內容

```html
@page "/MenuRole"
@attribute [Authorize(Roles = ShareBusiness.Helpers.MagicHelper.開發者的角色聲明)]

<div>
    <div class="page-title">@ShareBusiness.Helpers.MagicHelper.功能表角色功能名稱</div>
    <MenuRoleView PageSize="@MagicHelper.GridPageSize" />
</div>
```

最後，將會使用 `#region 加入登出按鈕 ... #endregion` 區段程式碼，為每個使用者加入 [登出] 功能表選項。


