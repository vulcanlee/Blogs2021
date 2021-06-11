# Blazor Server 快速開發專案樣板 7 - 無登入轉首頁與踢出停用帳號

![Blazor Server](../Images/x053.png)

[Blazor Server 快速開發專案樣板 - 相關系列文章清單](https://csharpkh.blogspot.com/2021/06/Blazor-Backend-project-template-syncfusion-NET5.html)

上一篇的文章 : [Blazor Server 快速開發專案樣板 6 - 動態功能表](https://csharpkh.blogspot.com/2021/06/Blazor-Backend-project-template-syncfusion-NET5-Part6-Dynamic-Menu-AuthenticationStateProvider-NavigationManager-Identity-IsAuthenticated.html)

這一篇文章所討論到的需求，是大多數 Blazor 開發者想要知道開發技能，但是，當使用 Visual Studio 建立 [Blazor Server](https://docs.microsoft.com/zh-tw/aspnet/core/blazor/hosting-models?view=aspnetcore-5.0&WT.mc_id=DT-MVP-5002220) 專案的時候，這個預設專案範本僅提供的靜態功能表的設計方式，因此，在這篇文章將會說明如何設計出這樣的功能。

在這個 [Blazor Server 快速開發專案樣板] 中，將會設計三種功能表角色 (當然，開發者可以根據本身不同需求，自行擴增)，分別是 [開發者角色]、[系統管理員角色]、[使用者角色]，若使用不同的使用者登入 ( god , admin , user1 而密碼皆為 123 )，將會看到不同的功能表清單。

