---
layout: post
title:  "Skipping .dacpac generation in Sitecore Azure Toolkit"
date:   2019-08-21 12:00:00 +0200
categories: sitecore azure
---

Within the `Sitecore.Cloud.Packaging.WebDeployPackages.WebDeployPackageBuilder.Build` method

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
