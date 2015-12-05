---
layout: post
title: Погружение в ASP.NET 5 Runtime
categories: ASP-NET-5, DNX
published: draft
---

## .NET Runtime Environment (DNX) ##

ASP.NET базируется на гибком, кроссплатформенном Runtime хосте, который может работать с разными .NET CLR (.NET Core, Mono, .NET Framework). Вы можете запустить ASP.NET 5 на полном .NET Framework или можете запустить на новом [.NET Core](https://docs.asp.net/en/latest/conceptual-overview/dotnetcore.html), который позволяет вам просто копировать его в существующее окружение, без изменения чего-либо еще на вашей машине. Используя .NET Core вы также можете запустить ASP.NET 5 кроссплатформенно на Linux и Mac OS.

Инфраструктура позволяющая запускать и исполнять приложения ASP.NET 5 называется [.NET Runtime Environment (DNX)](https://docs.asp.net/en/latest/dnx/overview.html). DNX предоставляет все что необходимо для разработки приложений на .NET: host process, CLR hosting логику, обнаружение управляемой Entry Point и т.д.

DNX базируется на том же самом .NET CLR и базовой библиотеке классов, что знают и любят существующие .NET разработчики, и в то же время он разработан с возможностью запускать приложения (на данный момент веб и консольные приложения) под операционными системами отличными от Windows.

Логически DNX имеет пять слоев функциональности. Я опишу каждый из этих слоев и их обязанности.

## Слой первый: Нативный процесс ## 

Нативный процесс - это очень тонкий слой с обязанностью найти и вызвать нативный CLR host, передав в него аргументы переданные в сам процесс. В Windows - это dnx.exe (находится в %YOUR_PROFILE%/.dnx/runtimes/%CHOOSEN_RUNTIME%); В Mac и Linux - это запускаемый bash script. [Запуск на IIS](https://docs.asp.net/en/latest/publishing/iis.html) происходит с помощью нативного HTTP-модуля: [HTTPPlatformHandler](https://azure.microsoft.com/en-us/blog/announcing-the-release-of-the-httpplatformhandler-module-for-iis-8/), который нужно установить на IIS. Использование HTTPPlatformHandler позволяет запускать веб-приложение без любых зависимостей от .NET Framework (естественно при запуске веб-приложений нацеленных на .NET Core, а не на полный .NET Framework).

## Слой второй и третий: Нативный CLR host и CLR ##

Имеют три главных обязанности:

1. Запустить CLR. Способы достижения этого отличаются в зависимости от используемой версии CLR. Например, для запуска .NET Core загружается coreclr.dll, настраивается и запускается runtime и создается домен приложений, в котором будет запускаться весь managed code. Для Mono и .NET Framework, процесс отчасти различается, но результат будет тот же самый.
2. Вызвать managed entry point, которая является следующим слоем.
3. Когда точка входа нативного хоста возвращает управление этот процесс будет "убирать за собой" и выключать CLR - выгружать домен приложений и останавливать runtime.

## Слой четвертый: Управляемая entry point ##

> Примечание: В целом логика этого слоя находится в [Microsoft.DNX.Host](https://github.com/aspnet/dnx/tree/1.0.0-rc1-final/src/Microsoft.Dnx.Host). Entry Point этого слоя можно считать [RuntimeBootstrapper](https://github.com/aspnet/dnx/blob/1.0.0-rc1-final/src/Microsoft.Dnx.Host/RuntimeBootstrapper.cs).

Этот первый слой написанный на managed code. Он ответственен за:

1. [Создание LoaderContainer](https://github.com/aspnet/dnx/blob/1.0.0-rc1-final/src/Microsoft.Dnx.Host/Bootstrapper.cs#L29), который будет содержать необходимые ILoader'ы. ILoader'ы ответственны за загрузку сборки. Когда CLR будет просить LoaderContainer предоставить какую-либо сборку, он будет делать это используя его ILoader'ы.
2. [Создание корневого ILoader'а](https://github.com/aspnet/dnx/blob/1.0.0-rc1-final/src/Microsoft.Dnx.Host/Bootstrapper.cs#L32), который будет загружать требуемые сборки (из папки bin выбранного dnx runtime: %YOUR_PROFILE%/.dnx/runtimes/%CHOOSEN_RUNTIME%/bin/ и предоставленных во время запуска нативного процесса дополнительных путей, с помощью параметра `--lib`).
3. [Настройку IApplicationEnvironment и ядра инфраструктуры системы Dependency Injection](https://github.com/aspnet/dnx/blob/dev/src/Microsoft.Dnx.Host/Bootstrapper.cs#L59-L70).
4. [Вызов entry point](https://github.com/aspnet/dnx/blob/dev/src/Microsoft.Dnx.Host/Bootstrapper.cs#L80) конкретного приложения или [Microsoft.DNX.ApplicationHost](https://github.com/aspnet/dnx/blob/1.0.0-rc1-final/src/Microsoft.Dnx.ApplicationHost/Program.cs).

В этом слое сборки загружаются только из каталога bin выбранного dnx runtime (%YOUR_PROFILE%/.dnx/runtimes/%CHOOSEN_RUNTIME%/bin/) или указанных с помощью параметра `--lib` при запуске нативного процесса дополнительных путей. Есть следующий слой, что добавляет дополнительные загрузчики для разрешения зависимостей из NuGet пакетов или даже кода скомпилированного во время выполнения.

## Слой пятый: Application Host ##

[Microsoft.DNX.ApplicationHost](https://github.com/aspnet/dnx/tree/1.0.0-rc1-final/src/Microsoft.Dnx.ApplicationHost) - это Application Host поставляемый вместе с DNX. В его обязанности входят:

1. Просмотреть зависимости указанные в project.json и загрузить их. Логика обхода зависимостей описана более детально, в этой статье: [Dependency-Resolution](https://github.com/aspnet/Home/wiki/Dependency-Resolution).
2. Добавление дополнительных загрузчиков сборок, которые могут загружать сборки из различных источников, таких как установленные NuGet пакеты, исходники компилируемые в runtime, используя Roslyn, и т.д.
3. [Вызов entry point](https://github.com/aspnet/dnx/blob/1.0.0-rc1-final/src/Microsoft.Dnx.ApplicationHost/Program.cs#L230-L240) вашей сборки или сборки [Microsoft.AspNet.Hosting](https://github.com/aspnet/Hosting/blob/1.0.0-rc1/src/Microsoft.AspNet.Hosting/Program.cs) в случае веб-приложения (которая знает как найти Startup.cs файл и вызвать Configure методы). Сборка может быть чем угодно, имеющим entry point, [которую Application Host знает как загрузить](https://github.com/aspnet/dnx/blob/1.0.0-rc1-final/src/Microsoft.Dnx.Runtime.Sources/Impl/EntryPointExecutor.cs#L20). Приходящий вместе с DNX Application Host [знает как найти](https://github.com/aspnet/dnx/blob/1.0.0-rc1-final/src/Microsoft.Dnx.Runtime.Sources/Impl/EntryPointExecutor.cs#L70-L110) `public static void Main` метод.

## Кроссплатформенныe SDK Tools ##

dnx поставляется вместе c SDK, который содержит все необходимое для построения кросс-платформенных .NET приложений.

[**DNVM** - DNX Version Manager](https://github.com/aspnet/Home/wiki/Version-Manager). Позволяет просмотреть установленные на компьютере DNX, установить новые и выбрать тот, который вы будете использовать. Прочитать про установку DNVM можно [здесь](https://docs.asp.net/en/latest/getting-started/index.html).

DNVM устанавливает DNX'ы из NuGet feed настроенного на использование DNX_FEED переменной среды. DNX'ы не являются NuGet пакетами в традиционном смысле - пакеты на которые вы можете ссылаться в качестве зависимостей. NuGet - это удобный путь доставки и управления версиями DNX. По-умолчанию DNX устанавливается копированием и распаковкой архива с DNX в "%USERPROFILE%\.dnx\runtimes".

[**DNU** - DNX Utility](https://github.com/aspnet/Home/wiki/DNX-utility). Инструмент для установки, восстановления и создания NuGet пакетов. По-умолчанию, пакеты устанавливаются в "%USERPROFILE%\.dnx\packages", но вы можете изменить это, установив другой путь в в вашем global.json файле.

## Кроссплатформенное консольное приложение на .NET ##

Сейчас я покажу вам как, используя DNX, создать простое кроссплатформенное консольное приложение на .NET.

Нам необходимо создать DLL с entry point и мы можем сделать это, используя "Console Application (Package)" шаблон в Visual Studio 2015.

Код нашего приложения:

	namespace ConsoleApp
	{
	    public class Program
	    {
	        public static void Main(string[] args)
	        {
	            Console.WriteLine("Hello World");
	            Console.ReadLine();
	        }
	    }
	}

Код выглядит абсолютно идентично обычному консольному приложению.

Вы можете запустить это приложение из Visual Studio или из командной строки, набрав: `dnx run`, находясь в папке с файлом project.json (корень проекта), приложения которое вы хотите запустить.

`dnx run`, фактически, разворачивается в:

`dnx.exe --appbase . Microsoft.DNX.ApplicationHost run`

Команда `dnx` запускает нативный процесс (dnx.exe), указывает application base, как текущую директорию и указывает управляемой entry point (четвертый слой) вызвать Microsoft.DNX.ApplicationHost. Нативный процесс не имеет специальных знаний о Microsoft.DNX.ApplicationHost. Он просто ищет [стандартную entry point](https://github.com/aspnet/dnx/blob/1.0.0-rc1-final/src/Microsoft.Dnx.ApplicationHost/Program.cs#L22) в сборке `Microsoft.DNX.ApplicationHost` и вызывает ее. Далее основываясь на application base ApplicationHost узнает путь до папки запускаемого проекта, а команда `run` говорит ему найти entry point этого проекта и вызвать ее.

Если вы не хотите использовать ApplicationHost. Вы можете вызвать нативный слой, передав ему ваше приложение напрямую.

Для того, чтобы сделать это:

1. Сбилдите ваше приложение, сгенерировав DLL (Убедитесь, что вы поставили галочку около пункта "Produce outputs on build" в разделе "Build" свойств вашего проекта). Вы можете найти результаты в директории artifacts в корне папки вашего решения (вы также можете использовать не Visual Studio, а команду `dnu build`, набрав ее находясь в папке с файлом project.json вашего проекта).
2. Наберите команду: `dnx <путь_до_библиотеки_и_ее_имя_включая_расширение>`.

Вызов DLL напрямую - это довольно низкоуровневый подход написания приложений. Вы не используете default application host, поэтому вы отказываетесь и от использования файла project.json и улучшенного NuGet-based механизма управления зависимостями. Вместо этого любые библиотеки от которых вы зависите, будут загружаться из указанных при запуске нативного процесса с помощью параметра `--lib` директорий. До окончания этой статьи я буду использовать default application host.

**Рассказать про доступ к сервисам** 

## Хостинг ##

В веб-приложениях поверх DNX ApplicationHost работает [слой хостинга](https://docs.asp.net/en/latest/fundamentals/hosting.html). Он ответственен за поиск веб-сервера, логику запуска веб-сайта, размещение сайта на сервере, а затем "уборку за собой", когда приложение выключается. Он также предоставляет приложению некоторые дополнительные, связанные со слоем хостинга сервисы.

DNX ApplicationHost вызывает [entry point метод](https://github.com/aspnet/Hosting/blob/1.0.0-rc1/src/Microsoft.AspNet.Hosting/Program.cs) [Microsoft.AspNet.Hosting](https://github.com/aspnet/Hosting/tree/1.0.0-rc1/src/Microsoft.AspNet.Hosting). Вы можете настроить какой веб-сервер использовать, указав опцию `--server` или использовав другие способы настройки хостинга, такие как файл hosting.json или environment variables. Слой хостинга загружает указанный сервер и стартует его. Обычно используемый сервер должен быть перечислен в списке зависимостей в файле pfoject.json, чтобы его можно было загрузить.

Заметьте, что команды определенные в вашем poject.json на самом деле только устанавливают дополнительные аргументы для `dnx.exe`. Например, шаблон веб-приложения ASP.NET 5 включает набор команд, таких как:

	  "commands": {
	    "web": "Microsoft.AspNet.Server.Kestrel",
	    "ef": "EntityFramework.Commands"
	  },

И когда вы набираете: `dnx web`, в реальности это преобразуется в:

`dnx.exe --appbase . Microsoft.DNX.ApplicationHost Microsoft.AspNet.Server.Kestrel`

В свою очередь вызов [entry point Microsoft.AspNet.Server.Kestrel](https://github.com/aspnet/KestrelHttpServer/blob/dev/src/Microsoft.AspNet.Server.Kestrel/Program.cs) преобразуется в вызов: 

`Microsoft.AspNet.Hosting --server Microsoft.AspNet.Server.Kestrel`

Так что итоговая команда будет:

`dnx.exe --appbase . Microsoft.DNX.ApplicationHost Microsoft.AspNet.Hosting --server Microsoft.AspNet.Server.Kestrel`

## Startup ##

Слой хостинга также ответственен за запуск стартовой логики вашего приложения. Обычно она находится в Startup классе с Configure методом для настройки request pipeline и ConfigureServices методом для настройки используемых приложением сервисов.

	namespace WebApplication1
	{
	  public class Startup
	  {
	    public void ConfigureService(IServiceCollection services)
	    {
	      // Добавьте сервисы для вашего приложения здесь
	    }
	    public void Configure(IApplicationBuilder app)
	    {
	      // Настройте ваш request pipeline здесь
	    }
	  }
	}

В вашем Configure методе вы используете IApplicationBuilder интерфейс, чтобы настраивать request pipeline вашего приложения. Application Builder позволяет вам использовать ("Use") middleware, создавать новые ("New") application builders и создает ("Build") итоговый request delegate.

Request delegate - это ключевая концепция ASP.NET 5. Request delegate принимает HttpContext и асинхронно делает нечто полезное с ним:

`public delegate Task RequestDelegate(HttpContext context);`

[ASP.NET 5 middleware](https://docs.asp.net/en/latest/fundamentals/middleware.html) принимают на вход следующий в request pipeline request delegate и предоставляет request delegate со своей логикой. Возвращаемый request delegate может вызывать или не вызывать следующий request delegate. В качестве упрощения создания middleware не вызывающего следующий request delegate, вы можете использовать Run extension метод IApplicationBuilder.

`app.Run(async context => await context.Response.WriteAsync("Hello, world!"));`

Того же самого можно достичь используя Use extension метод и не вызывая следующий request delegate:

`app.Use(next => async context => await context.Response.WriteAsync("Hello, world!"));`

Чтобы создать middleware, которое вы могли бы использовать в разных приложениях, вы можете написать его в виде класса. По-соглашению, следующий request delegate (а также необходимые сервисы и дополнительные параметры) передается в конструктор этого класса. request delegate этого middleware должен быть реализован в асинхронном Invoke методе, как это показано ниже:

	using Microsoft.AspNet.Builder;
	using Microsoft.AspNet.Http;
	using System.Threading.Tasks;
	public class XHttpHeaderOverrideMiddleware
	{
	  private readonly RequestDelegate _next;
	  public XHttpHeaderOverrideMiddleware(RequestDelegate next)
	  {
	    _next = next;
	  }
	  public Task Invoke(HttpContext httpContext)
	  {
	    var headerValue =
	      httpContext.Request.Headers["X-HTTP-Method-Override"];
	    var queryValue =
	      httpContext.Request.Query["X-HTTP-Method-Override"];
	    if (!string.IsNullOrEmpty(headerValue))
	    {
	      httpContext.Request.Method = headerValue;
	    }
	    else if (!string.IsNullOrEmpty(queryValue))
	    {
	      httpContext.Request.Method = queryValue;
	    }
	    return _next.Invoke(httpContext);
	  }
	}

В request pipeline вы можете включить middleware следующее этому соглашению с помощью extension метода у IApplicationBuilder: `UseMiddleware<T>`. Любые параметры переданные в этот метод, будут внедрены в конструктор middleware после next request delegate и запрошенных сервисов. По-соглашению, middleware должно быть определенно как "Use..." extension метод у IApplicationBuilder:

	public static class BuilderExtensions
	{
	  public static IApplicationBuilder UseXHttpHeaderOverride(
	    this IApplicationBuilder builder)
	  {
	    return builder.UseMiddleware<XHttpHeaderOverrideMiddleware>();
	  }
	}

Чтобы включить это middleware в request pipeline, вам необходимо вызвать этот extension метод в Configure методе вашего веб-приложения:

	public class Startup
	{
	  public void Configure(IApplicationBuilder app)
	  {
	    app.UseXHttpHeaderOverride();
	  }
	}

ASP.NET 5 поставляется с большим набором встроенных middleware. Есть middleware для работы с [файлами](https://docs.asp.net/en/latest/fundamentals/static-files.html), [маршрутизации](https://docs.asp.net/en/latest/fundamentals/routing.html), обработки ошибок, [диагностики](https://docs.asp.net/en/latest/fundamentals/diagnostics.html) и безопасности. Middleware поставляются как NuGet пакеты через nuget.org.

## Конфигурирование сервисов ##