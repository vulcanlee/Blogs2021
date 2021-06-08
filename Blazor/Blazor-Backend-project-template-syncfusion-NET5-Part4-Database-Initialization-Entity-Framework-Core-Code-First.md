# Blazor Server 快速開發專案樣板 4 - 資料庫重建與紀錄初始化

![Blazor Server](../Images/x050.png)

[Blazor Server 快速開發專案樣板 - 相關系列文章清單](https://csharpkh.blogspot.com/2021/06/Blazor-Backend-project-template-syncfusion-NET5.html)

上一篇的文章 : [Blazor Server 快速開發專案樣板 3 - 日誌與來源 IP](https://csharpkh.blogspot.com/2021/06/Blazor-Backend-project-template-syncfusion-NET5-Part3-logging-Source-IP-NLog.html)

在這篇文章將會說明在進行 [Blazor](https://docs.microsoft.com/zh-tw/aspnet/core/blazor/?view=aspnetcore-5.0&WT.mc_id=DT-MVP-5002220) 專案開發的時候，通常會使用存取資料庫的紀錄，在這個 [Blazor Server 快速開發專案樣板] 內，使用了 [Entity Framework Core 5](https://docs.microsoft.com/zh-tw/ef/core/what-is-new/ef-core-5.0/whatsnew?WT.mc_id=DT-MVP-5002220) Code First 方式來進行存取資料庫，並且在開發階段將會使用本機電腦上的 localDB 資料庫來進行開發。

在 [Blazor Server 快速開發專案樣板 1 - 建立一個新的專案](https://csharpkh.blogspot.com/2021/06/Blazor-Backend-project-template-syncfusion-NET5-Part1-Create-New.html) 文章中有提到，一旦建立起一個新專案之後，需要建立起一個開發用的資料庫，這樣這個 [Blazor Server 快速開發專案樣板] 便可以開始運作了；不過，在這裡到底做了什麼事情，可以做到這樣的成果，接下來將會來說明這些設計過程。

## InitializationPage.razor

```html
@page "/Initialization"
@using Microsoft.Extensions.Configuration;
@using Microsoft.EntityFrameworkCore;
@using System.Threading.Tasks
@using System.Threading
@using System.Diagnostics

@inject IWebHostEnvironment env
@inject DatabaseInitService DatabaseInitService

@layout EmptyLayout

@using System.Security.Claims
@using Microsoft.AspNetCore.Hosting
@using Microsoft.Extensions.Hosting
@using Syncfusion.Blazor.Spinner

<h1 class="bg-primary text-white-50 my-1">資料庫初始化作業!</h1>

<environment include="Staging,Production">
    @if (verified == false)
    {
        <div>
            <div class="h2 text-success">@Question</div>
            <p>請輸入驗證碼</p>
            <div class="input-group mb-3">
                <input type="text" class="form-control"
                       placeholder="驗證碼" aria-label="Username" aria-describedby="basic-addon1"
                       @bind="verifyCode">
            </div>
            <div class="input-group mb-3">
                <button class="btn btn-primary"
                        @onclick="OnVerifyCode">
                    送出
                </button>
            </div>
        </div>
    }
</environment>

@if (verified)
{
    <div class="card mb-4">
        <div class="card mb-4">
            <div class="card-header bg-danger text-white">
                <span class="h2">資料庫初始化</span>
            </div>
            <article class="card-body bg-light">
                <div>
                    <button class="btn btn-danger" @onclick="Init">資料庫重新建立與資料初始化</button>
                </div>
            </article>
        </div>
    </div>
}

<div id="container">
    <SfSpinner @bind-Visible="@VisibleProperty">
    </SfSpinner>
</div>


@code {
    bool verified = false;
    string verifyCode = "";
    int Question = 0;
    bool VisibleProperty = false;

    protected override void OnInitialized()
    {
        Question = new Random().Next(10000, 90000);
        if (env.IsDevelopment())
        {
            verified = true;
            verifyCode = Question.ToString();
        }
    }
    public async Task Init()
    {
        this.VisibleProperty = true;
        await DatabaseInitService.InitDataAsync();
        this.VisibleProperty = false;
    }
    void OnVerifyCode()
    {
        try
        {
            int verifyNumber = 0;
            int answer = 0;
            verifyNumber = Convert.ToInt32(verifyCode);
            answer = 98765 - Question;
            if (verifyNumber == answer)
            {
                verified = true;
            }
        }
        catch { }
    }
}
```

```csharp
public async Task InitDataAsync()
{
    Random random = new Random();
    #region 適用於 Code First ，刪除資料庫與移除資料庫
    string Msg = "";
    Msg = $"適用於 Code First ，刪除資料庫與移除資料庫";
    Logger.LogInformation($"{Msg}");
    await context.Database.EnsureDeletedAsync();
    Msg = $"刪除資料庫";
    Logger.LogInformation($"{Msg}");
    await context.Database.EnsureCreatedAsync();
    Msg = $"建立資料庫";
    await SystemLogHelper.LogAsync(new SystemLogAdapterModel()
    {
        Message = Msg,
        Category = LogCategories.Initialization,
        Content = "",
        LogLevel = LogLevels.Information,
        Updatetime = DateTime.Now,
        IP = HttpContextAccessor.HttpContext.Connection.RemoteIpAddress.ToString(),
    });
    Logger.LogInformation($"{Msg}");
    #endregion
    ...
}
```
