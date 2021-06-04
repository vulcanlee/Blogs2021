# Blazor SfGrid SfDataManager 的參數 Parameter 傳遞的異常設計分析

使用 Syncfusion SfGrid 元件可以使用 SfDataManager 來指定客製化轉接器，取得 Grid 元件所要顯示的紀錄，不過，需要根據使用者的操作，傳遞不同的參數數值到該客製化轉接器 Adapter 內，以便取得該參數數值所適合的相關集合物件，但是，對於許多大多數的開發者而言，面對到許多時候需要針具各種不同的情況，會發生不同的結果，這樣的困境會讓人無法適從。

因此，這篇文章將會是針對這樣的困境來進行說明，了解到發生了甚麼問題，以及要如何解決此一問題

這個說明專案的原始碼位於 [bzSfGridRefresh](https://github.com/vulcanlee/CSharp2021/tree/master/bzSfGridRefresh)

## 準備練習專案

由於這裡的專案將會使用到 Syncfusion 元件來做為講解，因此，首先需要建立一個 Blazor for Syncfusion 的專案，根據 Syncfusion 官方文件的說明，加入相關的程式碼，以便可以使用 Syncfusion 的元件。

## 先從 Blazor 預設功能來觀看元件間的參數傳遞

在此，先打開 [Pages] 資料夾內的 Index.razor 檔案

確定該 Blazor 元件如底下內容

```html
@page "/"

<h1>Hello, world!</h1>

@*<Index1View />*@
@*<Index2View />*@
<Index3View />

<SurveyPrompt Title="How is Blazor working for you?" />
```

在這裡將會使用自行設計的一個 Blazor 元件，這個 Index3View.razor 元件的程式碼如下

```html
@using Syncfusion.Blazor.Grids;
@using Syncfusion.Blazor.Data;
@using Syncfusion.Blazor;

<div>
    <button class="btn btn-primary" @onclick="()=>OnClick(1)">Send A</button>
    <button class="btn btn-primary" @onclick="()=>OnClick(2)">Send B</button>
    <button class="btn btn-primary" @onclick="()=>OnClick(3)">Send C</button>
</div>
<div>
    <Component @ref="component" CurrentTypeCondition="@CurrentTypeCondition" />
</div>

@code{
    public string CurrentTypeCondition { get; set; }
    Component component;

    async Task OnClick(int i)
    {
        Console.WriteLine();
        Console.WriteLine();
        OutputHelper.Output("CurrentTypeCondition 準備要變更了");

        #region 使用同步方式來進行變更 CurrentTypeCondition
        if (i == 1)
        {
            CurrentTypeCondition = "A";
        }
        else if (i == 2)
        {
            CurrentTypeCondition = "B";
        }
        else if (i == 3)
        {
            CurrentTypeCondition = "C";
        }
        OutputHelper.Output("CurrentTypeCondition 已經變更了");
        #endregion

        OutputHelper.Output("Raise component.Refresh");
        component.Refresh();
    }
}
```

此時，可以執行這個專案，將會看到執行結果

![](../Images/Csharp897.png)

現在來看看這個元件做了甚麼事情？

首先，從底下的 HTML 標記可以看出，在這個 Index3View.razor 元件中，有參考到一個 Component.razor 元件。

```html
<Component @ref="component" CurrentTypeCondition="@CurrentTypeCondition" />
```

接下來看看這個 Component.razor 元件有甚麼內容

```html
<h3>@CurrentTypeCondition</h3>

@code {
    private string myVar;

    [Parameter]
    public string CurrentTypeCondition
    {
        get { return myVar; }
        set
        {
            myVar = value;
            OutputHelper.Output($"Component CurrentTypeCondition has changed {myVar}");
        }
    }

    public void Refresh()
    {
        OutputHelper.Output("Component Refresh() is running");
        OutputHelper.Output($"【In Component, CurrentTypeCondition is ■{CurrentTypeCondition}■】");
    }
}
```

在這個元件中非常的簡單，這裡有宣告一個參數 CurrentTypeCondition ，因為該參數有宣告 `[Parameter]`，因此，可以從別的元件中將該物件值傳入到這個元件內，在剛剛看到的 Index3View.razor 檔案內，就有看到這樣的標記宣告，`<Component @ref="component" CurrentTypeCondition="@CurrentTypeCondition" />`，這裡表示會將 Index3View.razor 元件中的 CurrentTypeCondition 物件值，透過 Blazor 參數傳遞綁定方式 (更多這方面的資訊，可以參考 [使用元件參數進行系結](https://docs.microsoft.com/zh-tw/aspnet/core/blazor/components/data-binding?view=aspnetcore-5.0#binding-with-component-parameters&WT.mc_id=DT-MVP-5002220))，如此，在 Component.razor 這個元件內，就會接收到這個參數。

為了想要知道這個元件內的 CurrentTypeCondition 參數數值何時產生了變化，也就是該參數數值已經變更了，這裡使用了 [含有支援欄位的屬性](https://docs.microsoft.com/zh-tw/dotnet/csharp/programming-guide/classes-and-structs/properties#properties-with-backing-fields?WT.mc_id=DT-MVP-5002220) 設計方式，從程式碼可以看出
