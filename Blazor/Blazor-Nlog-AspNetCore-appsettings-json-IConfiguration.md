# 在 Blazor 專案內使用 NLog ，並透過 appsettings.json 來指定 Log 儲存路徑

在上一篇的文章 [在 Blazor 專案內使用 NLog 進行系統執行日誌紀錄] ，說明了如何建立一個 Blazor 專案，並且透過 NLog 套件來進行系統日誌紀錄的儲存工作。

現在面臨到一個情況，那就是要設定 NLog 所產生的檔案儲存位置，在之前的文章，建立了一個 NLog 設定檔案 nlog.config，在這裡設定日誌紀錄檔案要存放在哪個路徑下，可是，這裡希望能夠依據當時專案執行的環境 Development, Stage, Production，分別來指定 Log 檔案可以儲存到不同的目錄下；當然，看到這樣的需求，就會想到了 [ASP.NET Core 的設定](https://docs.microsoft.com/zh-tw/aspnet/core/fundamentals/configuration/?view=aspnetcore-5.0&WT.mc_id=DT-MVP-5002220) 這篇文章中所提到的功能。

因此，在這篇文章中，將會嘗試在 appsettings.Development.json 與 appsettings.json 建立不同的 Log 儲存路徑宣告，以便可以動態的指定 NLog 的儲存紀錄路徑。

這篇文章的原始碼位於 [bzNlogDynamicPath](https://github.com/vulcanlee/CSharp2021/tree/main/bzNlogDynamicPath)

## 進行 nlog.config 設定

在這篇文章中，將不會另外建立一個新的專案，而是使用上一篇文章 [在 Blazor 專案內使用 NLog 進行系統執行日誌紀錄] 所建立的專案，該專案的原始碼位於 https://github.com/vulcanlee/CSharp2021/tree/main/bzNlog

* 首先，打開 bzNlog 這個專案
* 在專案根目錄下找到並且打開 nlog.config 檔案
* 請將底下內容替換掉原先的內容
* 其實，這裡新的設定內容原先的設定的內容差異在於
  * 在標籤 target xsi:type="File" 節點內，使用了 ${var:LogRootPath} 變數來指定 Log 要儲存的路徑
  * 這個 ${var:LogRootPath} 變數將會在 C# 中來動態指定

```xml
<?xml version="1.0" encoding="utf-8" ?>
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      autoReload="true"
      throwConfigExceptions="true"
      internalLogLevel="info"
      internalLogFile="c:\temp\internal-nlog-AspNetCore5.txt">

	<!-- enable asp.net core layout renderers -->
	<extensions>
		<add assembly="NLog.Web.AspNetCore"/>
	</extensions>

	<!-- the targets to write to -->
	<targets>
		<!-- File Target for all log messages with basic details -->
		<target xsi:type="File" name="allfile" fileName="${var:LogRootPath}\nlog-AspNetCore5-all-${shortdate}.log"
				layout="${longdate}|${event-properties:item=EventId_Id:whenEmpty=0}|${uppercase:${level}}|${logger}|${message} ${exception:format=tostring}" />

		<!-- File Target for own log messages with extra web details using some ASP.NET core renderers -->
		<target xsi:type="File" name="ownFile-web" fileName="${var:LogRootPath}\nlog-AspNetCore5-own-${shortdate}.log"
				layout="${longdate}|${event-properties:item=EventId_Id:whenEmpty=0}|${uppercase:${level}}|${logger}|${message} ${exception:format=tostring}|url: ${aspnet-request-url}|action: ${aspnet-mvc-action}|" />

		<!--Console Target for hosting lifetime messages to improve Docker / Visual Studio startup detection -->
		<target xsi:type="Console" name="lifetimeConsole" layout="${level:truncate=4:lowercase=true}: ${logger}[0]${newline}      ${message}${exception:format=tostring}" />
	</targets>

	<!-- rules to map from logger name to target -->
	<rules>
		<!--All logs, including from Microsoft-->
		<logger name="*" minlevel="Trace" writeTo="allfile" />

		<!--Output hosting lifetime messages to console target for faster startup detection -->
		<logger name="Microsoft.Hosting.Lifetime" minlevel="Info" writeTo="lifetimeConsole, ownFile-web" final="true" />

		<!--Skip non-critical Microsoft logs and so log only own logs-->
		<logger name="Microsoft.*" maxlevel="Info" final="true" />
		<!-- BlackHole -->

		<logger name="*" minlevel="Trace" writeTo="lifetimeConsole, ownFile-web" />
	</rules>
</nlog>
```

## 加入 ASP.NET Core 的設定

* 找到 appsettings.json 檔案，並且打開這個檔案
* 使用底下的設定內容替換掉原先內容
* 這裡使用了 CustomNLog:LogRootPath 來定義出 NLog 日誌紀錄檔案要寫在哪個目錄下

```json
{
  "Logging": {
    "IncludeScopes": false,
    "LogLevel": {
      "Default": "Trace",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*",
  "CustomNLog": {
    "LogRootPath": "C:\\temp\\Custom"
  }
}
```

* 找到 appsettings.json 檔案，並且打開這個檔案
* 使用底下的設定內容替換掉原先內容

```json
{
  "DetailedErrors": true,
  "Logging": {
    "IncludeScopes": false,
    "LogLevel": {
      "Default": "Trace",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "CustomNLog": {
    "LogRootPath": "C:\\temp\\CustomDevelop"
  }
}
```

## 使用 C# 綁定 NLog 變數

* 在根目錄下找到 Startup.cs 檔案，並且打開
* 找到 Configure 方法
* 首先使用 `var logRootPath = Configuration["CustomNLog:LogRootPath"];` 敘述，取出 ASP.NET Core 的設定 內的設定值
* 接著使用 `LogManager.Configuration.Variables["LogRootPath"] = logRootPath;` 將這個值設定到 NLog 內

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    var logRootPath = Configuration["CustomNLog:LogRootPath"];
    LogManager.Configuration.Variables["LogRootPath"] = logRootPath;
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }
    else
    {
        app.UseExceptionHandler("/Error");
        // The default HSTS value is 30 days. You may want to change this for production scenarios, see https://aka.ms/aspnetcore-hsts.
        app.UseHsts();
    }
 
    app.UseHttpsRedirection();
    app.UseStaticFiles();
 
    app.UseRouting();
 
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapBlazorHub();
        endpoints.MapFallbackToPage("/_Host");
    });
}
```
 
## 進行測試

現在可以執行這個專案，分別在不同的環境下來執行，看看日誌是否有寫到不同的目錄下
