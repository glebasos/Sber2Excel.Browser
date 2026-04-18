# Sber2Excel.Browser

WebAssembly-«голова» для [Sber2Excel](../Sber2Excel) — то же самое приложение, но крутится прямо в браузере. Никакого сервера, файлы пользователя на сервер не уходят: PDF разбирается в WASM, CSV/XLSX скачиваются через File System Access API.

## Требования

- [.NET 10 SDK](https://dotnet.microsoft.com/download/dotnet/10.0) с workload `wasm-tools`:
  ```bash
  dotnet workload install wasm-tools
  ```
- Любой статический веб-сервер для раздачи `wwwroot/` после публикации.

## Запуск из исходников

```bash
cd Sber2Excel.Browser
dotnet run
```

Откроется страница в браузере (порт зависит от Avalonia.Browser dev-сервера).

## Сборка релиза

```bash
dotnet publish -c Release
```

Результат: `bin/Release/net10.0-browser/publish/wwwroot/` — статика, готовая к деплою на любой CDN / GitHub Pages / nginx / S3.

В csproj включены:

- `PublishTrimmed=true` — режет неиспользуемый IL ради размера сборки;
- `RunAOTCompilation=true` — компилирует .NET в WASM AOT, ускоряет работу таблицы и парсера ценой времени публикации.

Сборка с AOT занимает несколько минут; для быстрых итераций отключите его в csproj.

## Состав проекта

```
Sber2Excel.Browser/
  Program.cs                       ← AppBuilder + StartBrowserAppAsync("out")
  Properties/AssemblyInfo.cs       ← [SupportedOSPlatform("browser")]
  runtimeconfig.template.json      ← wasmHostProperties / perHostConfig
  wwwroot/
    index.html                     ← <div id="out"> — Avalonia монтируется сюда
    main.js
    app.css
  Sber2Excel.Browser.csproj
```

Весь UI и логика — в [`Sber2Excel`](../Sber2Excel). Здесь только хост.

## Особенности WASM-сборки

- **Никакого `System.Diagnostics.Process`.** `AvaloniaUI.DiagnosticsSupport` подключён только в десктопной «голове», иначе при старте — `Process_PlatformNotSupported`.
- **Файловые потоки.** В браузере экспорт идёт в `MemoryStream`, потом байты `CopyToAsync` в поток, выданный `StorageProvider.SaveFilePickerAsync`, плюс `FlushAsync`. Прямая запись `StreamWriter`/`ClosedXML` в браузерный поток давала пустые файлы.
- **Тримминг + биндинги.** XAML с `x:CompileBindings="True"` переживает `PublishTrimmed=true`. Биндинги, построенные в коде через `new Binding(nameof(...))`, — рефлексивные, тримминг вырезает их цели; в этом проекте такие биндинги (ячейки «Сумма» и «Остаток» в `TreeDataGrid`) заменены на прямое чтение свойств `Transaction` в `DataContextChanged`.

## Хостинг

Достаточно отдать содержимое `wwwroot/` любым статическим сервером. Для `.wasm` обязательно настройте корректный MIME-тип (`application/wasm`) и желательно сжатие brotli/gzip — это сильно влияет на время загрузки.
