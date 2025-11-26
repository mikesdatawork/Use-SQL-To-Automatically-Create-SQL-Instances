> **Note** — The folder `linguist-samples/` contains tiny real files so GitHub can correctly display all languages used in this repo.  
> The actual content and examples remain in this README.

![MIKES DATA WORK GIT REPO](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_01.png "Mikes Data Work")        

# Use SQL To Automatically Create SQL Instances
**Post Date: November 16, 2016**        



## Contents    
- [About Process](##About-Process)  
- [SQL Logic](#SQL-Logic)  
- [Build Info](#Build-Info)  
- [Author](#Author)  
- [License](#License)       

## About-Process

<p>The following logic will produce an SQL Server 2014 unattended installation configs file (.ini). Then it will execute the installation using the config file. 

Remember to change the following lines accordingly.
1. Add your Domain\ServiceAccounts
2. Add your passwords.
3. Add your Domain\AdminGroup
4. Add your drive\paths
5. Add the path where your SQL Server 2014 install medium resides
6. Add the SQL Services, or Features you want installed
7. Add the prefix name you want to use as your instance name.

Note:
Presently the SQL Server Named Instances has a convention of: SQLSHARE01, …02, …03, etc.
The only 2 services included with this installation is the SQL Server Service, and the SQL Server Reporting Service</p>      


## SQL-Logic
```SQL
use master;
set nocount on
 
declare @server_name    varchar(255) = (select @@servername)
declare @run_bcp    varchar(8000) 
declare @silent_string  varchar(max)
 
-- create table to hold instance names
declare @sql_instances table
(
    [id]            int identity(0,1)
,   [rootkey]       nvarchar(255)
,   [sql_instances] nvarchar(255)
,   [value]     nvarchar(255)
)
  
-- get instances names using xp_regread
insert into @sql_instances ([rootkey], [sql_instances], [value])
execute xp_regread
    @rootkey        = 'hkey_local_machine'
,   @key            = 'software\microsoft\microsoft sql server'
,   @value_name     = 'installedinstances'
 
delete  from @sql_instances where [sql_instances] = 'SQL_2016_01'
-- set prefix name for multi-instance environment aka: Shared SQL Environment "SQLSHARE"
-- produce the next instance name.
declare @prefix     varchar(255) = 'SQLSHARE'
declare @next_instance  varchar(255) = 
(
select
    case
        when max([sql_instances]) is null               then 'SQLSHARE01'
        when max([sql_instances]) = 'MSSQLSERVER'           then 'SQLSHARE01'
        else @prefix + cast(cast(right(max([sql_instances]), 2) as int) + 1 as varchar)
    end as [sql_instances]
from
    @sql_instances si
where
    si.[id] > 0
)
-- reference: https://msdn.microsoft.com/en-us/library/ms144259(v=sql.120).aspx
 
set @silent_string      =
'[OPTIONS]
ACTION              ="Install"
ENU             ="True"
QUIET               ="True"
QUIETSIMPLE         ="False"
UpdateEnabled           ="False"
ERRORREPORTING          ="False"
USEMICROSOFTUPDATE      ="False"
FEATURES            =SQLENGINE,RS
UpdateSource            ="MU"
HELP                ="False"
INDICATEPROGRESS        ="False"
X86             ="False"
INSTALLSHAREDDIR        ="E:\Program Files\Microsoft SQL Server"
INSTALLSHAREDWOWDIR     ="E:\Program Files (x86)\Microsoft SQL Server"
INSTANCENAME            ="' + @next_instance + '"
SQMREPORTING            ="False"
INSTANCEID          ="' + @next_instance + '"
RSINSTALLMODE           ="FilesOnlyMode"
INSTANCEDIR         ="E:\Program Files\Microsoft SQL Server"
AGTSVCACCOUNT           ="MyDomain\MyServiceAccount"
AGTSVCPASSWORD          ="MyServiceAccountPassword"
AGTSVCSTARTUPTYPE       ="Automatic"
COMMFABRICPORT          ="0"
COMMFABRICNETWORKLEVEL          ="0"
COMMFABRICENCRYPTION        ="0"
MATRIXCMBRICKCOMMPORT           ="0"
SQLSVCSTARTUPTYPE       ="Automatic"
FILESTREAMLEVEL         ="0"
ENABLERANU          ="False"
SQLCOLLATION            ="SQL_Latin1_General_CP1_CI_AS"
SQLSVCACCOUNT           ="MyDomain\MyServiceAccount"
SQLSVCPASSWORD          ="MyServiceAccountPassword"
SQLSYSADMINACCOUNTS     ="MyDomain\MyServiceAccount" "MyDomain\MyAdminGroup"
SECURITYMODE            ="SQL"
SAPWD               ="MyPassword' + right(@next_instance, 2) + '!!##"
SQLUSERDBDIR            ="E:\Program Files\Microsoft SQL Server\MSSQL12.' + @next_instance + '\MSSQL\Data"
SQLUSERDBLOGDIR         ="E:\Program Files\Microsoft SQL Server\MSSQL12.' + @next_instance + '\MSSQL\Data"
SQLBACKUPDIR            ="E:\Program Files\Microsoft SQL Server\MSSQL12.' + @next_instance + '\MSSQL\Backup"
ADDCURRENTUSERASSQLADMIN    ="False"
TCPENABLED          ="1"  
NPENABLED           ="0"
BROWSERSVCSTARTUPTYPE       ="Automatic"
RSSVCACCOUNT            ="MyDomain\SQLServiceAdmin"
RSSVCPASSWORD           ="MyServiceAccountPassword"
RSSVCSTARTUPTYPE        ="Automatic"'
 
if object_id('tempdb..##silent_string') is not null drop table ##silent_string
create table ##silent_string ([bcpcommand]  varchar(8000)) insert into  ##silent_string select @silent_string
 
-- create sql config file
select  @run_bcp = 
'bcp "select [bcpcommand] from ##silent_string" ' + 'queryout "E:\sql_silent_install_config_file\unattended_install_SQL_2014_STANDARD.ini" -c -t -T -S' + @server_name 
exec master..xp_cmdshell @run_bcp
 
-- install
exec master..xp_cmdshell 'e:\sql_server_2014\standard_x64\setup.exe /co

nfigurationfile=E:\sql_silent_install_config_file\unattended_install_SQL_2014_STANDARD.ini /IacceptSQLServerLicenseTerms'
```
Edit…
Here's a slight improvement on the case statement above.



## SQL-Logic
```SQL
<strong>case
case
    when max([sql_instances]) is null       then 'SQLSHARE01'
    when max([sql_instances]) = 'MSSQLSERVER'   then 'SQLSHARE01'
    else @prefix + cast(cast(right(max([sql_instances]), 2) as int) + 1 as varchar)
end as [sql_instances]
</strong>
```

To verify the instances were created, and get the port numbers; you can run the following:



## SQL-Logic
```SQL
use master;
set nocount on
   
declare @sql_instances  table
(   
    [rootkey]   varchar(255)
,   [value]     varchar(255)
)
insert into @sql_instances 
exec master.dbo.xp_instance_regenumvalues 
    @rootkey    = N'HKEY_LOCAL_MACHINE'
,   @key        = N'SOFTWARE\Microsoft\Microsoft SQL Server\Instance Names\SQL';
   
declare db_cursor   cursor for select upper([rootkey]), upper([value]) from @sql_instances
declare @instance_name  varchar(255)
declare @instance_path  varchar(255)
open    db_cursor;
fetch next from db_cursor into @instance_name, @instance_path
    while @@fetch_status = 0  
        begin
            declare @port_table table
            (
                [Instance]  varchar(255)
            ,   [Port]      int
            )
            declare @port   varchar(50)
            declare @key    varchar(255) = 'software\microsoft\microsoft sql server\' + @instance_path + '\mssqlserver\supersocketnetlib	cp\ipall'
            exec master..xp_regread
                @rootkey    = 'hkey_local_machine'
            ,   @key        = @key
            ,   @value_name = 'tcpdynamicports'
            ,   @value      = @port output
              
            insert into @port_table
            select
                'Instance'  = @instance_name
            ,   'Port'  = isnull(convert(varchar(10), @port), 1433)
            fetch next from db_cursor into @instance_name, @instance_path
        end;
    close db_cursor
deallocate db_cursor;
 
select * from @port_table order by [instance] desc
```


[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

[![Gist](https://img.shields.io/badge/Gist-MikesDataWork-<COLOR>.svg)](https://gist.github.com/mikesdatawork)
[![Twitter](https://img.shields.io/badge/Twitter-MikesDataWork-<COLOR>.svg)](https://twitter.com/mikesdatawork)
[![Wordpress](https://img.shields.io/badge/Wordpress-MikesDataWork-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

    
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Mikes Data Work](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_02.png "Mikes Data Work")
