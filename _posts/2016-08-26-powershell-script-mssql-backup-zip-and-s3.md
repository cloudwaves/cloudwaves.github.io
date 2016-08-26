---
layout: post
title: "PowerShell Script - MSSQL Backup to zip and upload to AWS S3 "
date:   2016-08-26 16:44:27
categories: [powershell, mssql, automation, aws, s3]
tags: [powershell, mssql, backup, automation, s3, aws]
author: cloudwaves
comments: true
---

PowerShell (including Windows PowerShell and PowerShell Core) is a task automation <!--more-->and configuration management framework from Microsoft, consisting of a command-line shell and associated scripting language built on the .NET Framework.

I was googling to find a script to backup my MSSQL database and zip the file and upload the content to S3 folder, but unfortunately I could not able to find. So I made myself and sharing here.

Requrinment:
    1.  [7zip][7zip]
    2.  [AWS Tools for Windows PowerShell][awspstools]

## PowerShell Script

{% highlight powershell %}

####################
#####Cloudwaves#####
####################

#Declare values
#Provide the Database server name
$SQLInstance = 'Database Server'

#Provide the Database name to be exclude from backup
$arrDBsToExclude = @("tempdb","ReportServer")

#Provide Local / Network path to store the backup data file
$BackupFolder = '\\Server1\BackupFolder\'

#Privide S3 bucket name
$s3Bucket = 'S3Bucket/DBBACKUPFOLDER'

#Provide AWS REGION 	
$region = 'us-east-1'

#Provide accessKey and secretKey can be removed if running on an EC2 instance and using IAM roles for security
$accessKey = 'Access key'
$secretKey = 'Secret key'

###############DO NOT CHANGE AFTER HERE###############

$timeStamp = Get-Date -format yyyy_MM_dd_HHmmss 
if (-not (test-path "$env:ProgramFiles\7-Zip\7z.exe")) {throw "$env:ProgramFiles\7-Zip\7z.exe needed"} 
set-alias sz "$env:ProgramFiles\7-Zip\7z.exe" 

[System.Reflection.Assembly]::LoadWithPartialName("Microsoft.SqlServer.SMO") | Out-Null
[System.Reflection.Assembly]::LoadWithPartialName("Microsoft.SqlServer.SmoExtended") | Out-Null
 
$srv = New-Object ("Microsoft.SqlServer.Management.Smo.Server") $SQLInstance
$dbs = New-Object Microsoft.SqlServer.Management.Smo.Database 
$dbs = $srv.Databases 

foreach ($dbname in $dbs) 
{ 
	If ($arrDBsToExclude -notcontains $dbname.name)
	{
		$name = $dbname -as [string]
		$name = $name.trim("[]") 
		$GetFile = @(Get-ChildItem $BackupFolder -Filter "$name*.bak") 
		$GetFile = $GetFile -as [string] 
 		if ($GetFile)
		{
			$Result = Test-Path  $GetFile
		}	
		if($Result -eq $false)
		{
			$bakfile ="$name" + "_" + "$timeStamp" + ".trn"
			Invoke-Sqlcmd -SuppressProviderContextWarning -Query "BACKUP LOG $name TO DISK=N'$BackupFolder$bakfile' WITH INIT" -querytimeout 0;
			$zipfile = $bakfile.Replace(".trn",".7z") 
			sz a -t7z "$BackupFolder$zipfile" "$BackupFolder$bakfile" 
			Write-S3Object -BucketName $s3Bucket -File $BackupFolder$zipfile -Key $zipfile -Region $region -AccessKey $accessKey -SecretKey $secretKey
			$Deletezipfile = $BackupFolder + $zipfile
			$Deletebakfile = $BackupFolder + $bakfile
			Remove-Item  -Path filesystem::$Deletezipfile 
			Remove-Item  -Path filesystem::$Deletebakfile
		}
		Else 
		{
			$bk = New-Object ("Microsoft.SqlServer.Management.Smo.Backup") 
			$bk.Action = [Microsoft.SqlServer.Management.Smo.BackupActionType]::Database 
			$bk.BackupSetName = $dbname.Name + "_backup_" + $timeStamp
			$bk.Database = $dbname.Name 
			$bk.CompressionOption = 1 
 			$bk.MediaDescription = "Disk"
   			$bk.Devices.AddDevice($BackupFolder + $dbname.Name + "_" + $timeStamp + ".bak", "File")

   		TRY 
		{
 			$bk.SqlBackup($srv)
 			$bkfilename = ($dbname.Name + "_" + $timeStamp + ".bak")
			Write-Host $bkfilename 
			$zipfile = $bkfilename.Replace(".bak",".7z") 
			sz a -t7z "$BackupFolder$zipfile" "$BackupFolder$bkfilename" 
			Write-S3Object -BucketName $s3Bucket -File $BackupFolder$zipfile -Key $zipfile -Region $region -AccessKey $accessKey -SecretKey $secretKey
			Remove-Item $BackupFolder$zipfile
   		} 

   		CATCH 
   		{
     			$dbname.Name + " backup failed."
     			$_.Exception.Message
   		} 

  		}
	}
}

{% endhighlight %}

[7zip]: http://www.7-zip.org/download.html
[awspstools]: https://aws.amazon.com/powershell/
