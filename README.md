# blazorwasmevalexample
Demonstration on how to dynamically evaluate C# code in Blazor WebAssembly (similar to CSharpScript.RunAsync with Roslyn which won't work in Blazor WebAssembly)

For starters: CSharpScript.RunAsync throws an exception in Blazor WebAssembly since it is trying to find a file on disk, although I/O operations
are restricted by the browser for security purposes.

### 1. Install NuGet package
First, we need to install a NuGet package called `Microsoft.CodeAnalysis.CSharp`. This is something known as ".NET Compiler Platform" or "Roslyn", a compiler
platform used by Visual Studio and many other services.

On Visual Studio, click Project -> Manage NuGet Packages, click the Browse tab, search `Microsoft.CodeAnalysis.CSharp` and install the latest version.

In the .NET CLI, open any terminal, 'cd' into your project directory, and type:
```cs
dotnet add our-good-app.csproj package Microsoft.CodeAnalysis.CSharp
```

### 2. Copy dll file
This is an important step. First, build your project.

In your project directory, go to the folder `bin/Debug/net8.0`. In there, find the file *System.Private.CoreLib.dll*.

In the wwwroot folder, create a folder called 'dll', and copy this System.Private.CoreLib.dll file into the 'dll' folder in your wwwroot directory.

And after this, put this line of code into the very top of the razor file:
```
@inject HttpClient Http
```

### 3. Evaluating code
Now that we have the necessary resources for evaluating C# code, we can begin doing so. The plan is to compile C# code into a DLL file in memory,
and then load it from memory using `Assembly.Load` method. That way, we're not writing anything to the disk, but we do compile the program, we just store
its bytes directly into a byte array and load it from there.

Add the following lines to the very top of the razor file:
```
﻿@using System.IO
@using System.Reflection
@using Microsoft.CodeAnalysis
@using Microsoft.CodeAnalysis.Emit
@using Microsoft.CodeAnalysis.CSharp
@inject HttpClient Http
@inject IJSRuntime JavaScriptRuntime
```

Here is the code to do this:
```cs
// This is the code we want to load.
SyntaxTree syntaxTree = CSharpSyntaxTree.ParseText(@"
public class MyClass
{
    public static int DoMath(int x, int y) => x + (x * y);
}"
);

// Now we read byte array from the System.Private.CoreLib.dll file in wwwroot/dll folder,
// to prevent getting hundreds of compiler errors during evaluation, and we import types
// from System.Private.CoreLib.dll, such as int or object
MetadataReference[] references =
[
    MetadataReference.CreateFromImage(await Http.GetByteArrayAsync("dll/System.Private.CoreLib.dll"))
];

// We create a new CSharpCompilation in order to compile our program.
// .WithOptions(...) is so that we don't need the Main method explicitly
CSharpCompilation compilation = CSharpCompilation.Create(
    "OurGoodAssemblyForTesting",
    syntaxTrees: new[] { syntaxTree },
    references: references
)
.WithOptions(new CSharpCompilationOptions(OutputKind.DynamicallyLinkedLibrary));

// Create a new MemoryStream. This is where the byte array of the compiled assembly will be stored.
using (var ms = new MemoryStream())
{
    // Compile the program.
    EmitResult result = compilation.Emit(ms);

    // If the program failed to compile, throw an exception and show all compiler errors.
    if (!result.Success)
    {
        IEnumerable<Diagnostic> failures = result.Diagnostics.Where(diagnostic => 
            diagnostic.IsWarningAsError || 
            diagnostic.Severity == DiagnosticSeverity.Error);

        string message =
            string.Join(", ", failures.Select(failure => $"{failure.Id}: {failure.GetMessage()}"));
        throw new InvalidOperationException(message);
    }

    // Here, create a new System.Reflection.Assembly and pass in the byte array of the compiled assembly, effectively
    // compiling C# code and loading the compiled assembly in memory.
    ms.Seek(0, SeekOrigin.Begin);
    Assembly assembly = Assembly.Load(ms.ToArray());

    // Finally, let's invoke MyClass.DoMath and pass in two parameters: 5 and 5.
    foreach (Type type in assembly.GetTypes())
    {
        if (type.Name == "MyClass")
        {
            MethodInfo? methodInfo = type.GetMethod("DoMath");
            if (methodInfo is not null)
            {
                int? invokeResult = (int?)methodInfo.Invoke(null, [5, 5]);

                if (invokeResult != 30)
                {
                    // This is where we know something went wrong...
                    throw new InvalidOperationException($"expected 30 got {result}");
                }
                else
                {
                    // If everything went as expected, we log "Success! The result of compilation/method invocation equals 30."
                    // into the browser console.
                    await JavaScriptRuntime.InvokeVoidAsync(
                        "console.log",
                        "Success! The result of compilation/method invocation equals 30."
                    );
                }
            }
        }
    }
}
```

# Ready Example
[This](https://github.com/winscripter/blazorwasmevalexample/blob/main/our-good-app/Pages/Counter.razor) file contains the entire razor page that demonstrates
the dynamic evaluation of C# code. If you compile 'our-good-app' and click on the Counter and the button in the Counter page, the webpage might freeze for a couple
seconds, because it is evaluating C# code behind the scenes. And well, compiling C# code takes time.
