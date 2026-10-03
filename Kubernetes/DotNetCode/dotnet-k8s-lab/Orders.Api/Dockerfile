FROM mcr.microsoft.com/dotnet/sdk:9.0

WORKDIR /src

COPY . .

RUN dotnet publish -c Release -o /app

WORKDIR /app

ENTRYPOINT [ "dotnet", "Orders.Api.dll" ]