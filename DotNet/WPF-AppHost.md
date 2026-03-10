Here's how to do it in WPF, mirroring your Avalonia pattern:

```csharp
// App.xaml.cs
using System.Threading;
using System.Threading.Tasks;
using System.Windows;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using NLog.Extensions.Logging;

namespace MyWpfApp;

public partial class App : Application
{
    internal static IHost? AppHost { get; private set; }

    protected override async void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        AppHost = Host.CreateDefaultBuilder(e.Args)
            .ConfigureServices((context, services) =>
            {
                services.AddLogging(builder =>
                {
                    builder.ClearProviders();
                    builder.AddNLog();
                });
                services.Configure<ApiOptions>(context.Configuration.GetSection("Api"));
                services.AddHostedServices();
                services.AddCobaltServices();
                services.AddPagesAndViewModels();

                // Register your main window and its VM
                services.AddTransient<MainWindowViewModel>();
                services.AddTransient<MainWindow>();
            })
            .Build();

        await AppHost.StartAsync();

        var mainWindow = AppHost.Services.GetRequiredService<MainWindow>();
        mainWindow.DataContext = AppHost.Services.GetRequiredService<MainWindowViewModel>();
        mainWindow.Show();
    }

    protected override async void OnExit(ExitEventArgs e)
    {
        if (AppHost is not null)
        {
            // Clear SynchronizationContext to avoid deadlocks during shutdown
            // (same pattern as your Avalonia code)
            SynchronizationContext.SetSynchronizationContext(null);
            await AppHost.StopAsync();
            AppHost.Dispose();
            AppHost = null;
        }

        base.OnExit(e);
    }
}
```

A few key points:

**Why `OnExit` instead of after `mainWindow.Show()`:** `Show()` is non-blocking in WPF — it returns immediately while the window stays open. So you can't put `StopAsync` right after it like you can in Avalonia where `StartWithClassicDesktopLifetime` *blocks* until the app closes. `OnExit` is the correct WPF equivalent of "after the UI has shut down".

**The `SynchronizationContext.SetSynchronizationContext(null)` trick** is equally applicable here — WPF's dispatcher SynchronizationContext can cause the same deadlock you're guarding against in Avalonia, so clearing it before `StopAsync` is the right move.

**`App.xaml` needs `ShutdownMode`** set appropriately (the default `OnLastWindowClose` is usually fine, but make sure you're not accidentally setting it to `OnExplicitShutdown` unless intentional):

```xml
<Application x:Class="MyWpfApp.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             ShutdownMode="OnLastWindowClose">
    <!-- Remove StartupUri if you're handling window creation manually in OnStartup -->
</Application>
```

Make sure to **remove `StartupUri`** from `App.xaml` since you're manually constructing and showing the window in `OnStartup`.


> If in my wpf code I call somewhere Application.Current.Shutdown(), will it go in OnExit or should I avoid using it and find a better way to close the app in some scenarios ?

`Application.Current.Shutdown()` **will** trigger `OnExit` — that's the correct and intended WPF shutdown path. You don't need to avoid it.

The flow is:
```
Shutdown() → closes all windows → OnExit fires → your StopAsync/Dispose runs
```

The only thing to be aware of is that `OnExit` is `async void`, so if you call `Shutdown()` from a context where you need to *await* the cleanup completing before something else happens, you can't do that — fire-and-forget is the nature of `async void`. But for normal app teardown that's perfectly fine.

One pattern worth considering if you want slightly more control — a dedicated shutdown method you call instead of `Shutdown()` directly:

```csharp
public partial class App : Application
{
    // Call this instead of Application.Current.Shutdown() throughout your app
    public static async Task ShutdownAsync()
    {
        if (AppHost is not null)
        {
            SynchronizationContext.SetSynchronizationContext(null);
            await AppHost.StopAsync();
            AppHost.Dispose();
            AppHost = null;
        }

        Current.Shutdown();  // triggers OnExit, but AppHost is already null so it no-ops
    }

    protected override async void OnExit(ExitEventArgs e)
    {
        // Handles the case where Shutdown() was called directly
        // (e.g. OS close, last window closed, etc.)
        if (AppHost is not null)
        {
            SynchronizationContext.SetSynchronizationContext(null);
            await AppHost.StopAsync();
            AppHost.Dispose();
            AppHost = null;
        }

        base.OnExit(e);
    }
}
```

This way:
- **Normal window close / OS shutdown** → `OnExit` handles cleanup
- **Your code needs to shut down programmatically** → call `App.ShutdownAsync()` which cleans up first, then calls `Shutdown()`, and `OnExit` safely no-ops because `AppHost` is already null