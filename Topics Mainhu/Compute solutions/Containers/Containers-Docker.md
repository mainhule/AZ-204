# Docker Best Practices

## Multi-stage builds

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /app
COPY . .
RUN dotnet publish -o out

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:6.0
WORKDIR /app
COPY --from=build /app/out .
ENTRYPOINT ["dotnet", "app.dll"]
```

## Base images
- dotnet/sdk → Voor builden
- dotnet/aspnet → Voor draaien
