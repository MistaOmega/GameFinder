# GameFinder

[![CI](https://github.com/erri120/GameFinder/actions/workflows/ci.yml/badge.svg)](https://github.com/erri120/GameFinder/actions/workflows/ci.yml) [![Nuget](https://img.shields.io/nuget/v/GameFinder)](https://www.nuget.org/packages/GameFinder/)

.NET 5 library for finding games on Windows.

## Supported Stores

- [Steam](#finding-steam-games)
- [GOG](#finding-gog-games)
- [Bethesda.net Launcher](#finding-bethnet-games)
- [Epic Games Store](#finding-egs-games)
- [Xbox Game Pass](#finding-xbox-games) (UWP apps, see [Finding Xbox Games](#finding-xbox-games) for more information)
- ~~Origin~~ _TODO_

## Example

```c#
var steamHandler = new SteamHandler();
steamHandler.FindAllGames();
foreach (var steamGame in steamHandler.Games)
{
    Console.WriteLine($"{steamGame} is located at {steamGame.Path}");
}
```

## When to use this Library

This library is best used when you have to find multiple different games from different game stores. If you build a tool for only one game then you might better be off finding it via the registry (common method is looking for the Uninstaller). You could still use this library to only find that one game but it's a bit overkill.

If you build some sort of game library manager like [Playnite](https://github.com/JosefNemec/Playnite/) or [LaunchBox](https://www.launchbox-app.com/) then you could make massive use of this library.

## How it works

### Finding Steam Games

Source: [SteamHandler.cs](GameFinder.StoreHandlers.Steam/SteamHandler.cs)

Steam games can be easily found by searching through _"Steam Universes"_. An Universe is simply a folder where you install Steam games. You can find all Universes by parsing some configuration files in the Steam folder. We can get the Steam folder by opening the registry key `HKEY_CURRENT_USER\Software\Valve\Steam` and getting the `SteamPath` value.

Steam changed the format of their configuration files multiple times which is why you have to differentiate between different formats:

#### Steam Versions before 1623193086 (2021-06-08)

The `config/config.vdf` file uses Valve's KeyValue format which is similar to JSON. What we want to look for are these `BaseInstallFolder_X` values which point to a Universe folder.

```text
"InstallConfigStore"
{
	"Software"
	{
		"Valve"
		{
			"Steam"
			{
				"BaseInstallFolder_1"		"F:\\SteamLibrary"
				"BaseInstallFolder_3"		"E:\\SteamLibrary"
			}
		}
	}
}
```

The `steamapps/libraryfolders.vdf` uses the same type of formatting as above. The lines with a numeric key point to a Universe folder. These numbers should match up with the `BaseInstallFolder_X` values in `config/config.vdf`.

```text
"LibraryFolders"
{
	"TimeNextStatsReport"		"1623187700"
	"ContentStatsID"		"-8832980547670729777"
	"1"		"F:\\SteamLibrary"
	"3"		"E:\\SteamLibrary"
}
```

Both `config/config.vdf` and `steamapps/libraryfolders.vdf` are parsed for possible Universe locations. Steam should keep these files synced but there have been some user reports of this not happening.

#### Steam Versions after 1623193086 (2021-06-08)

In the new format only `steamapps/libraryfolders.vdf` contains the Universe locations:

```text
"libraryfolders"
{
	"contentstatsid"		"1616900521946793171"
	"1"
	{
		"path"		"M:\\SteamLibrary"
		"label"		""
		"mounted"		"1"
		"contentid"		"5393764431939341339"
	}
	"2"
	{
		"path"		"E:\\SteamLibrary"
		"label"		""
		"mounted"		"1"
		"contentid"		"6734528685854492248"
	}
}
```

They now store an array of objects instead of strings which contain additional information like `label` and `mounted`.

#### Parsing appmanifest files

The `steamapps` subdirectory contains the `appmanifest_*.acf` files we need. `.acf` files have the same KeyValue format as `.vdf` files so parsing is very easy:

```text
"AppState"
{
	"appid"		"8930"
	"Universe"		"1"
	"LauncherPath"		"C:\\Program Files (x86)\\Steam\\steam.exe"
	"name"		"Sid Meier's Civilization V"
	"StateFlags"		"4"
	"installdir"		"Sid Meier's Civilization V"
	"LastUpdated"		"1600350073"
	"UpdateResult"		"0"
	"SizeOnDisk"		"9235434479"
	"buildid"		"4390913"
	"LastOwner"		"76561198110222274"
	"BytesToDownload"		"20736"
	"BytesDownloaded"		"20736"
	"BytesToStage"		"26039"
	"BytesStaged"		"26039"
}
```

Important in this file are the `appid`, `name` and `installdir` fields. Note: `installdir` is the name of the folder in `Universe/steamapps/common/` where the game is installed. It is not absolute but relative to the `common` folder.

### Finding GOG Games

Source: [GOGHandler.cs](GameFinder.StoreHandlers.GOG/GOGHandler.cs)

GOG stores all information in the registry. This can either be at `HKEY_LOCAL_MACHINE\Software\GOG.com\Games` or `HKEY_LOCAL_MACHINE\Software\WOW6432Node\GOG.com\Games`. Simply open the registry key and get all sub-key names. Each sub-key in `GOG.com\Games` is an installed game with the ID being the name of the sub-key:

![GOG Games Tree in Registry](assets/docs/readme-gog-1.png)

Now you can iterate over all sub-keys to get all the information you need:

![GOG Game in Registry](assets/docs/readme-gog-2.png)

Important fields are `path`, `gameID` and `gameName`.

### Finding BethNet Games

Source: [BethNetHandler.cs](GameFinder.StoreHandlers.BethNet/BethNetHandler.cs)

Finding games installed with the Bethesda.net Launcher was very rather tricky because there are no config files you can parse or simple registry keys you can open. I ended up using a similar method to the GOG Galaxy Bethesda.net plugin by TouwaStar: [GitHub](https://github.com/TouwaStar/Galaxy_Plugin_Bethesda). The interesting part is the `_scan_games_registry_keys` function in [`betty/local.py`](https://github.com/TouwaStar/Galaxy_Plugin_Bethesda/blob/master/betty/local.py#L154):

1) open the uninstaller registry key at `HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall`
2) iterate over every sub-key:
    - find the sub-keys that open the Bethesda Launcher with `bethesdanet://uninstall/` as an argument

![Bethesda.net Launcher Games Uninstaller in Registry](assets/docs/readme-bethnet-1.png)

With this you can find all games installed via Bethesda.net. The important fields are `DisplayName`, `ProductID` (64bit value) and `Path`.

### Finding EGS Games

Source: [EGSHandler.cs](GameFinder.StoreHandlers.EGS/EGSHandler.cs)

The Epic Games Store uses manifest files, similar to Steam, which contain all information we need. The path to the manifest folder can be found by opening the registry key `HKEY_CURRENT_USER\SOFTWARE\Epic Games\EOS` and getting the `ModSdkMetadataDir` value. Inside the manifest folder you will find `.item` files which are actually just JSON files with a different extension.

See [`8AAFB83044E76B812D3D8C9652E8C13C.item`](GameFinder.Tests/files/8AAFB83044E76B812D3D8C9652E8C13C.item) for an example file. Important fields are `InstallLocation`, `DisplayName` and `CatelogItemId`.

### Finding Xbox Games

Source: [XboxHandler.cs](GameFinder.StoreHandlers.Xbox/XboxHandler.cs)

These games are installed through the Xbox Game Pass app or the Windows Store. These games are UWP packages and in a UWP container. I had to use the Windows 10 SDK to get all packages:

```c#
internal static IEnumerable<Package> GetUWPPackages()
{
   var manager = new PackageManager();
   var user = WindowsIdentity.GetCurrent().User;
   var packages = manager.FindPackagesForUser(user.Value)
       .Where(x => !x.IsFramework && !x.IsResourcePackage && x.SignatureKind == PackageSignatureKind.Store)
       .Where(x => x.InstalledLocation != null);
   return packages;
}
```

Since the packages uses the Windows 10 SDK I had to change the project settings to reflect that:

```xml
 <PropertyGroup>
     <TargetFrameworks>net5.0-windows10.0.18362.0;netstandard2.1</TargetFrameworks>
 </PropertyGroup>
 
 <ItemGroup Condition="'$(TargetFramework)' == 'netstandard2.1'">
     <Reference Include="Windows, Version=255.255.255.255, Culture=neutral, PublicKeyToken=null, ContentType=WindowsRuntime">
         <HintPath>C:\Program Files (x86)\Windows Kits\10\UnionMetadata\10.0.18362.0\Windows.winmd</HintPath>
     </Reference>
 </ItemGroup>
```

Since the query from above will get us all UWP packages, some of those might not be Xbox games. The solution to this is getting a list of all games the current user owns on Xbox Game Pass which we can get using the Xbox REST API:

```
https://titlehub.xboxlive.com/users/xuid({_xuid})/titles/titlehistory/decoration/details
```

The main problem is getting the `xuid` parameter. This library can accept the `xuid` parameter in the `XboxHandler` constructor and get the title history. Using the title history the library will filter out all installed UWP packages that are not present in the title history.

The following information is on how to get the `xuid` parameter:

Follow [this](https://docs.microsoft.com/en-us/advertising/guides/authentication-oauth-live-connect) guide on how to get started with Live Connect Authentication. After the user logged in with OAuth at `https://login.live.com/oauth20_authorize.srf` you can get the following parameters from the redirection url: `#access_token`, `refresh_token`, `expires_in`, `token_type` and `user_id`.

The access token is needed to authenticate at `https://user.auth.xboxlive.com/user/authenticate` by doing a POST with the following data (remember to use content-type `application/json`):

```json
{
   "RelyingPart": "http://auth.xboxlive.com",
   "TokenType": "JWT",
   "Properties": {
      "AuthMethod": "RPS",
      "SiteName": "user.auth.xboxlive.com",
      "RpsTicket": "<access_token>"
   }
}
```

(DTO: [AuthenticationRequest](GameFinder.StoreHandlers.Xbox/DTO/AuthenticationRequest.cs))

The response is also JSON:

```json
{
   "Token": "the-only-important-field"
}
```

(DTO: [AuthorizationData](GameFinder.StoreHandlers.Xbox/DTO/AuthorizationData.cs))

Now you need to authorize and get the final token. This is another POST request with JSON data to `https://xsts.auth.xboxlive.com/xsts/authorize`:

```json
{
   "RelyingParty": "http://xboxlive.com",
   "TokenType": "JWT",
   "Properties": {
      "SandboxID": "RETAIL",
      "UserTokens": ["<token you got from the previous response>"]
   }
}
```

(DTO: [AuthorizationRequest](GameFinder.StoreHandlers.Xbox/DTO/AuthorizationRequest.cs))

The response is of course also in JSON:

```json
{
   "DisplayClaims": {
      "xui": [
         {
            "uhs": "",
            "usr": "",
            "utr": "",
            "prv": "",
            "xid": "THIS IS THE XUID TOKEN",
            "gtg": ""
         }
      ]
   }
}
```

(DTO: [AuthorizationData](GameFinder.StoreHandlers.Xbox/DTO/AuthorizationData.cs))

After all this requesting you finally have the xuid you can use in the constructor.

### Finding Origin Games

_TODO_

## Contributing

See [CONTRIBUTING](CONTRIBUTING.md) for more information.

## License

GPLv3.0, see [LICENSE](LICENSE) for more information.
