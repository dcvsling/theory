# MSBuild

## 如何建立可透過 PackageReference 來運行參考包中的 MSBuild 程式

### Prepare
- dotnet cli 或 NuGet Command 
  - 如需要同時進行編譯則需要 相關編譯器
- 想要運行的 MSBuild props 和 targets 及其相關建構好的 dll 等

### Step 1 建立封裝的資料夾與專案
- 建立一個新的 Msbuild project (ex: csproj, vsproj, .....etc) 以下會以csproj 作為示範
  - 如果透過dotnet new 來建立則可將期建立的 程式碼移除
- 將準備好的 props, targets 放進資料夾 targets (可改變名稱) 中
- 將分析器相關的檔案放在 analysers\dotnet\\{language} 資料夾下
- 將相關可執行檔 , dll 等放進 tools 中
- 將想放入對方專案的檔案放進 contentFiles 資料夾中 (例如: 原始碼 或 設定檔)

### Step 2 建立參考後引動程式的程式碼
- 建立資料夾 build 並建立與專案或資料夾名稱同名的 props 和 targets 兩個檔案
  - MSBuild 會在執行的時候一併加入  
  build/{TargetFramework}/{PackageName}.{props/targets}
- 如有分析器則需於tools資料夾中建立 install.ps1 與 uninstall.ps1 內容請參考[分析器NuGet官方文件]  
(如未建立tools資料夾則須先建立)
  - 分析器相關內容請參考上述連結

### Step 3 追加封裝時的專案檔或資料夾結構描述
- 於專案檔中寫入下方語法  
  - Content 標籤為 include的檔案是為Content
  - Include 屬性為 有哪些檔案要視為Content
  - build 路徑 為專案或資料夾中的 build 資料夾 
  - **/\*.\* 路徑 為 globbing pattern 表示 任意資料夾下的任意檔案
  - Pack 標籤為是否要打包
  - PackagePath 標籤為打包時所在的路徑
  - $(TargetFramework) 為打包時的目標框架
  - 以上寫法可能依照sdk 的不同而有所不同
  
```xml
  <ItemGroup>
    <Content Include="build/**/*.*">
      <Pack>true</Pack>
      <PackagePath>build/$(TargetFramework)</PackagePath>
    </Content>
  </ItemGroup>
```
- 可以將 build 改成 targets 或 tools 來將指定的資料夾放入特定的打包時的資料夾中  
對應表如下

| Path  | PackagePath |
|-------|:------:|
| build | build/$(TargetFramework) |
| contentFiles | contentFiles/{cs/vb}/$(TargetFramework)/ |
| tools | tools |
| analysis | analysis/dotnet/{cs/vb} |

- 加上 Version 以控管版本
  - 預設為1.0.0
```xml
  <PropertyGroup>
    <Version>0.1.0</Version>
  </PropertyGroup>
```
- 撰寫封裝資訊 dotnet專案可透過[DotNet專案追加屬性]來追加 nuspec 相關屬性
  - 如果是只有資料夾的部分則僅需要建立 nuspec 檔並撰寫相關內容


### Step 4 打包
- 透過上述的流程所建立的專案可直接用 dotnet pack 指令來進行封裝
- NuGet Command 使用 nuget pack 打包即可

## 參考連結
---
- [分析器NuGet官方文件]
- [DotNet專案追加屬性]
- [MSBuild官方文件]

[分析器NuGet官方文件]: https://docs.microsoft.com/zh-tw/nuget/reference/analyzers-conventions#install-and-uninstall-scripts
[DotNet專案追加屬性]: https://docs.microsoft.com/zh-tw/dotnet/core/tools/csproj#nuget-metadata-properties
[MSBuild官方文件]: https://docs.microsoft.com/zh-tw/visualstudio/msbuild/msbuild?view=vs-2019