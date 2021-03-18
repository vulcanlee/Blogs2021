# .NET Core 使用地址來查詢 GPS 經度緯度

因為專案需求，需要使用地址來查詢出所在地點的經度與緯度數值，做為日後定位處理依據，經過上網搜尋，選用了 [Generic C# Geocoding API](https://github.com/chadly/Geocoding.net) 這裡的套件來幫忙設計這樣的需求。

這個套件支援了不同的地圖提供者的存取 API，在這裡將會使用 Google Map 這個提供者作為練習範例

## 建立測試用主控台應用程式專案

* 開啟 Visual Studio 2019
* 選擇右下方的 [建立新的專案] 按鈕
* 在 [建立新專案] 對話窗中
* 從右上方的專案類型下拉按鈕中，找到並選擇 [主控台]
* 從可用專案範本清單內，找到並選擇 [主控台應用程式]
* 點選左下方 [下一步] 按鈕
* 在 [設定新的專案] 對話窗中
* 在 [專案名稱] 欄位中輸入 `csAddressToGPS`
* 點選左下方 [下一步] 按鈕
* 在 [其他資訊] 對話窗中
* 在 [目標 Framework] 下拉選單中，選擇 [.NET 5.0 (目前)]
* 點選左下方 [建立] 按鈕

## 加入所需要使用到的 NuGet 套件

* 滑鼠右擊 [csAddressToGPS] 專案內的 [相依性] 節點
* 從彈出功能表中，選擇 [管理 NuGet 套件]
* 當 [NuGet: csAddressToGPS] 視窗出現後，切換到 [瀏覽] 標籤頁次
* 搜尋 [Geocoding.Core] 並且安裝並且套件
* 搜尋 [Geocoding.Google] 並且安裝並且套件

## 設計程式碼

請把 Main 方法改成底下的程式碼

>**記得要將 this-is-my-optional-google-api-key 這個文字，改成你申請到的 Google Map API Key**

```csharp
static async Task Main(string[] args)
{
    IGeocoder geocoder = new GoogleGeocoder() { ApiKey = "this-is-my-optional-google-api-key" };
    IEnumerable<Address> addresses = await geocoder.GeocodeAsync("高雄市鼓山區明倫路59號");
    Console.WriteLine("Formatted: " + addresses.First().FormattedAddress); 
    Console.WriteLine("Coordinates: " + addresses.First().Coordinates.Latitude + ", " + addresses.First().Coordinates.Longitude); 
}
```

## 執行結果

請按下 F5 開始執行這個專案，將會看到底下的執行結果

```
Formatted: No. 59, Minglun Road, Gushan District, Kaohsiung City, Taiwan 804
Coordinates: 22.6676008, 120.2975978
```
