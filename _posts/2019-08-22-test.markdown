---
layout: post
title:  "Skipping .dacpac generation in Sitecore Azure Toolkit"
date:   2019-08-21 12:00:00 +0200
categories: sitecore azure
---

TLDR: Remove `ConnectionStrings.config` from the web directory you are packaging. Be sure that the `dataFolder` is available during build.

The Sitecore Azure Toolkit provides a handy method to create `.wdp` packages for deployments. A call to `Start-SitecoreAzurePackaging` might look something like this:

```powershell
Start-SitecoreAzurePackaging `
  -sitecorePath "C:\output\habitat" `
  -destinationFolderPath "C:\output" `
  -cargoPayloadFolderPath "C:\Sitecore Azure Toolkit\resources\9.2.0\CargoPayloads\Sitecore.Cloud.Common.sccpl" `
  -commonConfigPath "C:\Sitecore Azure Toolkit\resources\9.2.0\Configs\Common.Packaging.config.json" `
  -skuConfigPath "C:\Sitecore Azure Toolkit\resources\9.2.0\Configs\XPSingle.Packaging.config.json" `
  -parameterXmlPath "C:\Sitecore Azure Toolkit\resources\9.2.0\MsDeployXmls\XPSingle.Parameters.xml" `
  -integratedSecurity $false
```

During my first attempt to run the command above, the script failed with the following errors:

```powershell
Connecting to database '<...>' on server '<...>.windows.net,1433'.
Extracting schema
Extracting schema from database
*** A network-related or instance-specific error occurred while establishing a connection to SQL Server. The server was not found or was not accessible. Verify that the instance name is correct and that SQL Server is configured to allow remote connections. (provider: TCP Provider, error: 0 - No such host is known.)
*** A network-related or instance-specific error occurred while establishing a connection to SQL Server. The server was not found or was not accessible. Verify that the instance name is correct and that SQL Server is configured to allow remote connections. (provider: TCP Provider, error: 0 - No such host is known.)
*** A network-related or instance-specific error occurred while establishing a connection to SQL Server. The server was not found or was not accessible. Verify that the instance name is correct and that SQL Server is configured to allow remote connections. (provider: TCP Provider, error: 0 - No such host is known.)
*** A network-related or instance-specific error occurred while establishing a connection to SQL Server. The server was not found or was not accessible. Verify that the instance name is correct and that SQL Server is configured to allow remote connections. (provider: TCP Provider, error: 0 - No such host is known.)
*** A network-related or instance-specific error occurred while establishing a connection to SQL Server. The server was not found or was not accessible. Verify that the instance name is correct and that SQL Server is configured to allow remote connections. (provider: TCP Provider, error: 0 - No such host is known.)
*** A network-related or instance-specific error occurred while establishing a connection to SQL Server. The server was not found or was not accessible. Verify that the instance name is correct and that SQL Server is configured to allow remote connections. (provider: TCP Provider, error: 0 - No such host is known.)
```

My goal is to have the Azure Toolkit included in our DevOps Pipeline in order to generate the needed packages for the web root deployment. Obviously, the .dacpac files it generates are not needed and it's also none of the build agent's business to get schemas from the databases.

Having a quick look into `Sitecore.Cloud.Cmdlets.dll` revealed an invocation of the following method: `Sitecore.Cloud.Packaging.WebDeployPackages.WebDeployPackageBuilder.Build( Sitecore.Cloud.Packaging.SitecoreSources.SitecoreInstallationFolderTree scInstFoldTree, DotNet.Basics.IO.DirPath outputDir,System.String targetFileName, System.Boolean force, System.String version, System.Boolean integratedSecurity)`. The method has the following check in place which triggers `DatabasePackager`.

```csharp
string uri = ((IEnumerable<string>) scInstFoldTree.Website).FirstOrDefault<string>((Func<string, bool>) (i => i.ToFile(Array.Empty<string>()).Name == "ConnectionStrings.config"));
if (uri != null)
{
  new DatabasePackager(this._logger, (IDatabasePackagerCore) new DatabasePackagerCore((ILogger) null), (IFileSystemProvider) new FileSystemProvider(), (ISqlServerProvider) new SqlServerProvider(this._logger), (IZipPackageSource) source).PackageDatabases(XDocument.Load(uri), integratedSecurity);
  FilePath file2 = ((IEnumerable<string>) scInstFoldTree.Website).FirstOrDefault<string>((Func<string, bool>) (i => i.ToFile(Array.Empty<string>()).Name == "ConnectionStrings.config")).ToFile(Array.Empty<string>());
  DirPath dir = (scInstFoldTree.WebsitePath + string.Format("\\tempConnectionString{0}\\", (object) Guid.NewGuid())).ToDir(Array.Empty<string>());
  file2.CopyTo(dir, false, true);
  string str = string.Format("{0}\\ConnectionStrings.config", (object) dir);
  XElement xelement1 = XElement.Load(str);
  foreach (XElement xelement2 in xelement1.Elements((XName) "add").Select<XElement, XElement>((Func<XElement, XElement>) (el => el)))
    xelement2.Attribute((XName) "connectionString").Value = "";
  xelement1.Save(str);
  source.UpdateEntryEnsureDirectoriesExists(source.WebsitePath + "/App_Config/ConnectionStrings.config", File.ReadAllBytes(str));
  Directory.Delete(dir.ToString(), true);
}
```

For the next run i've removed the `ConnectionStrings.config` from the web root (`C:\output\habitat` in the example above). This resulted in the script proceeding for a bit longer, but after all failing with a new error:

```powershell
System.IO.DirectoryNotFoundException: Could not find a part of the path 'C:\output\habitat\data'.
   at System.IO.__Error.WinIOError(Int32 errorCode, String maybeFullPath)
   at System.IO.FileSystemEnumerableIterator`1.CommonInit()
   at System.IO.FileSystemEnumerableIterator`1..ctor(String path, String originalUserPath, String searchPattern, SearchOption searchOption, SearchResultHandler`1 resultHandler, Boolean checkHost)
   at System.IO.Directory.GetFiles(String path, String searchPattern, SearchOption searchOption)
   at Sitecore.Cloud.Packaging.WebDeployPackages.WebDeployPackageBuilder.Build(SitecoreInstallationFolderTree scInstFoldTree, DirPath outputDir, String targetFileName, Boolean force, String version, Boolean integratedSecurity)
   at Sitecore.Cloud.Cmdlets.Packaging.NewSCWebDeployPackage.ProcessRecord()
```

Turns out there is a lookup going on in the same method, which tries to determine the `dataFolder`:

```csharp
string path1 = ((IEnumerable<string>) scInstFoldTree.Website).FirstOrDefault<string>((Func<string, bool>) (i => i.ToFile(Array.Empty<string>()).Name == "DataFolder.config"));
string folderPhysicalPath = WebDeployPackageBuilder.GetDataFolderPhysicalPath(path1 == null ? this.ReadDataFolderFromSitecoreConfig(((IEnumerable<string>) scInstFoldTree.Website).First<string>((Func<string, bool>) (i => i.ToFile(Array.Empty<string>()).Name == "Sitecore.config")).ToFile(Array.Empty<string>()).ToString()) : this.ReadDataFolderFromDataFolderConfig(path1.ToFile(Array.Empty<string>()).FullName), scInstFoldTree.WebsitePath);
```

So in order to get the script to work, the defined folder (which it will try to find in either `DataFolder.config` or `Sitecore.config`) has to be available during build. For testing purposes, I just created a `Data` folder in the web root.

After that, the script finished successfully and generated a shiny .wdp file we can now promote to Azure App Services.

There were some warnings at the end of the script, but I assume they can be ignored :)

```powershell
VERBOSE: Performing the operation "Copy File" on target "Item: C:\output\habitat.scwdp.zip Destination: C:\output\habitat_single.scwdp.zip".
WARNING: Parameter Core Admin Connection String refers to missing file Sitecore.Core.dacpac.
WARNING: Parameter Core Admin Connection String refers to missing file CreateUser.Core.sql.
WARNING: Parameter Core Admin Connection String refers to missing file SetSitecoreAdminPassword.sql.
WARNING: Parameter Security Admin Connection String refers to missing file CreateUser.Security.sql.
WARNING: Parameter Master Admin Connection String refers to missing file Sitecore.Master.dacpac.
WARNING: Parameter Master Admin Connection String refers to missing file CreateUser.Master.sql.
WARNING: Parameter Web Admin Connection String refers to missing file Sitecore.Web.dacpac.
WARNING: Parameter Web Admin Connection String refers to missing file CreateUser.Web.sql.
WARNING: Parameter Experience Forms Admin Connection String refers to missing file Sitecore.Experienceforms.dacpac.
WARNING: Parameter Experience Forms Admin Connection String refers to missing file CreateUser.ExperienceForms.sql.
WARNING: Parameter EXM Master Admin Connection String refers to missing file Sitecore.EXM.Master.dacpac.
WARNING: Parameter EXM Master Admin Connection String refers to missing file CreateUser.ExmMaster.sql.
WARNING: Parameter XDB Processing Tasks Admin Connection String refers to missing file Sitecore.Processing.Tasks.dacpac.
WARNING: Parameter XDB Processing Tasks Admin Connection String refers to missing file CreateUser.ProcessingTasks.sql.
```