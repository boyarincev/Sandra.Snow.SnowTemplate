---
layout: post
title: Погружение в ASP.NET 5 Runtime
categories: ASP-NET-5, DNX
published: draft
---

## Вступление от переводчика ##

Данная статья является переводом статьи [ASP.NET 5 - A Deep Dive into the ASP.NET 5 Runtime](https://msdn.microsoft.com/en-us/magazine/dn913182.aspx) - великолепного введения в архитектуру DNX Runtime и построенного на нем ASP.NET 5. К сожалению, так как оригинальная статья была написана в марте 2015 года, во время когда ASP.NET 5 был еще в стадии активной разработки (примерно beta 3), многое в ней устарело. Поэтому при переводе вся информация была актуализирована до текущей версии ASP.NET 5 (RC1), также были добавлены ссылки на связанные ресурсы и исходный код на GitHub (так как исходный код самая лучшая документация).

Я считаю, что эта статья важна для всех ASP.NET разработчиков собирающихся переходить на новый стек, так как дает понимание того, как работает новая версия ASP.NET и позволяет перестать работать с ней как с черным ящиком. Приятного погружения!

## .NET Runtime Environment (DNX) ##

ASP.NET базируется на гибком, кроссплатформенном Runtime, который может работать с разными .NET CLR (.NET Core, Mono, .NET Framework). Вы можете запустить ASP.NET 5 на полном .NET Framework или можете запустить на новом [.NET Core](https://docs.asp.net/en/latest/conceptual-overview/dotnetcore.html), который позволяет вам просто копировать его в существующее окружение, без изменения чего-либо еще на вашей машине. Используя .NET Core вы также можете запустить ASP.NET 5 кроссплатформенно на [Linux](https://docs.asp.net/en/latest/getting-started/installing-on-linux.html) и [Mac OS](https://docs.asp.net/en/latest/getting-started/installing-on-mac.html).

<!--excerpt-->

Инфраструктура позволяющая запускать и исполнять приложения ASP.NET 5 называется [.NET Runtime Environment или DNX](https://docs.asp.net/en/latest/dnx/overview.html). DNX предоставляет все что необходимо для разработки приложений на .NET: host process, CLR hosting логику, обнаружение управляемой Entry Point и т.д.

DNX базируется на том же самом .NET CLR и базовой библиотеке классов, что знают и любят существующие .NET разработчики и в то же время он разработан с возможностью запускать приложения (на данный момент веб и консольные) под операционными системами отличными от Windows.

Логически DNX имеет пять слоев функциональности. Я опишу каждый из этих слоев вместе с их обязанностями.

![Архитектура ASP.NET 5 и DNX](/images/asp-net-5/runtime/dnxDiagram2.jpg)

Изображение взято из статьи [DNX-structure](https://github.com/aspnet/Home/wiki/DNX-structure)

## Слой первый: Нативный процесс  ## 

Нативный процесс - это очень тонкий слой с обязанностью найти и вызвать нативный CLR host, передав в него аргументы переданные в сам процесс. В Windows - это dnx.exe (находится в %YOUR_PROFILE%/.dnx/runtimes/%CHOOSEN_RUNTIME%); В Mac и Linux - это запускаемый bash script. [Запуск на IIS](https://docs.asp.net/en/latest/publishing/iis.html) происходит с помощью устанавливаемого на IIS нативного HTTP-модуля: [HTTPPlatformHandler](https://azure.microsoft.com/en-us/blog/announcing-the-release-of-the-httpplatformhandler-module-for-iis-8/). Использование HTTPPlatformHandler позволяет запускать веб-приложение без любых зависимостей от .NET Framework (естественно при запуске веб-приложений нацеленных на .NET Core, а не на полный .NET Framework).

## Слой второй и третий: Нативный CLR host и CLR ##

Имеют три главных обязанности:

1. Запустить CLR. Способы достижения этого отличаются в зависимости от используемой версии CLR. Например, для запуска .NET Core загружается coreclr.dll, настраивается и запускается runtime и создается домен приложений, в котором будет запускаться весь managed code. Для Mono и .NET Framework, процесс отчасти различается, но результат будет тот же самый.
2. Вызвать управляемую entry point, которая является следующим слоем.
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

DNX поставляется вместе c SDK, который содержит все необходимое для построения кросс-платформенных .NET приложений.

[**DNVM** - DNX Version Manager](https://github.com/aspnet/Home/wiki/Version-Manager). Позволяет просмотреть установленные на компьютере DNX, установить новые и выбрать тот, который вы будете использовать. Прочитать про установку DNVM можно [здесь](https://docs.asp.net/en/latest/getting-started/installing-on-windows.html#install-asp-net-5-from-the-command-line).

После установки вы сможете использовать DNVM из командной строки, просто наберите `dnvm`

DNVM устанавливает DNX'ы из NuGet feed настроенного на использование DNX_FEED переменной среды. DNX'ы не являются NuGet пакетами в традиционном смысле - пакеты на которые вы можете ссылаться в качестве зависимостей. NuGet - это удобный способ доставки и управления версиями DNX. По-умолчанию DNX устанавливается копированием и распаковкой архива с DNX в "%USERPROFILE%\.dnx\runtimes".

[**DNU** - DNX Utility](https://github.com/aspnet/Home/wiki/DNX-utility). Инструмент для установки, восстановления и создания NuGet пакетов. По-умолчанию, пакеты устанавливаются в "%USERPROFILE%\.dnx\packages", но вы можете изменить это, установив другой путь в в вашем global.json файле.

Для использования DNU наберите `dnu` в командной строке

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

Команда `dnx` запускает нативный процесс (dnx.exe), выбирает текущую директорию, как корневую папку приложения (`--appbase .`) и указывает управляемой entry point (четвертый слой) вызвать Microsoft.DNX.ApplicationHost. Нативный процесс не имеет специальных знаний о Microsoft.DNX.ApplicationHost. Он [просто ищет](https://github.com/aspnet/dnx/blob/dev/src/Microsoft.Dnx.Host/Bootstrapper.cs#L80) [стандартную entry point](https://github.com/aspnet/dnx/blob/1.0.0-rc1-final/src/Microsoft.Dnx.ApplicationHost/Program.cs#L22) в сборке `Microsoft.DNX.ApplicationHost` и вызывает ее. Далее основываясь на `--appbase` ApplicationHost узнает путь до папки запускаемого проекта, а команда `run` говорит ему найти entry point этого проекта и вызвать ее.

Если вы не хотите использовать ApplicationHost. Вы можете вызвать нативный слой, передав ему ваше приложение напрямую.

Для того, чтобы сделать это:

1. Cгенерируйте DLL, запустив build вашего приложения (убедитесь, что вы поставили галочку около пункта "Produce outputs on build" в разделе "Build" свойств вашего проекта). Вы можете найти результаты в директории artifacts в корне папки вашего решения (вы также можете использовать не Visual Studio, а команду `dnu build`, набрав ее находясь в папке с файлом project.json вашего проекта).
2. Наберите команду: `dnx <путь_до_библиотеки_и_ее_имя_включая_расширение>`.

Вызов DLL напрямую - это довольно низкоуровневый подход написания приложений. Вы не используете Microsoft.DNX.ApplicationHost, поэтому вы отказываетесь и от использования файла project.json и улучшенного NuGet-based механизма управления зависимостями. Вместо этого любые библиотеки от которых вы зависите, будут загружаться из указанных при запуске нативного процесса с помощью параметра `--lib` директорий. До окончания этой статьи я буду использовать Microsoft.DNX.ApplicationHost.

Microsoft.DNX.ApplicationHost - последний слой DNX, все что находится выше можно считать ASP.NET 5.

## Хостинг веб-приложений ##

В веб-приложениях ASP.NET 5 поверх DNX ApplicationHost работает [слой хостинга](https://docs.asp.net/en/latest/fundamentals/hosting.html). Он ответственен за поиск веб-сервера, запуск веб-приложения на нем и "уборку за собой" во время выключения. Он также предоставляет приложению некоторые дополнительные, связанные со слоем хостинга [сервисы](https://github.com/aspnet/Hosting/blob/1.0.0-rc1/src/Microsoft.AspNet.Hosting/WebHostBuilder.cs#L69).

Для запуска веб-приложения DNX Application Host должен вызывать [entry point метод](https://github.com/aspnet/Hosting/blob/1.0.0-rc1/src/Microsoft.AspNet.Hosting/Program.cs) [Microsoft.AspNet.Hosting](https://github.com/aspnet/Hosting/tree/1.0.0-rc1/src/Microsoft.AspNet.Hosting). Используемый хостингом веб-сервер можно выбрать, указав опцию `--server` или использовав другие способы настройки хостинга, такие как файл hosting.json или environment variables. Слой хостинга загружает выбранный веб-сервер и стартует его. Обычно используемый сервер должен быть перечислен в списке зависимостей в файле pfoject.json, чтобы его можно было загрузить.

Заметьте, что команды определенные в вашем poject.json на самом деле только устанавливают дополнительные аргументы для `dnx.exe`. Например, шаблон веб-приложения ASP.NET 5 включает набор команд, таких как:

	  "commands": {
	    "web": "Microsoft.AspNet.Server.Kestrel",
	    "ef": "EntityFramework.Commands"
	  },

И когда вы набираете `dnx web` для запуска веб-приложения, в реальности это преобразуется в:

`dnx.exe --appbase . Microsoft.DNX.ApplicationHost Microsoft.AspNet.Server.Kestrel`

В свою очередь вызов [entry point Microsoft.AspNet.Server.Kestrel](https://github.com/aspnet/KestrelHttpServer/blob/dev/src/Microsoft.AspNet.Server.Kestrel/Program.cs) преобразуется в вызов: 

`Microsoft.AspNet.Hosting --server Microsoft.AspNet.Server.Kestrel`

Так что итоговая команда будет:

`dnx.exe --appbase . Microsoft.DNX.ApplicationHost Microsoft.AspNet.Hosting --server Microsoft.AspNet.Server.Kestrel`

В результате выполнения которой ApplicationHost вызовет entry point метод Microsoft.AspNet.Hosting.

> В данный момент, пока еще статья про хостинг в документации docs.asp.net не готова, более подробно про ключи используемые для настройки хостинга можно прочитать [здесь](https://github.com/aspnet/Announcements/issues/108)

## Стартовая логика веб-приложения ##

Слой хостинга также ответственен за [запуск стартовой логики](https://github.com/aspnet/Hosting/blob/1.0.0-rc1/src/Microsoft.AspNet.Hosting/WebApplication.cs#L56) веб-приложения. Раньше она находилась в файле Global.asax, теперь по умолчанию находится в классе Startup и состоит из Configure метода, используемого для настройки request pipeline (цепочки обработчиков входящего запроса) и ConfigureServices метода, используемого для настройки сервисов веб-приложения.

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

Для настройки request pipeline в Configure методе используется интерфейс IApplicationBuilder. IApplicationBuilder позволяет зарегистрировать request delegate ("Use" метод) и зарегистрировать middleware ("UseMiddleware" метод) в request pipeline.

Request delegate - это ключевая концепция ASP.NET 5. Request delegate - это обработчик входящего запроса, он принимает HttpContext и асинхронно делает нечто полезное с ним:

`public delegate Task RequestDelegate(HttpContext context);`

Обработка запроса в ASP.NET 5 - это вызов по цепочке зарегистрированных request delegate. Но принятие решения о вызове следующего request delegate в цепочке, остается за их автором.

В качестве упрощения создания request delegate не вызывающего следующий request delegate, вы можете использовать Run extension метод IApplicationBuilder.

`app.Run(async context => await context.Response.WriteAsync("Hello, world!"));`

Того же самого можно достичь используя Use extension метод и не вызывая следующий request delegate:

`app.Use(next => async context => await context.Response.WriteAsync("Hello, world!"));`

И пример с вызовом следующего в цепочке request delegate:

`app.Use(next => async context => await context.Response.WriteAsync("Hello, world!"));`

Для того, чтобы request delegate было удобно переиспользовать, можно оформить его в виде [ASP.NET 5 middleware](https://docs.asp.net/en/latest/fundamentals/middleware.html).

Middleware ASP.NET 5 - это обычный класс, следующий определенному соглашению. Следующий в цепочке request delegate (а также необходимые сервисы и дополнительные параметры) передается в конструктор middleware. Логика обработки HttpContext должна быть реализована в асинхронном Invoke методе, как это показано ниже:

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

Вызов следующего (если вы хотите вызвать следующий) в цепочке request delegate должен осуществляться внутри Invoke метода. Если вы разместите какую-нибудь логику ниже вызова следующего request delegate, то она будет выполнена после того, как все следующие за вашим обработчики входящего запроса отработают.

В request pipeline вы можете включить middleware следующее этому соглашению с помощью `UseMiddleware<T>` extension метода у IApplicationBuilder:

    public void Configure(IApplicationBuilder app)
    {
      app.UseMiddleware<XHttpHeaderOverrideMiddleware>();
    }

Любые параметры переданные в этот метод, будут внедрены в конструктор middleware после `RequestDelegate next` и запрошенных сервисов (о сервисах поговорим ниже):

	//конструктор middleware  
    public XHttpHeaderOverrideMiddleware(RequestDelegate next, SomeServise1 service1, 
		SomeServise2 service2, string param1, bool param2)
    {
      _next = next;
    }
	
	//включение в request pipeline
    public void Configure(IApplicationBuilder app)
    {
      app.UseMiddleware<XHttpHeaderOverrideMiddleware>(param1, param2);
    }

По-соглашению, включение middleware в цепочку вызовов следует оформить в виде "Use..." extension метода у IApplicationBuilder:

	public static class BuilderExtensions
	{
	  public static IApplicationBuilder UseXHttpHeaderOverride(
	    this IApplicationBuilder builder)
	  {
	    return builder.UseMiddleware<XHttpHeaderOverrideMiddleware>();
	  }
	}

Чтобы включить это middleware в request pipeline, вам необходимо вызвать этот extension метод в Configure методе класса Startup:

	public class Startup
	{
	  public void Configure(IApplicationBuilder app)
	  {
	    app.UseXHttpHeaderOverride();
	  }
	}

ASP.NET 5 поставляется с большим набором встроенных middleware. Есть middleware для работы с [файлами](https://docs.asp.net/en/latest/fundamentals/static-files.html), [маршрутизации](https://docs.asp.net/en/latest/fundamentals/routing.html), обработки ошибок, [диагностики](https://docs.asp.net/en/latest/fundamentals/diagnostics.html) и безопасности. Middleware поставляются как NuGet пакеты через nuget.org.

## Сервисы ##

В ASP.NET 5 вводится понятие Сервисов - "общих" компонентов, доступ к которым может требоваться в нескольких местах приложения. Сервисы доступны приложению через систему dependency injection. ASP.NET 5 поставляется с простым [IoC-контейнером](https://docs.asp.net/en/latest/fundamentals/dependency-injection.html), поддерживающим внедрение зависимостей в конструктор, но вы легко можете [заменить](https://docs.asp.net/en/latest/fundamentals/dependency-injection.html#replacing-the-default-services-container) его на любой другой контейнер.

Startup класс также поддерживает внедрение зависимостей, для этого достаточно запросить их в качестве параметров конструктора.

По умолчанию вам доступны следующие сервисы:

`Microsoft.Extensions.PlatformAbstractions.IApplicationEnvironment` - информация о приложении (физический путь до папки приложения, его имя, версия, конфигурация (Release, Debug), используемый Runtime фреймворк).

`Microsoft.Extensions.PlatformAbstractions.IRuntimeEnvironment` - информация о DNX runtime и ОС.

`Microsoft.AspNet.Hosting.IHostingEnvironment` - доступ к Web-root вашего приложения (обычно папка "wwwroot"), а также информация о текущей среде(dev, stage, prod).

`Microsoft.Extensions.Logging.ILoggerFactory` - фабрика для создания [логгеров](https://docs.asp.net/en/latest/fundamentals/logging.html).

`Microsoft.AspNet.Hosting.Builder.IApplicationBuilderFactory` - фабрика для создания IApplicationBuilder (используется для построения request pipeline).

`Microsoft.AspNet.Http.IHttpContextFactory` - фабрика для создания Http-контекста.

`Microsoft.AspNet.Http.IHttpContextAccessor` - предоставляет доступ к текущему Http-контексту.

Вы настраиваете существующие сервисы и добавляете новые в ConfigureServices методе класса Startup с помощью интерфейса IServiceCollection.

Обычно фреймворки и библиотеки предоставляют "Add..." extension метод у IServiceCollection для добавления их сервисов в IoC-контейнер. Например, добавление сервисов используемых ASP.NET MVC 6 производится так:

	public void ConfigureServices(IServiceCollection services)
	{
	  // Добавление MVC сервисов в сервис-контейнер
	  services.AddMvc();
	}

Вы можете добавлять собственные сервисы в IoC-контейнер. Добавляемые сервисы могут быть одними из трех типов: transient (AddTransient метод), scoped (AddScoped метод) или singleton (AddSingleton метод). Transient сервисы создаются при каждом их запросе из контейнера. Scoped сервисы создаются только если они еще не создавались в текущем scope. В веб-приложениях scope-контейнер создается для каждого запроса, поэтому можно думать о них как о сервисах создаваемых для каждого http запроса. Singleton сервисы создаются только один раз за цикл жизни приложения.

>В консольном приложении, где доступ к dependency injection отсутствует, для доступа к сервисам: IApplicationEnvironment и IRuntimeEnvironment необходимо использовать статический объект `Microsoft.Extensions.PlatformAbstractions.PlatformServices.Default`.

## Конфигурация приложения ##

Web.config и app.config файлы больше не поддерживаются. Вместо них ASP.NET 5 использует новое, упрощенное [Configuration API](https://docs.asp.net/en/latest/fundamentals/configuration.html). Оно позволяет получать данные из разных источников. Используемые по-умолчанию configuration-провайдеры поддерживают JSON, XML, INI, аргументы командной строки, environment variables, а также установку параметров прямо из кода (in-memory collection). Вы можете указать несколько источников и они будут использоваться в порядке их добавления (добавленные последними будут переопределять настройки добавленных ранее). Также вы [можете иметь разные настройки для каждой среды](https://docs.asp.net/en/latest/fundamentals/environments.html) (test, stage, prod), что облегчает публикацию приложения в разные среды.

Пример получения настроек приложения, используя Configuration API приведен ниже:

	var builder = new ConfigurationBuilder()
	            .AddJsonFile("appsettings.json")
	            .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true)
				.AddEnvironmentVariables();

    var Configuration = builder.Build();

Запросить данные, вы можете используя метод GetSection и имя настройки:

	var user = Configuration.GetSection("user");
	var password = Configuration.GetSection("password");

Работать с Configuration API рекомендуется в Startup классе, а в дальнейшем разделять настройки на небольшие наборы данных, соответствующие какой-либо функциональности и передавать другим частям приложения с помощью механизма Options.

[Options](https://docs.asp.net/en/latest/fundamentals/configuration.html#using-options-and-configuration-objects) позволяют использовать Plain Old CLR Object (POCO) классы в качестве объектов с настройками. Вы можете добавить механизм Options в ваше приложение вызвав AddOptions extension-метод у IServiceCollection в ConfigureServices методе. Фактически, вызов AddOptions добавляет `IOptions<TOption>` сервис в сервис-контейнер, Этот сервис может быть использован для получения Options разных типов, везде где dependency injection доступно (достаточно лишь запросить из системы dependency injection `IOption<TOption>`, где TOption POCO класс с настройками).

Внутренне механизм Options работает через добавление `IConfigureOptions<TOptions>` в сервис-контейнер, где TOptions - класс с настройками. Стандартная реализация `IOptions<TOption>` будет собирать все `IConfigureOptions<TOptions>` одного типа и "суммировать их свойства", а затем предоставлять запросившему конечный экземпляр - это происходит потому что вы можете множество раз добавлять объект с настройками одного и того же типа в сервис-контейнер, переопределяя настройки.

Для добавлении новой options вы можете использовать `Configure<TOption>` extension-метод у IServiceCollection:


	public void ConfigureServices(IServiceCollection services)
	{
		services.Configure<MvcOptions>(options => options.Filters.Add(
		  new MyGlobalFilter()));
	}

Вы также можете легко передать часть конфигурационных настроек в options:

	services.Configure<MyOptions>(Configuration);

В этом случае имена настроек из конфигурации будут мапиться на имена свойств класса настроек.

## Веб-сервер ##

Как только веб-сервер стартует он начинает ожидать входящие запросы и запускать процесс обработки	 для каждого из них. Уровень веб-сервера, поднимает запрос на уровень хостинга, отправляя ему набор feature интерфейсов. Есть feature интерфейсы для отправки файлов, веб-сокетов, поддержки сессий, клиентских сертификатов и многих других, вы можете увидеть полный список поддерживаемых feature интерфейсов [здесь](https://docs.asp.net/en/latest/fundamentals/request-features.html).

Например feature интерфейс для Http request:

	namespace Microsoft.AspNet.Http.Features
	{
	    public interface IHttpRequestFeature
	    {
	        string Protocol { get; set; }
	        string Scheme { get; set; }
	        string Method { get; set; }
	        string PathBase { get; set; }
	        string Path { get; set; }
	        string QueryString { get; set; }
	        IHeaderDictionary Headers { get; set; }
	        Stream Body { get; set; }
	    }
	}

Веб-сервер использует feature интерфейсы для раскрытия низкоуровневой функциональности уровню хостинга. А он в свою очередь делает доступными их всему приложению, через `HttpContext`. Это позволяет разорвать тесные связи между уровнем веб-сервера и хостинга и размешать приложение на различных веб-серверах. ASP.NET 5 поставляется с встроенной поддержкой IIS, оберткой над HTTP.SYS ([Microsoft.AspNet.Server.Web­Listener](https://www.nuget.org/packages/Microsoft.AspNet.Server.WebListener)) и новым кроссплатформенным веб-сервером под названием [Kestrel](https://github.com/aspnet/KestrelHttpServer).

Open Web Interface for .NET (OWIN) стандарт разделяющий эти же цели. OWIN стандартизирует как .NET сервера и приложения должны общаться друг с другом. ASP.NET 5 [поддерживает OWIN](https://docs.asp.net/en/latest/fundamentals/owin.html) с помощью [Microsoft.AspNet.Owin](https://github.com/aspnet/HttpAbstractions/tree/1.0.0-rc1/src/Microsoft.AspNet.Owin) пакета. Вы [можете хостить](https://github.com/aspnet/Entropy/tree/dev/samples/Owin.Nowin.HelloWorld) ASP.NET 5 приложения на OWIN-based веб-серверах и вы можете [использовать OWIN middleware](https://github.com/aspnet/Entropy/tree/dev/samples/Owin.HelloWorld) в ASP.NET 5 pipeline.

[Katana Project](http://katanaproject.codeplex.com) была первой попыткой Microsoft реализовать поддержку OWIN на стеке ASP.NET и многие идеи и концепты были перенесены из нее в ASP.NET 5. Katana имеет похожую модель построения pipeline из middleware и хостинга на различных веб-серверах. Однако, в отличие от Katana, которая открывает ниже лежащий уровень OWIN приложению, ASP.NET 5 переходит к более удобным абстракциям. Но вы до сих пор можете использовать Katana middleware в ASP.NET 5 с помощью [OWIN моста](https://github.com/aspnet/Entropy/tree/dev/samples/Owin.IAppBuilderBridge)

## Итоги ##

ASP.NET 5 runtime построен с нуля для поддержки кроссплатформенных веб-приложений. ASP.NET 5 имеет гибкую, многослойную архитектуру, которая может запускаться на полном .NET Framework, .NET Core и даже на Mono. Новая хостинг модель позволяет легко компоновать приложения, используя middleware, хостить их на различных веб-серверах, а также поддерживает dependency injection и новые улучшенные возможности по конфигурированию приложений.

## Cсылки ##

[docs.asp.net](https://docs.asp.net/en/latest/index.html) - документация ASP.NET 5.

[github.com/aspnet/Home/wiki](https://github.com/aspnet/Home/wiki) - статьи про DNX Runtime.