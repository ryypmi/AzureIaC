# Vaihe 1: Riippuvuuksien palautus
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS restore
WORKDIR /src
COPY TodoApi/TodoApi.csproj TodoApi/
RUN dotnet restore TodoApi/TodoApi.csproj

# Vaihe 2: Rakentaminen
FROM restore AS build
COPY . .
RUN dotnet publish TodoApi/TodoApi.csproj -c Release -o /app/publish --no-restore

# Vaihe 3: Tuotantoimage
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS runtime
WORKDIR /app

ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080

COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "TodoApi.dll"]
