# Creating and Publishing .NET Templates

## 1. Set Up the Template Structure

A .NET template is just a regular project (or solution) with a special `.template.config` folder containing a `template.json` file. Here's the typical folder layout:

```
MyTemplates/
├── templates/
│   └── my-webapi/          ← your actual project
│       ├── .template.config/
│       │   └── template.json
│       ├── MyWebApi.csproj
│       ├── Program.cs
│       └── ...
├── MyName.Templates.csproj  ← the NuGet pack project
└── README.md
```

## 2. Create `template.json`

Inside your project's `.template.config/template.json`:

```json
{
  "$schema": "http://json.schemastore.org/template",
  "author": "Your Name",
  "classifications": ["Web", "API"],
  "identity": "MyName.WebApi",
  "name": "My Custom Web API",
  "shortName": "my-webapi",
  "tags": {
    "language": "C#",
    "type": "project"
  },
  "sourceName": "MyWebApi",
  "preferNameDirectory": true,
  "symbols": {
    "useAuth": {
      "type": "parameter",
      "datatype": "bool",
      "defaultValue": "false",
      "description": "Whether to include JWT authentication setup"
    }
  },
  "sources": [
    {
      "modifiers": [
        {
          "condition": "(!useAuth)",
          "exclude": ["Auth/**"]
        }
      ]
    }
  ]
}
```

Key fields to understand: `shortName` is what people type after `dotnet new`, and `sourceName` is the token that gets replaced with whatever the user passes via `--name`. So if someone runs `dotnet new my-webapi --name Acme.Api`, every occurrence of "MyWebApi" in file names and content gets replaced with "Acme.Api".

The `symbols` section lets you define custom parameters (like `--useAuth`) that users can pass when creating the template, and you can use conditionals in `sources` to include/exclude files or inside your source files with `#if (useAuth)` directives.

## 3. Create the NuGet Pack Project

The `MyName.Templates.csproj` at the root is what produces the installable NuGet package:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <PackageType>Template</PackageType>
    <PackageVersion>1.0.0</PackageVersion>
    <PackageId>MyName.Templates</PackageId>
    <Title>My Custom Templates</Title>
    <Authors>Your Name</Authors>
    <Description>Custom .NET project templates</Description>
    <PackageTags>dotnet-new;template</PackageTags>
    <TargetFramework>net8.0</TargetFramework>
    <IncludeContentInPack>true</IncludeContentInPack>
    <IncludeBuildOutput>false</IncludeBuildOutput>
    <ContentTargetFolders>content</ContentTargetFolders>
    <NoDefaultExcludes>true</NoDefaultExcludes>
    <NoWarn>$(NoWarn);NU5128</NoWarn>
  </PropertyGroup>

  <ItemGroup>
    <Content Include="templates/**/*" Exclude="templates/**/bin/**;templates/**/obj/**" />
    <Compile Remove="**/*" />
  </ItemGroup>
</Project>
```

The important bits: `PackageType` must be `Template`, `IncludeBuildOutput` is false because you're not shipping a compiled DLL, and the `Content` item pulls in everything under `templates/`.

## 4. Pack and Test Locally

```bash
# Pack it
dotnet pack -o ./nupkg

# Install locally to test
dotnet new install ./nupkg/MyName.Templates.1.0.0.nupkg

# Try it out
dotnet new my-webapi --name TestProject --useAuth true

# Uninstall when done testing
dotnet new uninstall MyName.Templates
```

## 5. Publish to NuGet.org

Once you're happy with the template:

```bash
# Push to NuGet (get your API key from nuget.org/account/apikeys)
dotnet nuget push ./nupkg/MyName.Templates.1.0.0.nupkg \
  --api-key YOUR_API_KEY \
  --source https://api.nuget.org/v3/index.json
```

After NuGet indexes it (usually a few minutes), anyone can install it with:

```bash
dotnet new install MyName.Templates
```

## A Few Tips

**Conditional content inside files** works with the standard C# preprocessor syntax. In a `.cs` file you can write `#if (useAuth)` / `#endif` blocks, and in JSON files you can use `//` comment-style conditions — the template engine strips them during generation.

**Multiple templates in one package** — just add more folders under `templates/`, each with its own `.template.config/template.json`. The single NuGet package installs all of them at once.

**Solution-level templates** are also possible. Set `"tags": { "type": "solution" }` and include a `.sln` file. The `sourceName` replacement works across the entire solution tree.

If you want to go deeper, the official docs at `github.com/dotnet/templating` cover advanced features like post-creation actions, computed symbols, and template groups.