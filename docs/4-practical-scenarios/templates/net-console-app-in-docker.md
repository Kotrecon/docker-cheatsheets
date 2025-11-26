Абсолютно прав! Упрощаю до действительно простого консольного приложения:

---

# .NET Console App в Docker

Простой и минималистичный подход к контейнеризации .NET Console приложений.

## 🎯 Задача

Создать минимальный Docker образ для простого .NET Console приложения.

## 📁 Структура проекта

```
console-app/
├── Program.cs
├── console-app.csproj
└── Dockerfile
```

## 🐳 Минимальный Dockerfile

```dockerfile
# Базовый образ .NET
FROM mcr.microsoft.com/dotnet/runtime:7.0

# Рабочая директория
WORKDIR /app

# Копируем собранное приложение
COPY bin/Release/net7.0/linux-x64/publish/ .

# Запускаем приложение
ENTRYPOINT ["dotnet", "console-app.dll"]
```

## 🚀 Сборка и запуск

### 1. Сборка приложения

```bash
dotnet publish -c Release -r linux-x64 --self-contained false
```

### 2. Сборка образа

```bash
docker build -t my-console-app .
```

### 3. Запуск контейнера

```bash
docker run --rm my-console-app
```

## 💡 Простой пример Program.cs

```csharp
// Program.cs
using System;

Console.WriteLine("Hello from Docker!");
Console.WriteLine($"Current time: {DateTime.Now}");
Console.WriteLine($"Environment: {Environment.GetEnvironmentVariable("DOTNET_ENVIRONMENT") ?? "Production"}");

// Простая работа
for (int i = 0; i < 5; i++)
{
    Console.WriteLine($"Processing item {i}");
    await Task.Delay(1000);
}

Console.WriteLine("Done!");
```

## 🔧 Минимальный .csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net7.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

</Project>
```

## 🎯 Вариант с multi-stage build

```dockerfile
# Сборка
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app/publish

# Финальный образ
FROM mcr.microsoft.com/dotnet/runtime:7.0
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "console-app.dll"]
```

## 🚀 Запуск одной командой

```bash
# Сборка и запуск сразу
docker build -t console-app . && docker run --rm console-app
```

**Всё!** Больше ничего не нужно. Простое консольное приложение в контейнере.
