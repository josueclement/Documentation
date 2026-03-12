# Create new solution
dotnet new sln -n MySolution --format slnx

# Class library
dotnet new classlib -n MyLibrary -o ./src/MyLibrary

# List templates
dotnet new list

# Add a single project
dotnet sln MySolution.sln add ./src/MyProject/MyProject.csproj

# Add all projects recursively
dotnet sln MySolution.sln add **/*.csproj