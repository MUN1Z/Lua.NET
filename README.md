# Lua.NET
![Logo](https://raw.githubusercontent.com/tilkinsc/Lua.NET/main/Lua.NET.Logo.png)

C# .NET Core 7.0 Lua bindings and helper functions.

https://github.com/tilkinsc/Lua.NET  
Copyright © Cody Tilkins 2022 MIT License  

Supports Lua5.4 Lua5.3 Lua5.2 Lua5.1 and LuaJIT  

Hardcoded to only use doubles and 64-bit integers.
This CAN be changed with manual edits, but it wasn't fun writing this library.
This code was made with with the default includes on a 64-bit windows 10 machine using Lua's and LuaJIT's makefiles.
No DLL has to be built for this library as its included for you.

Custom DLLs are supported as long as they don't change any call arguments or return values.

To build Lua, get the Lua source from [Lua.org](https://www.lua.org/download.html) or [LuaJIT](https://luajit.org/download.html).
```bat
make -j24
```
Then rename the dll to the above convention.

# Design Considerations / Usage

Your delegates you pass to functions such as `lua_pushcfunction(...)` should be static.
If you do not use static, then the lifetime of your functions should exceed the lifetime of the Lua the final Lua context you create during the course of the program.
Do not use lambdas.
C# is liable to GC your delegates otherwise.

There are functions prefixed with an underscore.
These functions denote raw DllImported functions.
The reason these exist is because some functions needed a shim function for it to work properly/sanely, i.e. marshaling.
You can write your own functions against those.
For example, if you want a function like lua_pcall but not have to specify an error handler offset you can invoke _lua_pcall(...) in a util class (all functions are static).
This library does not use unsafe, however, going unsafe should work perfectly.
If you are just here to use the library, you can get by without having to worry about the underscore prefixed functions.

# Examples

Example Usage Lua5.4.4:
```C#
using Lua54;
using static Lua54.Lua;

namespace LuaNET;

class Project
{
	
	public static int lfunc(lua_State L)
	{
		lua_getglobal(L, "print");
		lua_pushstring(L, "lol");
		lua_pcallk(L, 1, 0, 0, IntPtr.Zero, null);
		return 0;
	}
	
	public static void Main(String[] args)
	{
		lua_State L = luaL_newstate();
		if (L.Handle == UIntPtr.Zero)
		{
			Console.WriteLine("Unable to create context!");
		}
		luaL_openlibs(L);
		lua_pushcfunction(L, lfunc);
		Console.WriteLine(lua_pcallk(L, 0, 0, 0, null, null));
		lua_close(L);
		
		Console.WriteLine("Success");
	}
	
}
```

Example Usage LuaJIT:
```C#
using LuaJIT;
using static LuaJIT.Lua;

namespace LuaNET;

class Project
{
	
	public static int lfunc(lua_State L)
	{
		lua_getglobal(L, "print");
		lua_pushstring(L, "lol");
		lua_pcall(L, 1, 0, 0);
		return 0;
	}
	
	public static void Main(String[] args)
	{
		lua_State L = luaL_newstate();
		if (L.Handle == UIntPtr.Zero)
		{
			Console.WriteLine("Unable to create context!");
		}
		luaL_openlibs(L);
		lua_pushcfunction(L, lfunc);
		Console.WriteLine(lua_pcall(L, 0, 0, 0));
		lua_close(L);
		
		Console.WriteLine("Success");
	}
	
}
```

Example Usage NativeAOT DLL Library:
```C#
// test2.csproj
// <PropertyGroup>
//     ...
//     <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
//     <PublishAot>true</PublishAot>
// </PropertyGroup>

using System.Runtime.InteropServices;
using LuaJIT;
using static LuaJIT.Lua;

namespace test2;

public unsafe class Test2
{
	
	[UnmanagedCallersOnly]
	public static int l_GetHello(lua_State L)
	{
		lua_pushstring(L, "Hello World!");
		return 1;
	}
	
	public static luaL_Reg[] test2Lib = new luaL_Reg[]
	{
		AsLuaLReg("GetHello", &l_GetHello),
		AsLuaLReg(null, null)
	};
	
	[UnmanagedCallersOnly(EntryPoint = "luaopen_test2")]
	public static int luaopen_test2(lua_State L)
	{
		luaL_newlib(L, test2Lib);
		lua_setglobal(L, "test2");
		return 1;
	}
	
}

// test1.csproj

using LuaJIT;
using static LuaJIT.Lua;

namespace test1;

public class Test1
{
	
	public static void Main(string[] args)
	{
		lua_State L = luaL_newstate();
		if (L.Handle == UIntPtr.Zero)
		{
			Console.WriteLine("Unable to create context!");
		}
		luaL_openlibs(L);
		
		// require("test2.dll");
		lua_getglobal(L, "require");
		lua_pushstring(L, "test2");
		int result = lua_pcall(L, 1, 0, 0);
		if (result != LUA_OK)
		{
			string? err = luaL_checkstring(L, 1);
			if (err != null)
			{
				Console.WriteLine($"1 Error Result: {err}");
			}
		}
		
		// print(_G.test2.GetHello())
		lua_getglobal(L, "test2");
		lua_getglobal(L, "print");
		lua_getfield(L, 1, "GetHello");
		result = lua_pcall(L, 0, 1, 0);
		if (result != LUA_OK)
		{
			string? err = luaL_checkstring(L, 1);
			if (err != null)
			{
				Console.WriteLine($"2 Error Result: {err}");
			}
		}
		result = lua_pcall(L, 1, 0, 0);
		if (result != LUA_OK)
		{
			string? err = luaL_checkstring(L, 1);
			if (err != null)
			{
				Console.WriteLine($"3 Error Result: {err}");
			}
		}
		// pop test2
		lua_pop(L, 1);
		
		lua_close(L);
		
		Console.WriteLine("Success!");
	}
	
}
```
