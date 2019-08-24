---
layout: post
title:  "The hidden cmdlet: Removing Database Operations from .scwdp packages created with Sitecore Azure Toolkit"
date:   2019-08-24 12:00:00 +0200
categories: sitecore azure
---

In my [previous post]({{ site.baseurl }}{% post_url 2019-08-21-skip-dacpac-generation-sitecore-azure-toolkit %}) I've explained a way to skip `.dacpac` creation when running the `Start-SitecoreAzurePackaging` command. While this removes the Database lookups during package creation, we might still be left with a rather sub-optimal `.scwdp` package when it comes to deployment.

It turns out that packages still contain `.sql` files, such as `CreateUser.Core.sql`, `CreateUser.Master.sql`, `SetSitecoreAdminPassword.sql`, etc. which are being invoked during installation of the web deploy package. Let's have a quick look at the `archive.xml` within the generated package for a content management role:

```xml
<sitemanifest MSDeploy.ObjectResolver.createApp="Microsoft.Web.Deployment.CreateApplicationObjectResolver" MSDeploy.ObjectResolver.dirPath="Microsoft.Web.Deployment.DirPathObjectResolver" MSDeploy.ObjectResolver.filePath="Microsoft.Web.Deployment.FilePathObjectResolver">
  <dbFullSql path="CreateUser.Core.sql" MSDeploy.MSDeployLinkName="Child1" MSDeploy.MSDeployKeyAttributeName="path" MSDeploy.MSDeployProviderOptions="XXX">
    <sqlScript size="3gMAAAAAAAA=" MSDeploy.size.Type="Microsoft.Web.Deployment.DeploymentObjectInt64AttributeValue" lastWriteTime="GaqtG4Io10g=" MSDeploy.lastWriteTime.Type="Microsoft.Web.Deployment.DeploymentObjectDateTimeAttributeValue" MSDeploy.MSDeployObjectFlags="1" MSDeploy.MSDeployStreamRelativeFilePath="CreateUser.Core.sql" />
  </dbFullSql>
  <dbFullSql path="SetSitecoreAdminPassword.sql" MSDeploy.MSDeployLinkName="Child2" MSDeploy.MSDeployKeyAttributeName="path" MSDeploy.MSDeployProviderOptions="XXX">
    <sqlScript size="JwUAAAAAAAA=" MSDeploy.size.Type="Microsoft.Web.Deployment.DeploymentObjectInt64AttributeValue" lastWriteTime="GaqtG4Io10g=" MSDeploy.lastWriteTime.Type="Microsoft.Web.Deployment.DeploymentObjectDateTimeAttributeValue" MSDeploy.MSDeployObjectFlags="1" MSDeploy.MSDeployStreamRelativeFilePath="SetSitecoreAdminPassword.sql" />
  </dbFullSql>
  <snip />
  <iisApp path="WebSite" MSDeploy.path="2" MSDeploy.MSDeployLinkName="Child8" MSDeploy.MSDeployKeyAttributeName="path" MSDeploy.MSDeployProviderOptions="XXXX">
    <createApp path="WebSite" MSDeploy.path="2" isDest="AA==" MSDeploy.isDest.Type="Microsoft.Web.Deployment.DeploymentObjectBooleanAttributeValue" managedRuntimeVersion="" MSDeploy.managedRuntimeVersion="2" enable32BitAppOnWin64="" MSDeploy.enable32BitAppOnWin64="2" managedPipelineMode="" MSDeploy.managedPipelineMode="2" applicationPool="" MSDeploy.applicationPool="1" appExists="True" MSDeploy.appExists="1" MSDeploy.MSDeployLinkName="createApp" MSDeploy.MSDeployKeyAttributeName="path" />
    <contentPath path="WebSite" MSDeploy.path="2" MSDeploy.MSDeployLinkName="contentPath" MSDeploy.MSDeployKeyAttributeName="path" MSDeploy.MSDeployProviderOptions="XXX">
      <MSDeploy.dirPath path="WebSite" MSDeploy.MSDeployLinkName="contentPath" />
    </contentPath>
  </iisApp>
</sitemanifest>
```

I assume that those invocations are somehow derived from the `MsDeployXmls`. I didn't really like the idea of manipulating those parameter files, as I'm sure that this will only cause problems down the road when a Sitecore upgraded is needed, so I searched for alternative solution.

Back in `Sitecore.Cloud.Cmdlets.dll`, I found another interesting `Cmdlet`, called `RemoveSCDatabaseOperations` invoking `Sitecore.Cloud.Packaging.WebDeployPackages.WebDeployPackageBuilder.RemoveDBOperations( WebDeployPackageTree, FilePath)`. Taking a closer look at the method, it seems to be doing exactly what I was looking for:

```csharp
public void RemoveDBOperations(WebDeployPackageTree wdpTree, FilePath targetFile)
{
  if (wdpTree == null)
    throw new ArgumentNullException(nameof (wdpTree));
  XDocument xdocument;
  using (MemoryStream memoryStream = new MemoryStream())
  {
    wdpTree.GetEntry("parameters.xml").Extract((Stream) memoryStream);
    memoryStream.Seek(0L, SeekOrigin.Begin);
    xdocument = XDocument.Load((Stream) memoryStream);
  }
  xdocument.Descendants((XName) "parameterEntry").Where<XElement>((Func<XElement, bool>) (x =>
  {
    if (x.Attribute((XName) "scope") == null || !(x.Attribute((XName) "scope").Value.ToLower() == "dbfullsql"))
      return x.Attribute((XName) "scope").Value.ToLower() == "dbdacfx";
    return true;
  })).Remove<XElement>();
  xdocument.Descendants((XName) "parameterEntry").Where<XElement>((Func<XElement, bool>) (x => x.Attribute((XName) "scope").Value.ToLower().EndsWith(".sql"))).Remove<XElement>();
  try
  {
    wdpTree.RemoveEntries((ICollection<Ionic.Zip.ZipEntry>) wdpTree.RootSQLFiles.ToList<Ionic.Zip.ZipEntry>());
    wdpTree.RemoveEntries((ICollection<Ionic.Zip.ZipEntry>) wdpTree.Dacpacs.ToList<Ionic.Zip.ZipEntry>());
    wdpTree.Commit();
  }
  catch
  {
    wdpTree.Rollback();
  }
  this.UpdateMSDeployXmls(targetFile.FullName, xdocument.ToString());
}
```

Unfortunately, `Sitecore.Cloud.Cmdlets.psm1` doesn't export the cmdlet:

```powershell
Export-ModuleMember -Function Start-SitecoreAzureDeployment
Export-ModuleMember -Function Start-SitecoreAzurePackaging
Export-ModuleMember -Function Start-SitecoreAzureModulePackaging
Export-ModuleMember -Function ConvertTo-SitecoreWebDeployPackage
Export-ModuleMember -Function Set-SitecoreAzureTemplates
Export-ModuleMember -Cmdlet New-SCCargoPayload
```

However, since the class is a proper `PSCmdlet`, we can just call `Import-Module` on the `Sitecore.Cloud.Cmdlets.dll` and then invoke the cmdlet as usual. As of right now, my PowerShell Script in Azure DevOps for building packages looks like this:

```powershell
Import-Module "$(satPsmPath)" -Force
Import-Module "$(satDllPath)" -Force

# Remove ConnectionStrings.config and create a data folder
# see: https://schwindelig.io/sitecore/azure/2019/08/21/skip-dacpac-generation-sitecore-azure-toolkit.html
Remove-Item "$(buildOutTempPath)/App_Config/ConnectionStrings.config" -ErrorAction Ignore
New-Item -Path "$(buildOutTempPath)" -Name "data" -ItemType "directory" -ErrorAction Ignore

# Invoke Sitecore Azure Toolkit magic for creating web deploy packages
Start-SitecoreAzurePackaging `
  -sitecorePath "$(satSitecorePath)" `
  -destinationFolderPath "$(satDestinationFolderPath)" `
  -cargoPayloadFolderPath "$(satCargoPayloadFolderPath)" `
  -commonConfigPath "$(satCommonConfigPath)" `
  -skuConfigPath "$(satSkuConfigPath)" `
  -parameterXmlPath "$(satParameterXmlPath)" `
  -integratedSecurity $false `
  -Verbose

# Remove all DB operations from packages
Get-ChildItem "$(satDestinationFolderPath)" -Filter *.scwdp.zip |
ForEach-Object {
  Remove-SCDatabaseOperations -Path $_.FullName -Destination "$(satDestinationFolderPath)" -Force -Verbose
  Remove-Item $_.FullName -Force -ErrorAction Ignore
}
```
