# 快取 Web API 內容 1 : 將 .NET 物件永久保存儲存，並可以讀取回應用程式內

當在進行 Xamarin.Forms 手機 App 開發的時候，會需要呼叫 Web API 來存取後端伺服器上的資料，不過，對於手機 App 的開發，有許多時候因為網路品質問題，需要能夠進行 off-line 的操作；面對這樣的需求，很多人會使用資料庫架構來進行與遠端資料庫的相關紀錄進行同步更新的設計方式，面對這樣的設計，當然會遇到許多設計上的問題，也就是說，整體技術門檻與設計能力需求較高。

我則是採用另外一種設計方式，那就是採用 Cache 快取方式，設計想是，每次進行 Web API 呼叫的時候，會把取得的集合紀錄快取到手機檔案系統內，採用的設計方式為把希望快取的 .NET 物件，經過 JSON 序列化處理後，將會得到 JSON 物件的文字內容，接著再把這些 JSON 文字內容寫入到檔案內；當重新啟動 App 的時候，會先將之前快取起來的相關 JSON 物件文字檔案內容讀取出來，接著，使用 JSON 反序列化成為 .NET 物件，這樣重新啟動的 App，便會擁有最後一次呼叫 Web API 取得的相關物件值。

採用這樣的設計方式的好處是十分輕巧，因為，只需要 JSON 文字內容寫入到檔案內，或者從檔案中讀取出來。另外，可以免除面對資料庫設計上的相關知識與經驗。

根據這樣的需要，規劃出需要具備底下的相關開發技能：

* 可以將 .NET 物件寫入到檔案的支援方法
* 可以將 檔案文字內容讀取出來，轉換成為 .NET 物件的支援方法
* 一個基底類別，可以呼叫 CRUD 的 Web API
* 該基底 Web API 類別，可以提供寫入或者讀取出需要永久保存的物件內容

在這篇文章中，將會先來設計出如何將 .NET 物件轉成 檔案的 JSON 文字檔案功能(當然，反向操作也要能夠運作)

這篇文章的原始碼位於 [csObjectToFile](https://github.com/vulcanlee/CSharp2021/tree/main/csObjectToFile)

## 建立測試用主控台應用程式專案

* 開啟 Visual Studio 2019
* 選擇右下方的 [建立新的專案] 按鈕
* 在 [建立新專案] 對話窗中
* 從右上方的專案類型下拉按鈕中，找到並選擇 [主控台]
* 從可用專案範本清單內，找到並選擇 [主控台應用程式]
* 點選左下方 [下一步] 按鈕
* 在 [設定新的專案] 對話窗中
* 在 [專案名稱] 欄位中輸入 `csObjectToFile`
* 點選左下方 [下一步] 按鈕
* 在 [其他資訊] 對話窗中
* 在 [目標 Framework] 下拉選單中，選擇 [.NET 5.0 (目前)]
* 點選左下方 [建立] 按鈕

## 加入所需要使用到的 NuGet 套件

* 滑鼠右擊 [csObjectToFile] 專案內的 [相依性] 節點
* 從彈出功能表中，選擇 [管理 NuGet 套件]
* 當 [NuGet: csObjectToFile] 視窗出現後，切換到 [瀏覽] 標籤頁次
* 搜尋 [Newtonsoft.Json] 並且安裝並且套件

## 建立 Helper 類別使用到的資料夾

* 滑鼠右擊 [csObjectToFile] 專案
* 點選 [加入] > [新增資料夾]
* 輸入 `Storages` 這個文字成為此資料夾的名稱

## 建立 把文字內容寫入到檔案或者從檔案讀取出來的支援類別

* 滑鼠右擊 [Storages] 資料夾
* 點選 [加入] > [類別]
* 輸入 `StorageUtility` 這個類別的名稱
* 點選 [新增] 按鈕，完成建立這個類別檔案
* 使用底下程式碼，替換這個檔案內的內容

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using System.Text;
using System.Threading.Tasks;

namespace csObjectToFile.Storages
{
    /// <summary>
    /// Storage 相關的 API
    /// </summary>
    public class StorageUtility
    {
        /// <summary>
        /// 將所指定的字串寫入到指定目錄的檔案內
        /// </summary>
        /// <param name="folderName">目錄名稱</param>
        /// <param name="filename">檔案名稱</param>
        /// <param name="content">所要寫入的文字內容</param> 
        /// <returns></returns>
        public static async Task WriteToDataFileAsync
            (string folderName, string filename, string content)
        {
            //string rootPath = Environment
            //    .GetFolderPath(Environment.SpecialFolder.ApplicationData);
            string rootPath = Environment.CurrentDirectory;

            if (string.IsNullOrEmpty(folderName))
            {
                throw new ArgumentNullException(nameof(folderName));
            }

            if (string.IsNullOrEmpty(filename))
            {
                throw new ArgumentNullException(nameof(filename));
            }

            if (string.IsNullOrEmpty(content))
            {
                throw new ArgumentNullException(nameof(content));
            }

            try
            {
                #region 建立與取得指定路徑內的資料夾
                string fooPath = Path.Combine(rootPath, folderName);
                if (Directory.Exists(fooPath) == false)
                {
                    Directory.CreateDirectory(fooPath);
                }
                fooPath = Path.Combine(fooPath, filename);
                Console.WriteLine($"寫入檔案的路徑 {fooPath}");
                #endregion

                byte[] encodedText = Encoding.UTF8.GetBytes(content);

                using (FileStream sourceStream = new FileStream(fooPath,
                    FileMode.Create, FileAccess.Write, FileShare.None,
                    bufferSize: 4096, useAsync: true))
                {
                    await sourceStream.WriteAsync(encodedText, 0, encodedText.Length);
                }
            }
            catch (Exception ex)
            {
                Debug.WriteLine(ex.ToString());
            }
            finally
            {
            }
        }

        /// <summary>
        /// 從指定目錄的檔案內將文字內容讀出
        /// </summary>
        /// <param name="folderName">目錄名稱</param>
        /// <param name="filename">檔案名稱</param>
        /// <returns>文字內容</returns>
        public static async Task<string> ReadFromDataFileAsync
            (string folderName, string filename)
        {
            string content = "";
            //string rootPath = Environment
            //    .GetFolderPath(Environment.SpecialFolder.ApplicationData);
            string rootPath = Environment.CurrentDirectory;

            if (string.IsNullOrEmpty(folderName))
            {
                throw new ArgumentNullException(nameof(folderName));
            }

            if (string.IsNullOrEmpty(filename))
            {
                throw new ArgumentNullException(nameof(filename));
            }

            try
            {
                #region 建立與取得指定路徑內的資料夾
                string fooPath = Path.Combine(rootPath, folderName);
                if (Directory.Exists(fooPath) == false)
                {
                    Directory.CreateDirectory(fooPath);
                }
                fooPath = Path.Combine(fooPath, filename);
                Console.WriteLine($"讀取檔案的路徑 {fooPath}");
                #endregion

                if (File.Exists(fooPath) == false)
                {
                    return content;
                }

                using (FileStream sourceStream = new FileStream(fooPath,
                    FileMode.Open, FileAccess.Read, FileShare.Read,
                    bufferSize: 4096, useAsync: true))
                {
                    StringBuilder sb = new StringBuilder();

                    byte[] buffer = new byte[0x1000];
                    int numRead;
                    while ((numRead = await sourceStream.ReadAsync(buffer, 0, buffer.Length)) != 0)
                    {
                        string text = Encoding.UTF8.GetString(buffer, 0, numRead);
                        sb.Append(text);
                    }

                    content = sb.ToString();
                }
            }
            catch (Exception ex)
            {
                Debug.WriteLine(ex.ToString());
            }
            finally
            {
            }

            return content.Trim();
        }
    }
}
```

這個 [StorageUtility] 類別能夠將指定的文字內容寫入到檔案內或者從指定的目錄與檔案名稱，讀取出來該檔案內的文字內容；該類別主要是提供兩個方法

* WriteToDataFileAsync 

  將所指定的字串寫入到指定目錄的檔案內

  在這個方法內將會接收 folderName 參數，要寫入資料夾的名稱，這裡可以是一個不同路徑組合，不過，這個檔案要寫入的最上層主要目錄，將會透過 Environment.CurrentDirectory 來取得。另外兩個則是這個檔案 filename 的名稱與要寫入的檔案文字內容 content。

  若指定的目錄不存在，則在寫入檔案之前，將會強制建立起這些目錄，以免寫入檔案的時候發生例外異常。

  當在寫入檔案的時候，會使用到 FileStream 這個類別，接著需要把寫入的文字內容使用 UTF8 進行編碼成為位元陣列，然後就可以寫入到指定的檔案內。

* ReadFromDataFileAsync 

  從指定目錄的檔案內將文字內容讀出

  在這個方法內將會接收 folderName 參數，要讀出資料夾的名稱。另外則是這個檔案 filename 的名稱，而該方法將會回傳讀取到的文字內容

  同樣的，若指定的目錄不存在，則在寫入檔案之前，將會強制建立起這些目錄，以免寫入檔案的時候發生例外異常。

  在讀取檔案內的文字內容時候，將會透過 FileStream 這個類別，使用 FileAccess.Read & FileShare.Read 屬性來讀取內容，另外，也會宣告 useAsync: true 表示要使用非同步的方式來讀取檔案內容。為了降地記憶體回收 GC Garbage Collection 的處理工作量，這裡會使用 StringBuilder 來組合從檔案內讀取到的許多小區塊文字內容，從程式碼可以看的出來，每次將會從檔案中讀取至多 0x1000 大小的資料。

## 建立 .NET 物件 <-> JSON 永久儲存的支援類別


* 滑鼠右擊 [Storages] 資料夾
* 點選 [加入] > [類別]
* 輸入 `StorageJSONService` 這個類別的名稱
* 點選 [新增] 按鈕，完成建立這個類別檔案
* 使用底下程式碼，替換這個檔案內的內容

這個 [StorageJSONService] 類別設計的目的在於提供可以將任何 .NET 型別的物件，寫入到檔案內，或者可以從檔案中讀取該 .NET 物件狀態值，而建立好的 .NET 物件就像當初寫入到檔案內的物件值相同；該類別主要是提供兩個方法

* WriteToDataFileAsync 

  這裡將會傳入要寫入的資料夾名稱與檔案名稱(通常，我會將這個檔案名稱使用該物件的類別名稱，作為該檔案的名稱，這裡可以使用 nameof 這個方法來讀取到該物件的類別名稱)，最後是要傳入所要寫入的 .NET 物件，由於這個類別是個泛型類別 ，因此，`StorageJSONService<T>` ，在使用該類別的時候，需要傳入這個寫入 .NET 物件的型別。

  在這個方法內，會使用 `JsonConvert.SerializeObject(data)` 將這個 .NET 物件轉換成為 JSON 物件，也就是會將 .NET 物件轉換成為文字內容，接著呼叫上面設計的 StorageUtility 類別，使用 `await StorageUtility.WriteToDataFileAsync(directoryName, fileName, output);` 敘述，接這個 JSON 文字內容寫入到檔案內。

  因此，有了這個方法，便可以將任何的 .NET 物件寫入到檔案內來保存當時的狀態值。

* ReadFromFileAsync 

  這裡將會傳入要讀取的資料夾名稱與檔案名稱。

  在這個方法內，呼叫上面設計的 StorageUtility 類別，使用 `await StorageUtility.ReadFromDataFileAsync(directoryName, fileName);` 敘述，將儲存在檔案內的 JSON 文字內容讀取出來；接著會使用 `JsonConvert.DeserializeObject<T>(tempStr)` 將這個 JSON 文字內容轉換成為 .NET 物件

  因此，有了這個方法，便可以將任何的 JSON 文字內容 從檔案內讀取出來，變成一個新的 .NET 物件。

  在這裡有使用到 `T loadedFile = (T)Activator.CreateInstance(typeof(T));` 敘述，會建立一個指定型別的物件，這裡將會使用 無參數建構函式，所謂 [「無參數建構函式」](https://docs.microsoft.com/zh-tw/dotnet/csharp/programming-guide/classes-and-structs/using-constructors?WT.mc_id=DT-MVP-5002220) 就是不接受任何參數的建構函式

```csharp
using Newtonsoft.Json;
using System;
using System.Collections.Generic;
using System.Text;
using System.Threading.Tasks;

namespace csObjectToFile.Storages
{
    public class StorageJSONService<T>
    {

        /// <summary>
        /// Loads data from a file
        /// </summary>
        /// <param name="fileName">Name of the file to read.</param>
        /// <returns>Data object</returns>
        public static async Task<T> ReadFromFileAsync
            (string directoryName, string fileName)
        {
            //T loadedFile = default(T);
            T loadedFile = (T)Activator.CreateInstance(typeof(T));
            string tempStr = "";
            try
            {
                tempStr = await StorageUtility.ReadFromDataFileAsync(directoryName, fileName);
                loadedFile = JsonConvert.DeserializeObject<T>(tempStr);
            }
            catch
            {
                //ApplicationState.ErrorLog.Add(new ErrorLog("LoadFromFile", e.Message));
            }

            return loadedFile;
        }

        public static T LoadFromString(string SourceString)
        {
            T loadedFile = (T)Activator.CreateInstance(typeof(T));
            try
            {
                loadedFile = JsonConvert.DeserializeObject<T>(SourceString);
            }
            catch
            {
                //ApplicationState.ErrorLog.Add(new ErrorLog("LoadFromFile", e.Message));
            }

            return loadedFile;
        }

        /// <summary>
        /// Saves data to a file.
        /// </summary>
        /// <param name="fileName">Name of the file to write to</param>
        /// <param name="data">The data to save</param>
        public static async Task WriteToDataFileAsync
            (string directoryName, string fileName, T data)
        {
            try
            {
                string output = JsonConvert.SerializeObject(data);
                await StorageUtility.WriteToDataFileAsync(directoryName, fileName, output);
            }
            catch
            {
                // Add desired error handling for your application
                // ApplicationState.ErrorLog.Add(new ErrorLog("SaveToFile", e.Message));
            }
        }

    }
}
```

## 設計測試程式碼

* 修正 [Program.cs] 檔案內容，使用底下程式碼來替換

```csharp
using csObjectToFile.Storages;
using System;
using System.Threading.Tasks;

namespace csObjectToFile
{
    class Program
    {
        static async Task Main(string[] args)
        {
            var myClass = new MyClass()
            {
                MyPropertyInt = 20,
                MyPropertyString = "Vulcan Lee",
                MyPropertyDateTime = DateTime.Now.AddDays(-3),
                MyPropertyDouble = 99.82,
                MyPropertyTimeSpan = new TimeSpan(15, 23, 58),
            };

            await StorageJSONService<MyClass>.WriteToDataFileAsync
                ("Data", nameof(myClass), myClass);
        }
    }
}
```

## 執行結果

請按下 F5 開始執行這個專案，將會看到底下的執行結果

```
寫入檔案的路徑 D:\Vulcan\GitHub\CSharp2021\csObjectToFile\csObjectToFile\bin\Debug\net5.0\Data\myClass
```
