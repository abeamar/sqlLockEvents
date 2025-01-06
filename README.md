
# Block and Deadlock monitor <img src="https://cdn-dynmedia-1.microsoft.com/is/image/microsoftcorp/UHFbanner-MSlogo?fmt=png-alpha&bfc=off&qlt=100,1" align="right" width="150">
Showcase of my way to monitor blocking and deadlock sessions occurrence history with all the information needed to be analysed, using views and extended events. Created for MS SQL.  
<hr>
    </ul>
    <p dir="auto">
        <a href="#about">1. About</a><br>
        <a href="#blocking-event">2. Blocking Event</a><br>
        <a href="#deadlock-event">3. Deadlock Event</a><br>
        <a href="#create-views">4. Create Views</a><br>
        <a href="#conclusion">5. Conclusion</a><br>
    </p>
    <hr>
    <br>
     <section id="about">
        <h2>1. About</h2>
        <p>
	I wanted to have accurate information about locks on my databases. I did that using the right extended events and then created a custom SELECT query to extract the right information from the created file and present it in my reports.<br><br>&emsp; - Key columns for <b>deadlock sessions</b> are deadlock victim, deadlock object, sql_text and the users in process. All with clear overview.<br>&emsp; - Key columns for <b>blocking sessions</b> are blocking start and end time, duration, sql text and users in process. It's all displayed in ONE line, so it gives you a clear overview.
	<br>Below is an overview of the information you get at the end. 
	</p>
    </section>
        <br>
            <hr>
    <section id="blocking-event">
        <h2>2. Blocking Event</h2>
        <p>First we need to enable blocked process treshold and then create Extended Event. Treshold min value is 5 sec to check for blocks.</p>
 <pre><code>
exec sp_configure 'show advanced options',1;
GO
RECONFIGURE;
GO
exec sp_configure 'blocked process threshold (s)',5;
GO
RECONFIGURE;
GO
</code></pre>
    
  <pre><code>
CREATE EVENT SESSION blckCapture
ON SERVER
ADD EVENT sqlserver.blocked_process_report(
    ACTION (
        sqlserver.sql_text,
        sqlserver.session_id,
        sqlserver.username,
        sqlserver.client_hostname
    ))
ADD TARGET package0.event_file(SET filename=N'C:\blckSessions.xel',max_file_size=(100),max_rollover_files=(5))
WITH (MAX_MEMORY=4096 kB,EVENT_RETENTION_MODE=ALLOW_SINGLE_EVENT_LOSS,MAX_DISPATCH_LATENCY=36000 SECONDS,MAX_EVENT_SIZE=0 KB,MEMORY_PARTITION_MODE=NONE,TRACK_CAUSALITY=OFF,STARTUP_STATE=ON)
GO
ALTER EVENT SESSION blckCapture ON SERVER STATE = START;
ALTER EVENT SESSION blckCapture ON SERVER WITH (STARTUP_STATE=ON);
GO
</code></pre>
</div> 
    </section>
        <br>
            <hr>
    <section id="deadlock-event">
        <h2>3. Deadlock Event</h2>
        <p>Create one more Extended Event for deadlocks, with target as a filename.</p>

  <pre><code>
CREATE EVENT SESSION deadlckCapture
ON SERVER
ADD EVENT sqlserver.xml_deadlock_report
ADD TARGET package0.event_file(SET filename=N'C:\dlckSessions.xel',max_file_size=(10),max_rollover_files=(5))
WITH (MAX_MEMORY=4096 KB,EVENT_RETENTION_MODE=ALLOW_SINGLE_EVENT_LOSS,MAX_DISPATCH_LATENCY=30 SECONDS,MAX_EVENT_SIZE=0 KB,MEMORY_PARTITION_MODE=NONE,TRACK_CAUSALITY=OFF,STARTUP_STATE=ON)
GO
ALTER EVENT SESSION deadlckCapture ON SERVER STATE = START;
ALTER EVENT SESSION deadlckCapture ON SERVER WITH (STARTUP_STATE=ON);
GO
</code></pre>
</div>   
    </section>
        <br>
            <hr>
    <section id="create-views">
        <h2>4. Create Views</h2>
        <p>Last step is to create Views in database of your choice. And after that you can create your reports.<br></p>
 <pre><code>
CREATE VIEW vw_blckSessions AS
;WITH blckData AS (
    SELECT
        DATEADD(HOUR, 1, event_data.value('(event/@timestamp)[1]', 'DATETIME')) AS EventTime,
        blocked_process.value('@spid', 'INT') AS BlockedSPID,
        blocking_process.value('@spid', 'INT') AS BlockingSPID,
        blocked_process.value('@hostname', 'NVARCHAR(256)') AS BlockedHostname,
        blocked_process.value('@loginname', 'NVARCHAR(256)') AS BlockedLoginName,
        blocked_process.value('(inputbuf)[1]', 'NVARCHAR(MAX)') AS BlockedSQLText,
        blocking_process.value('@hostname', 'NVARCHAR(256)') AS BlockingHostname,
        blocking_process.value('@loginname', 'NVARCHAR(256)') AS BlockingLoginName,
        blocking_process.value('(inputbuf)[1]', 'NVARCHAR(MAX)') AS BlockingSQLText
    FROM (
        SELECT CAST(event_data AS XML) AS event_data
        FROM sys.fn_xe_file_target_read_file('C:\UPDATE\blckSessions*.xel', NULL, NULL, NULL)) AS Data
    	CROSS APPLY event_data.nodes('//event[@name="blocked_process_report"]/data[@name="blocked_process"]/value/blocked-process-report') AS XEventData (blocked_report)
    	CROSS APPLY XEventData.blocked_report.nodes('blocked-process/process') AS BlockedProcessNode (blocked_process)
   	CROSS APPLY XEventData.blocked_report.nodes('blocking-process/process') AS BlockingProcessNode (blocking_process))
	,blckData2 AS (SELECT
                    CONVERT(VARCHAR(19), MIN(EventTime), 120) AS Eventime_start,
                    CONVERT(VARCHAR(19), MAX(EventTime), 120) AS Eventime_last,
                    DATEDIFF(SECOND, MAX(EventTime), MIN(EventTime)) as Duration,
                    BlockingSPID,
   		    BlockingHostname,
  		    BlockingLoginName,
 		    BlockingSQLText,
   		    BlockedSPID,
    		    BlockedHostname,
    		    BlockedLoginName,
    		    BlockedSQLText
		    FROM blckData
			GROUP BY BlockedSPID, BlockedHostname, BlockedLoginName, BlockedSQLText, BlockingSPID, BlockingHostname, BlockingLoginName, BlockingSQLText)
	SELECT
                    Eventime_start
                    ,Eventime_last
                    ,ABS(Duration) AS Duration
                    ,BlockingSPID
                    ,BlockedSPID
    		    ,BlockingHostname
    		    ,BlockingLoginName
    		    ,BlockingSQLText
    		    ,BlockedHostname
    		    ,BlockedLoginName
   		    ,BlockedSQLText
	FROM blckData2 ORDER BY Eventime_last DESC;
</code></pre>       
  <pre><code>
CREATE VIEW vw_dlckSessions AS
    SELECT
    event_data.value('(event/@timestamp)[1]', 'DATETIME') AS DeadlockStartTime,
    deadlock_node.value('@hostname', 'NVARCHAR(256)') AS Hostname,
    deadlock_node.value('@loginname', 'NVARCHAR(256)') AS LoginName,
    deadlock_node.value('@spid', 'INT') AS SPID,
    deadlock_node.value('(inputbuf)[1]', 'NVARCHAR(MAX)') AS SQLText,
    resource_node.value('@objectname', 'NVARCHAR(256)') AS ObjectName,
    CASE
        WHEN deadlock_node.value('@id', 'NVARCHAR(256)') = victim_node.value('@id', 'NVARCHAR(256)')
        THEN 1
        ELSE 0
    END AS Victim,
    CASE
        WHEN deadlock_node.value('@id', 'NVARCHAR(256)') = victim_node.value('@id', 'NVARCHAR(256)')
        THEN 'Yes'
        ELSE 'No'
    END AS Evicted
    FROM (
    SELECT CAST(event_data AS XML) AS event_data
    FROM sys.fn_xe_file_target_read_file('C:\dlckSessions*.xel', NULL, NULL, NULL)) AS Data
CROSS APPLY Data.event_data.nodes('//event[@name="xml_deadlock_report"]/data[@name="xml_report"]/value/deadlock/process-list/process') AS ProcessNode (deadlock_node)
CROSS APPLY Data.event_data.nodes('//event[@name="xml_deadlock_report"]/data[@name="xml_report"]/value/deadlock/resource-list/keylock') AS ResourceNode (resource_node)
CROSS APPLY Data.event_data.nodes('//event[@name="xml_deadlock_report"]/data[@name="xml_report"]/value/deadlock/victim-list/victimProcess') AS VictimNode (victim_node)
ORDER BY DeadlockStartTime DESC;
</code></pre>
    </section>
        <br>
            <hr>
    <section id="conclusion">
        <h2>5. Conclusion</h2>
        <p>With these two views <br>
&emsp;vw_blckSessions <br>
&emsp;vw_dlckSessions <br>
Now you can monitor SPIDs on your sql instance with all the necessary information to debug.</p>
    </section>
        <br>


 <pre><code>






dcomcnfg -- component
Provjera drivera:  C:\WINDOWS\SysWOW64\odbcad32.exe
--RECOVER CLOSED QUERIES SESSION:
--Documents\Visual Studio 2017\Backup Files\Solution1

--msiexec /i "path"     wusa.exe "path"     start   --primejr wusa.exe "windows10.0-kb5043124-x64_1377c8d258cc869680b69ed7dba401b695e4f2ed.msu"  
-- PowerShell.exe -File C:\RESTORE\RESTORE.ps1
-- C:\temp\SW_DVD9_NTRL_SQL_Svr_Ent_Core_2019Dec2019_64Bit_English_OEM_VL_X22-22120>Setup.exe /SkipRules=StandaloneInstall_HasClusteredOrPreparedInstanceCheck /Action=Install
--=LEFT(A1,(FIND(" ",A1,1)-1))     =RIGHT (A1, LEN (A1)-1)
--https://drive.google.com/drive/folders/12OVM2d1zSzEcVaogUP6WC9Kb3R2-oXjQ dobri test sujecti

--adapter.SelectCommand.CommandTimeout = 0;
--c$\Program Files\Microsoft SQL Server\160\Setup Bootstrap\Log\20240704_120410\ConfigurationFile.ini

get-hotfix 
wmic qfe
Cells.EntireColumn.Autofit
sqlservr -m -T4022  -T3659 -s"MSSQLServer" -q"SQL_Latin1_General_CP1_CI_AS"

sp_replmonitorhelppublication
sp_refreshview
sp_removedbreplication 'FON'
pushd popd

select * from distribution.dbo.msrepl_errors order by time desc

RESTORE HEADERONLY  FROM DISK = '\\DATAb\c$\UPDATE\TEST\FILE1.bck'; 
SELECT * FROM fn_virtualservernodes() 


IF OBJECT_ID('dbo.tabela', 'U') IS NOT NULL 
  --DROP TABLE dbo.ImportHolisticApproach; 
  print 'prazno'
/*
EXEC xp_cmdshell 'del \\testcentnew\SQLTEST\GDWH\pisardb_20230407 /S /Q' 
sp_configure 'xp_cmdshell',1
RECONFIGURE
EXEC master.sys.xp_delete_files   '\\testcentnew\SQLTEST\GDWH\pisardb_20230407\'
*/
Klist –li 0x3e7 purge

INSERT INTO OPENROWSET('Microsoft.ACE.OLEDB.12.0','Excel 12.0; Database=C:\UPDATE\tester.xlsx;',
'SELECT * FROM [Sheet1$]') select * from [dbo].[check_db]

SELECT sqlserver_start_time FROM sys.dm_os_sys_info


--smseagle upute
https://www.smseagle.eu/code-samples/



SELECT DATABASEPROPERTYEX(DB_NAME(), 'Collation') AS DatabaseCollation;
SELECT SERVERPROPERTY('Collation') AS ServerCollation;
SELECT @@SERVERNAME





EXEC sp_repldone 
@xactid = NULL, @xact_segno = NULL, @numtrans = 0, @time= 0, @reset = 1 



USE FON;
exec sp_scriptpublicationcustomprocs @publication = 'publ'



--SUBSTRING
SET @STATUS = SUBSTRING(@STATUS, CHARINDEX('bo.xp_create_subdir', @STATUS)+6, LEN(@STATUS))

  UPDATE [dbo].[tabela]
  SET linija_vrijednosti = 
  REPLACE([linija_vrijednosti],[linija_vrijednosti],
  '<Spiskovi><Racun>0'+SUBSTRING([linija_vrijednosti],CHARINDEX('<Spiskovi><Racun>',[linija_vrijednosti])+17,LEN([linija_vrijednosti])))
  where vrijeme_upisa = ''

\package.variables[putanja].Value



DBCC INPUTBUFFER (54)
SELECT
   Req.session_id ,InBuf.event_info 
FROM sys.dm_exec_requests AS Req
JOIN sys.dm_exec_sessions AS Ses 
   ON Ses.session_id = Req.session_id
CROSS APPLY sys.dm_exec_input_buffer(Req.session_id, Req.request_id) AS InBuf
WHERE
    Ses.session_id>239 and Ses.is_user_process = 1
GO

--LOKACIJA FAJLA
EXEC xp_dirtree 'C:\Users\Public\Documents', 0, 1;

SELECT SUBSTRING(@1, CHARINDEX('LOG', @1)+3, LEN(@1)) 

--tnsname:
--C:\app\client\product\19.0.0\client_1\network\admin

SELECT DATEPART(HOUR, GETDATE());
SELECT DATEPART(DAY, GETDATE()) AS Month;

sqlcmd -v ColumnName ="LastName" -i c:\testscript.sql
					





ALTER AUTHORIZATION ON SCHEMA::stage2 TO dbo;
GO 
ALTER SCHEMA dbo TRANSFER S1.PRPA_ACCNT_AC;  
GO  



--DACKPACK BEZ VALIDACIJE
--C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\Common7\IDE\Extensions\Microsoft\SQLDB\DAC\130
--.\sqlpackage /action:Extract /TargetFile:"C:\Temp\T24DataTransformationCopy.dacpac" /SourceConnectionString:"Server=dedo2;Integrated Security=SSPI;Database="T24DataTransformation_copy"


IF OBJECT_ID(N'dbo.customerInfo_test', N'U') IS NOT NULL  
   DROP TABLE [dbo].[customerInfo_test];  



----sp_dropdistributor

--EXEC sp_dropdistributor @no_checks = 1, @ignore_distributor = 1
--GO
--execute sys.sp_MSrepl_getdistributorinfo

--SELECT * FROM sys.databases WHERE name = 'distribution';

--select * from msdb.dbo.MSdistpublishers
--delete from msdb.dbo.MSdistpublishers

--sp_dropdistributiondb 'distribution'

--USE master
--GO
--sp_removedbreplication
--EXEC sp_dropdistributor @no_checks = 1

sp_replrestart 



-- DODAVANJE SUBSCRIPTIONuse [test]

use [test]
exec sp_droparticle @publication = N'DATAPOOL_test', @article = N'Racuni', @force_invalidate_snapshot = 1
GO
-----------------------------------------------------------------------------------------------------------------



@snapshot_in_defaultfolder = N'false', @alt_snapshot_folder = N'c:\SNAPSHOT',


--ALTER USER WITH LOGIN
EXEC sp_change_users_login 'Update_One', 'test_user', 'testalias';  
GO  

-- PROVJERA LOCK DISABLED Account User
-----------------------------------------------------------------------------------------------------------------
SELECT name, is_disabled, LOGINPROPERTY(name, N'isLocked') as is_locked,
LOGINPROPERTY(name, N'LockoutTime') as LockoutTime
FROM sys.sql_logins
-----------------------------------------------------------------------------------------------------------------

--SERVER INFO
-----------------------------------------------------------------------------------------------------------------
SELECT @@VERSION
SELECT SERVERPROPERTY ('MachineName')
SELECT SERVERPROPERTY ('Edition')
SELECT SERVERPROPERTY ('INSTANCEDEFAULTDATAPATH')
SELECT SERVERPROPERTY ('INSTANCEDEFAULTLOGPATH')
SELECT SERVERPROPERTY (' PRODUCTVERSION')
SELECT SERVERPROPERTY ('BUILDCLRVERSION')
SELECT SERVERPROPERTY ('PROCESSID')
SELECT SERVERPROPERTY ('ResourceLastUpdateDateTime')
SELECT SERVERPROPERTY ('EditionID')
select SERVERPROPERTY ('collation')

SELECT host_platform, host_distribution, host_release
FROM sys.dm_os_host_info;
GO
-----------------------------------------------------------------------------------------------------------------




RTRIM(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(UPPER(COALESCE(Prezime,'')), 
	N'Æ','C'), N'č','C'), N'ć','C'),N'Č','C'), N'Ć','C'), N'š','S'), N'Š','S'), N'Đ','DJ'),N'Ð','DJ'), 'Ž','Z'), 'ž','Z')) AS PREZIME,

=LEFT(A1, SEARCH(" ",A1,1))
=LEFT(E2, FIND(" ", E2)-1)

CREATE NONCLUSTERED INDEX [IX_analitika_datum_val_2023] ON [dbo].[analitika]
(
	[datum_val] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, SORT_IN_TEMPDB = OFF, DROP_EXISTING = OFF, ONLINE = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, FILLFACTOR = 90, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF)
GO






DECLARE @seconds AS int = 896434;
SELECT
    CONVERT(varchar, (@seconds / 86400))                --Days
    + ':' +
    CONVERT(varchar, DATEADD(ss, @seconds, 0), 108);    --Hours, Minutes, Seconds
	
	
	
	
	SELECT a.[job_id],[name],[enabled]
 ,[last_run_outcome]
      ,[last_outcome_message]
      ,[last_run_date]
      ,[last_run_time]
      ,[last_run_duration]
     FROM [msdb].[dbo].[sysjobs] a 
     Inner Join [msdb].[dbo].[sysjobservers] b
     On a.job_id = b.job_id
	 where [name] like ('%analitika%')





CREATE FUNCTION [prc_vwalation]()
RETURNS TABLE
AS
RETURN
  SELECT *
  FROM OPENROWSET('SQLNCLI', 'Server=DATAB\PRODUCT;Trusted_Connection=yes;', 'EXEC [appdb].[dbo].[prc_alation]') AS a;




--PROVJERA NAPSHOT
SELECT name
, s.snapshot_isolation_state
, snapshot_isolation_state_desc
, is_read_committed_snapshot_on
FROM sys.databases s





--TIP ZA UMJESTO CASE IIF
SET stage = IIF(partija <= '56','Y','N')
-----------------------------------------------------------------------------------------------------------------

--PROVJERE BOOT, RESTART TIME EVENTVIEW
Check EVENTVIEW for restart : 41, 1074, 6006, 6008
In the left pane of Event Viewer, open Windows Logs and System, right click or press and hold on System, and click/tap on Filter Current Log. (see screenshot below)
systeminfo | find "System Boot Time"



--check sebr broj naloga
select getdate(),count(1) as Count from SEB.seb.c_payment with (nolock) where seb_status = '6'
-----------------------------------------------------------------------------------------------------------------
Kada kreirate database:
EXEC [dba].[sp_createdb] 'Ime database-a'
Kada kreirate novog SQL user-a za bazu (na microsvcdb):
EXEC [dba].[sp_createSQLLogin] 'Ime database-a'
-----------------------------------------------------------------------------------------------------------------



--RDL DATASOURCE:
Data Source=test;Initial Catalog=test
--DATASOURCE SSRS DA IDE NA READONLY REPORTSERVICE
data source="st\head";initial catalog=GRAS;ApplicationIntent=ReadOnly;MultiSubnetFailover=True;
-----------------------------------------------------------------------------------------------------------------






-----------------------------------------------------------------------------------------------------------------
exec master..sp_DefragIndexes 'istorija',100,100,0,'FULLSCAN'
DBCC FREEPROCCACHE;
-----------------------------------------------------------------------------------------------------------------



-----------------------------------------------------------------------------------------------------------------
select name, is_read_committed_snapshot_on from sys.databases
ALTER DATABASE TestDB SET READ_COMMITTED_SNAPSHOT ON 

SELECT name, is_read_committed_snapshot_on
FROM sys.databases
WHERE name = DB_NAME();

dbcc useroptions
ALTER DATABASE [tester] SET READ_COMMITTED_SNAPSHOT Off
-----------------------------------------------------------------------------------------------------------------




--PROVJERA PERMISIJA
SELECT * 
FROM master.sys.database_permissions [dp] 
JOIN master.sys.system_objects [so] ON dp.major_id = so.object_id
JOIN master.sys.sysusers [usr] ON usr.uid = dp.grantee_principal_id AND usr.name = 'amar'
WHERE permission_name = 'EXECUTE' 





--ALTER NAME LOGIC
-----------------------------------------------------------------------------------------------------------------
ALTER DATABASE [ime] MODIFY FILE ( NAME = rev, NEWNAME = KremaOdsDEV );
GO
ALTER DATABASE [me] MODIFY FILE ( NAME = ev_log, NEWNAME = KremaOdsDEV_log );
GO






-- PROMJENA SERVERA
--CHANGE DATASOURCE LINKED SERVER, RENAME LINK
-----------------------------------------------------------------------------------------------------------------
DECLARE @name sysname = 'BOSNADB\head', @datasource sysname = 'uatbosnadb\bos';
EXECUTE sp_setnetname @server = @name, @netname = @datasource;
-----------------------------------------------------------------------------------------------------------------







--PROVJERA ZA LOG PUNJENJE
-----------------------------------------------------------------------------------------------------------------
SELECT database_transaction_log_bytes_reserved,session_id 
  FROM sys.dm_tran_database_transactions AS tdt 
  INNER JOIN sys.dm_tran_session_transactions AS tst 
  ON tdt.transaction_id = tst.transaction_id 
  WHERE database_id = 2;

--PROVJERA LOG KORISTENJA
-----------------------------------------------------------------------------------------------------------------
DBCC SQLPERF(logspace)
DBCC OPENTRAN
dbcc showcontig ('dospjeca',1)



-- PREGLED DANA
-----------------------------------------------------------------------------------------------------------------
DECLARE @dayNumber INT;
SET @dayNumber = DATEPART(DW, GETDATE());


--Sunday = 1, Saturday = 7.
IF(@dayNumber != 1 OR @dayNumber != 7) 
    PRINT 'Weekend';
ELSE
    PRINT 'NOT Weekend';

IF (SELECT DATEPART(DW, GETDATE())) IN (2,3,4,5,6) 
print 'amar'
-----------------------------------------------------------------------------------------------------------------






--KOMANDE ZA GIT
-----------------------------------------------------------------------------------------------------------------
/*
git clone ""
git clone "http://azuredevops:8080/tfs/CBSDBCollection/COREDB/_git/db.test.test"
git add --all
git commit -m "new"
git push
*/
-----------------------------------------------------------------------------------------------------------------




-----------------------------------------------------------------------------------------------------------------
--DETELE PROVJERA
begin transaction
UPDATE [test].T.CA_DM_Bills
SET 
[test].T.CA_DM_Bills.ID_ARR_SEQUENCE=[dba].[dbo].[priprema].ID_ARR_SEQUENCE
FROM [dba].[dbo].[priprema]
INNER JOIN
[test].T.CA_DM_Bills
ON [dba].[dbo].[priprema].INTERNAL_NO = T.CA_DM_Bills.INTERNAL_NO
commit


begin transaction
UPDATE [FON].dbo.Customer
SET 
[FON].dbo.customers.regno=[DATAb].[FON].[dbo].customers.regno
FROM [DATAb].[FON].[dbo].customers.regno
INNER JOIN
[FON].dbo.Customer
ON [DATAb].[FON].[dbo].customer.customerID = FON.dbo.[Customer].customerID
-----------------------------------------------------------------------------------------------------------------





--CHECK TRIGGERS
-----------------------------------------------------------------------------------------------------------------
SELECT  
    name,
    is_instead_of_trigger
FROM 
    sys.triggers  
WHERE 
    type = 'TR';

-- OPCIJA DISABLE TRIGGER NA BAZI GLOBALNO
sp_msforeachtable 'ALTER TABLE ? DISABLE TRIGGER all'
-----------------------------------------------------------------------------------------------------------------








--USEFUL TIPS
-----------------------------------------------------------------------------------------------------------------
sp_spaceused
sp_help_revlogin
sp_helpdb 'baza'
select * from sys.databases
select * from sys.database_files
SELECT * FROM SYS.master_files
sp_helprotect
select * from sys.dm_exec_sessions 
select * from sys.dm_exec_connections 
xp_readerrorlog 0,1,"NESTO",null,null,null,[asc]
select DB_NAME(database_id),* from sys.master_files
select * from sys.database_principals     
select * from sys.server_principals     
select * from sys.messages            
select * from sys.dm_db_log_space_usage  
select * from sys.key_encryptions
select * from sysjobsteps
select * from syssessions
select * from sysjobactivity
select * from sysjobhistory

--SHEMA OPCIJE
SELECT SCHEMA_NAME();
CREATE SCHEMA TEST
SELECT * FROM SYS.schemas

EXEC sp_configure 'show advanced options', 1;  

--COUNT TABELA U BAZI
select count(*) from INFORMATION_SCHEMA.TABLES
   where TABLE_TYPE = 'base table'
-----------------------------------------------------------------------------------------------------------------



SELECT * FROM [dbo].[tester] 
WHERE (convert(date, substring(datum_pohrane,1,10)) >= '2021-01-01' AND convert(date, substring(datum_pohrane,1,10)) <= '2021-09-11')


SELECT *, $PARTITION.PartitionFunctionblPartition(DateOfEntry) as PartitionNumber from [dbo].[tb1Partition]



select * from sys.dm_os_waiting_tasks
where session_in in (58, 59)

SELECT @@SERVERNAME, 
SERVERPROPERTY('ComputerNamePhysicalNetBIOS') AS [Local Machine Name],  
(cpu_count / hyperthread_ratio) AS [Physical CPUs],  
hyperthread_ratio AS [Hyperthread Ratio],  
cpu_count AS [Logical CPUs],  
softnuma_configuration AS [Soft-NUMA Configuration],  
softnuma_configuration_desc AS [Soft-NUMA Description],  
socket_count AS [Available Sockets],  
numa_node_count AS [Available NUMA Nodes]  
FROM  
sys.dm_os_sys_info;  



select * from sys.dm_tran_locks
select * from sys.dm_exec_requests

select * from sys.dm_os_schedulers   -- os


select count(database_id)*8/1024.0 as [Cache in Mb], database_id from sys.dm_buffer_descriptors group by database_id


select EOMONTH(DATEADD(month,-1, GETDATE()))  as datum
select  EOMONTH(CONVERT(DATE, GETDATE()))
SET @PERIOD = convert(varchar(10), DATEADD(month,-6, GETDATE()),126) + ' 23:59:00'






--ALTER STATE WRITE READ ONLY READ-ONLY
-----------------------------------------------------------------------------------------------------------------
ALTER DATABASE [SMSDB_WEBIZVODI]
SET READ_ONLY WITH NO_WAIT
GO
ALTER DATABASE [SMSDB_WEBIZVODI]
SET READ_WRITE WITH NO_WAIT
GO
-----------------------------------------------------------------------------------------------------------------





--PROVJERA PORTA:
select distinct local_net_address, local_tcp_port from sys.dm_exec_connections where local_net_address is not null

--CHECK PORT INSTANCE PROVJERA
-----------------------------------------------------------------------------------------------------------------
USE master
GO
xp_readerrorlog 0, 1, N'Server is listening on', N'any', NULL, NULL, N'asc' 
GO
-----------------------------------------------------------------------------------------------------------------






--REPLACE
SELECT @_restoreLogpath = REPLACE(@_restoreLogpath,'\\DIDI195192','\\testcent')






--AUDIT SEARCH
-----------------------------------------------------------------------------------------------------------------
SELECT event_time, server_principal_name, statement FROM sys.fn_get_audit_file('C:\Audit\TODO\*.sqlaudit', DEFAULT, DEFAULT)
WHERE SERVER_PRINCIPAL_NAME LIKE '%amar%'
-----------------------------------------------------------------------------------------------------------------





USE [test99]
GRANT EXECUTE ON OBJECT::[dbo].[sms] TO [TODO\jovana.prodan];  
GO  





-- ZANIMLJIV CREATE TEMP TABLE 
-----------------------------------------------------------------------------------------------------------------
declare @temp table
(
[MB firme] varchar(7), 
[Naziv firme] varchar(100),
[Imaoc kartice] varchar(100),
[Vrsta kartice] varchar(50),
[Skriveni broj kartice] char(19),
[Datum isteka limita u TM-u] datetime,
[Iznos limita] varchar(10),
[Naziv org. dijela gdje se kartica nalazi] char(45), 
[Sales Manager] varchar(50),
SM char(7),
email varchar(150)
);

INSERT into @temp  Exec prcBCMailNotification '';

select distinct SM, email, 'Informacije o business karticama koje su zaprimljene u agencijama' as subject from @temp where SM is not null and email is not null;
-----------------------------------------------------------------------------------------------------------------









--Prepis kroz cmd
-----------------------------------------------------------------------------------------------------------------
/*
dtexec /f "c:\"
*/
-----------------------------------------------------------------------------------------------------------------










--CREATE SNAPSHOT
-----------------------------------------------------------------------------------------------------------------
CREATE DATABASE dba_20230218 ON  
( 
NAME = dba, 
FILENAME =   'D:\dba_20230218.ss' )  
AS SNAPSHOT OF dba;  
GO  
-----------------------------------------------------------------------------------------------------------------














--REPORT SERVER KAD NE RADI RDL POGLEDATI USER ID
-----------------------------------------------------------------------------------------------------------------
select * from dbo.Subscriptions s
left join dbo.Users u
on s.OwnerID = u.UserID
where s.OwnerID != '3D95520E-C9A7-4847-905C-24F22EC3D69F'


update dbo.Subscriptions
set OwnerID = '3D95520E-C9A7-4847-905C-24F22EC3D69F'

select * from dbo.Users where UserID = '3D95520E-C9A7-4847-905C-24F22EC3D69F'
--------------------------------------

use ReportServer
go

select * from dbo.Subscriptions
where Description like '%natasa.ducic@TAMBURAgroup.ba%'

use ReportServer
go
select * from [dbo].[Catalog]
where ItemID = '6B496E1F-5014-49CB-A1D1-D79498B43256'


begin transaction
delete from dbo.Subscriptions 
where subscriptionid = '4E78D37B-24A0-4125-83F4-13E0FFFC56EA'
-----------------------------------------------------------------------------------------------------------------








--RDL POKRENI
--PRONACI AGENT ID IZ SUBSRIPTION ID SSRS
-----------------------------------------------------------------------------------------------------------------
use ReportServer
go

select * from dbo.Subscriptions
where Description like '%Compliance_monitoring_transakcija%'

select j.job_id
from ReportServer.dbo.Subscriptions s
join ReportSchedule r on r.SubscriptionID = s.SubscriptionID
join msdb.dbo.sysjobs j on j.name = convert(sysname, r.ScheduleID)
where s.SubscriptionID = 'DA5C8EE7-DCF0-42CC-B2F9-E1B5936AFB87';


EXEC [uattestdb\report].msdb.dbo.sp_start_job @job_name = '43119915-3861-4BF9-B9CE-ECDF6DB7309C'

SELECT
c .Name AS ReportName
, rs . ScheduleID AS JOB_NAME
, s . [Description]
, s . LastStatus
, s . LastRunTime
FROM
ReportServer ..[Catalog] c
JOIN ReportServer .. Subscriptions s ON c. ItemID = s. Report_OID
JOIN ReportServer .. ReportSchedule rs ON c. ItemID = rs. ReportID
AND rs . SubscriptionID = s . SubscriptionID
where c.Name = 'EKSLOanReport'
-----------------------------------------------------------------------------------------------------------------





-- PROVJERA SVIH RDLA

	SELECT
       c.Name as [ReportName]
      ,su.Description as [SubscriptionDesc]
      ,c.Path as [ReportPath]
      ,coalesce(NULLIF(pc.Path,''),'/') as [ReportFolder]
      ,su.EventType as [SubscriptionType]
      ,replace(su.DeliveryExtension,'Report Server ', '') as [DeliveryExtension]
      ,su.LastRunTime
      ,su.LastStatus
      ,su.SubscriptionID as [SubscriptionID]
    FROM Subscriptions su
     left JOIN Catalog c
    ON su.Report_OID = c.ItemID
      left join Catalog pc on c.ParentID=pc.ItemID
	  where su.laststatus  like '%@TAMBURAgroup.ba%'
    ORDER BY su.LastRunTime  desc, ReportPath, ReportName
	-----------------------------------------------------------------------------------------------------------------
	
	
	
	
	
	
	
	
-- PROVJERA SSRS SVIH DATA SOURCEA
	-----------------------------------------------------------------------------------------------------------------
SELECT
    C2.Name AS Data_Source_Name,
    C.Name AS Dependent_Item_Name,
    C.Path AS Dependent_Item_Path
FROM
    ReportServer.dbo.DataSource AS DS
        INNER JOIN
    ReportServer.dbo.Catalog AS C
        ON
            DS.ItemID = C.ItemID
                AND
            DS.Link IN (SELECT ItemID FROM ReportServer.dbo.Catalog
                        WHERE Type = 5) --Type 5 identifies data sources
        FULL OUTER JOIN
    ReportServer.dbo.Catalog C2
        ON
            DS.Link = C2.ItemID
WHERE
    C2.Type = 5
	AND C2.Name = 'BOSNADB GRAS (PRIMARY SITE)' or C2.Name = 'SARAJDB test (PRIMARY SITE)'
ORDER BY
    C2.Name ASC,
    C.Name ASC;
		-----------------------------------------------------------------------------------------------------------------
--Provjera lista reporta
--------------------------------------------------------------------
SELECT
  ItemID -- Unique Identifier
, [Path] --Path including object name
, [Name] --Just the objectd name
, ParentID --The ItemID of the folder in which it resides
, CASE [Type] --Type, an int which can be converted using this case statement.
    WHEN 1 THEN 'Folder'
    WHEN 2 THEN 'Report'
    WHEN 3 THEN 'File'
    WHEN 4 THEN 'Linked Report'
    WHEN 5 THEN 'Data Source'
    WHEN 6 THEN 'Report Model - Rare'
    WHEN 7 THEN 'Report Part - Rare'
    WHEN 8 THEN 'Shared Data Set - Rare'
    WHEN 9 THEN 'Image'
    ELSE CAST(Type as varchar(100))
  END AS TypeName
--, content
, LinkSourceID --If a linked report then this is the ItemID of the actual report.
, [Description] --This is the same information as can be found in the GUI
, [Hidden] --Is the object hidden on the screen or not
, CreatedBy.UserName CreatedBy
, CreationDate

FROM 
  ReportServer.dbo.[Catalog] CTG
    INNER JOIN 
  ReportServer.dbo.Users CreatedBy ON CTG.CreatedByID = CreatedBy.UserID
--------------------------------------------------------------------


















--SENT RDL KROZ MSDB JOB
-----------------------------------------------------------------------------------------------------------------
exec dbo.insertReport '/_DBMGM_/Ostalo/DBA Reports/FailedReports', 'FailedReports', 'Send e-mail to dbarequest@test.ba'
-----------------------------------------------------------------------------------------------------------------
CREATE PROCEDURE [dbo].[insertReport] 
	@rPath nvarchar(425), 
	@rName nvarchar(425),
	@rDescription nvarchar(512)
AS
BEGIN

	insert into [reportservice\report].dba.dbo.tblReport ([path], [name], [description]) values (@rPath, @rName, @rDescription)
END
GO
-----------------------------------------------------------------------------------------------------------------











--Memory OPT. Temp TABELE
-----------------------------------------------------------------------------------------------------------------
ALTER DATABASE AdventureWorks2017 
ADD FILEGROUP AdventureWorks2017_MemoryOptFileGroup CONTAINS MEMORY_OPTIMIZED_DATA;
GO

ALTER DATABASE AdventureWorks2017 
ADD FILE (name='AdventureWorks2017_MemoryOptFile', 
filename='C:\Program Files\Microsoft SQL Server\MSSQL15.MSSQLSERVER\MSSQL\DATA\AdventureWorks2017Mem.ldf') 
TO FILEGROUP AdventureWorks2017_MemoryOptFileGroup
-----------------------------------------------------------------------------------------------------------------











-----------------------------------------------------------------------------------------------------------------
use SSISDB
go
open master key decryption by password = 'test'
ALTER MASTER KEY FORCE REGENERATE WITH ENCRYPTION BY PASSWORD = 'test' 

sp_removedbreplication 'test'
sp_removedbreplication 'test99'

--ako se bas mora
backup master key to file = 'C:\ddd\masterkey' 
encryption by password = 'test'
 
Restore master key from file ='C:\powershell\masterkey'
Decryption by password = 'test'
Encryption by password = 'test'   
open master key decryption by password = 'test'


-----------------------------------------------------------------------------------------------------------------









--CHANGE OWNER
-----------------------------------------------------------------------------------------------------------------
USE GRAS
GO
EXEC sp_changedbowner 'sa'
GO
-----------------------------------------------------------------------------------------------------------------





use [T24DataTransformation]
GO
GRANT SELECT ON SCHEMA::[S2] TO [TODO\test] AS [S2]
GO





-----------------------------------------------------------------------------------------------------------------
CREATE ROLE [db_viewdefinitions] AUTHORIZATION [dbo]
GRANT VIEW DEFINITION TO [db_viewdefinitions]
ALTER ROLE [db_viewdefinitions] ADD MEMBER [testu1]
-----------------------------------------------------------------------------------------------------------------
USE AdventureWorks 
GO 
GRANT VIEW Definition TO User1
USE master  
GO  
REVOKE VIEW ANY DEFINITION TO User1  
or 
USE AdventureWorks  
GO  
REVOKE VIEW Definition TO User1 
grant alter ON SCHEMA::dbo to [TODO\niain-ext]
-----------------------------------------------------------------------------------------------------------------











-----------------------------------------------------------------------------------------------------------------
ALTER USER amrado
WITH LOGIN = amrado;
GO
CREATE USER [TODO\test] FOR LOGIN [TODO\test] WITH DEFAULT_SCHEMA=[dbo]
GO
ALTER ROLE [db_datareader] ADD MEMBER [TODO\vte]
execute sp_addrolemember RoleName, UserName
-----------------------------------------------------------------------------------------------------------------




-- provjera joba po hash id
select * from msdb.dbo.sysjobs where CONVERT(binary(16), job_id)=0x069A2F69592ECE4BA1F350DB65562089






-----------------------------------------------------------------------------------------------------------------
use [hr_nav]
GO
GRANT SELECT ALL USER SECURABLES TO [test]
GO
-----------------------------------------------------------------------------------------------------------------







--TIMEZONE
-----------------------------------------------------------------------------------------------------------------
select * from sys.time_zone_info
DECLARE @TimeZone VARCHAR(50)
EXEC MASTER.dbo.xp_regread 'HKEY_LOCAL_MACHINE',
'SYSTEM\CurrentControlSet\Control\TimeZoneInformation',
'TimeZoneKeyName',@TimeZone OUT
SELECT @TimeZone
-----------------------------------------------------------------------------------------------------------------











--INDEXI PROVJERA:
-----------------------------------------------------------------------------------------------------------------
SELECT S.name as 'Schema',
T.name as 'Table',
I.name as 'Index',
DDIPS.avg_fragmentation_in_percent,
DDIPS.page_count
FROM sys.dm_db_index_physical_stats (DB_ID(), NULL, NULL, NULL, NULL) AS DDIPS
INNER JOIN sys.tables T on T.object_id = DDIPS.object_id
INNER JOIN sys.schemas S on T.schema_id = S.schema_id
INNER JOIN sys.indexes I ON I.object_id = DDIPS.object_id
AND DDIPS.index_id = I.index_id
WHERE DDIPS.database_id = DB_ID()
and I.name is not null
AND DDIPS.avg_fragmentation_in_percent > 0
ORDER BY DDIPS.avg_fragmentation_in_percent desc
-----------------------------------------------------------------------------------------------------------------













--AGENT - KREIRANJE MASTER TARGER ZA JOBOVE:
-----------------------------------------------------------------------------------------------------------------
use msdb
go
sp_help_targetserver

use msdb
go
EXEC dbo.sp_delete_targetserver 
 @server_name = N'test' ;

---------------------------------
use msdb
go
sp_msx_defect

use msdb
go
EXECUTE msdb.dbo.sp_delete_targetserver @server_name = 't24uatsarajdb', @post_defection = 0:

sp_delete_targetserver [ @server_name = ] 't24uatsarajdb'
[ , [ @clear_downloadlist = ] clear_downloadlist ]
[ , [ @post_defection = ] post_defection ]

--------------------------------------------
--TARGET SERVER ADD & DELETE
--------------------------------------------
use msdb
go
sp_help_targetserver

use msdb
go
EXEC dbo.sp_delete_targetserver 
 @server_name = N't24uatsarajdb' ;

use msdb
go
sp_msx_defect -- @forced_defection = 1

use msdb
go
EXECUTE msdb.dbo.sp_delete_targetserver @server_name = 't24uatsarajdb', @post_defection = 0:

--PROVJERE
EXEC dbo.sp_resync_targetserver N'test';
------------------------------------------
-----------------------------------------------------------------------






--Izbaciti master agenta i ponovo refresh uraditi
--uzmi jobove trenutno na targetu. Poslije zadnjeg stepa execute na Masteru dobivenu skriptu.
-----------------------------------------------------------------------------------------------------------------
USE msdb;
GO
SELECT
    'EXEC msdb.dbo.sp_add_jobserver
    @job_name = N''' + 
    name + ''',
    @server_name = N''' + @@SERVERNAME + ''';' AS cmd
FROM
    dbo.sysjobs
WHERE
    originating_server_id = 1;


USE msdb;
GO
EXEC dbo.sp_msx_defect;

USE msdb;
GO
EXEC dbo.sp_msx_enlist N'mssqlservicesdb\agent';
-----------------------------------------------------------------------------------------------------------------













-----------------------------------------------------------------------------------------------------------------
USE TEMPDB;    
GO    
CHECKPOINT;  

DBCC DROPCLEANBUFFERS WITH NO_INFOMSGS;  
DBCC FREEPROCCACHE WITH NO_INFOMSGS;  
DBCC FREESESSIONCACHE WITH NO_INFOMSGS;  
DBCC FREESYSTEMCACHE ('ALL');  
DBCC FREESYSTEMCACHE ('SQL Plans');
-----------------------------------------------------------------------------------------------------------------






--force index sa Option
-----------------------------------------------------------------------------------------------------------------
select datum_dok, DATum_nal, COUNT(*) from st1.dbo.test a 
where vrdok like 'T24%'
group by datum_dok, datum_nal
order by 1 desc
OPTION (TABLE HINT(a, INDEX (IX_analitika_vrdok_2024)))
-----------------------------------------------------------------------------------------------------------------









--ACTIVE USERS, provjera usera:
-----------------------------------------------------------------------------------------------------------------
--pregled usera koji su zadnji konektovani, po datumu, multiple time preview
SELECT LEFT(d.connect_time, 17) + ' h' as LastTime, g.login_name AS LogUser, g.host_name AS Host, d.client_net_address AS IPAdd, d.auth_scheme AS AUTH, d.protocol_type AS Protocol, d.net_transport AS NET   
FROM sys.dm_exec_sessions g    
INNER JOIN sys.dm_exec_connections d        
ON g.session_id = d.session_id      
WHERE g.login_name NOT LIKE 'TODO\MSSQL%'    
ORDER BY d.connect_time desc
GO

----------------------------------------------------
--pregled usera opcenito koji su bili konektovani Distinct, single time preview.
SELECT DISTINCT g.login_name AS LogUser, g.host_name AS Host, d.client_net_address AS IPAdd, d.auth_scheme AS AUTH, d.protocol_type AS Protocol, d.net_transport AS NET
    FROM sys.dm_exec_sessions g
    INNER JOIN sys.dm_exec_connections d
        ON g.session_id = d.session_id
    ORDER BY g.login_name
GO
-----------------------------------------------------------------------------------------------------------------

SELECT net_transport, auth_scheme   
FROM sys.dm_exec_connections   
WHERE session_id = @@SPID;

--Provjera trenutnih usera
-----------------------------------------------------------------------------------------------------------------
SELECT 
    DB_NAME(dbid) as DBName, 
    COUNT(dbid) as NumberOfConnections,
    loginame as LoginName
FROM
    sys.sysprocesses
WHERE 
    dbid > 0
GROUP BY 
    dbid, loginame
-----------------------------------------------------------------------------------------------------------------







--SERVICE BROKER
-----------------------------------------------------------------------------------------------------------------
USE [master]
GO
SELECT   [name]
 ,[is_broker_enabled] 
 ,[service_broker_guid]
FROM [sys].[databases] 
GO
USE [master]
GO
ALTER DATABASE [FIMService] SET NEW_BROKER
GO
ALTER DATABASE [FIMService] set enable_broker
-----------------------------------------------------------------------------------------------------------------










--BACKUP RESTORE OPCIJE
-----------------------------------------------------------------------------------------------------------------
BACKUP LOG [testDB] TO DISK = N'D:\testdb2.TRN'
RESTORE LOG [testDB] FROM  DISK = N'D:\testdb2.TRN' WITH  FILE = 1,  NORECOVERY,  NOUNLOAD,  STATS = 5

--BACKUP KROZ SQLCMD 
--------------------------------------------------------------------------------
sqlcmd -E -S $(ESCAPE_SQUOTE(SRVR)) -d dba -Q "EXECUTE [dbo].[DatabaseBackup] @Databases = 'GG30_live,RFB,satest', @Directory = N'\\DIDI195192\DIDI195', @BackupType = 'FULL', @Verify = 'N',@Compress = 'Y', @CleanupTime = 1, @CheckSum = 'Y'" -b
sqlcmd -E -S $(ESCAPE_SQUOTE(SRVR)) -d dba -Q "EXECUTE [dbo].[DatabaseBackup] @Databases = 'SYSTEM_DATABASES', @Directory = N'\\DIDI195192\DIDI195', @BackupType = 'FULL', @Verify = 'Y', @CleanupTime = 95, @CheckSum = 'Y'" -b
--------------------------------------------------------------------------------
================================================================================
--BACKUP WITH CMD
sqlcmd -E -S $(ESCAPE_SQUOTE(SRVR)) -d dba -Q "EXECUTE [dbo].[DatabaseBackup] @Databases = 'G3_test,GG30_test', @Directory = N'\\DIDI195192\DIDI195', @BackupType = 'FULL', @Verify = 'N',@Compress = 'Y', @CheckSum = 'Y'" -b
================================================================================
--SQLCMD BACKUP 2 DANA
===========================================================
sqlcmd -E -S $(ESCAPE_SQUOTE(SRVR)) -d dba -Q "EXECUTE [dbo].[DatabaseBackup] @Databases = 'GRAS', @Directory = N'\\DIDI195192\DIDI195_HIGH', @BackupType = 'FULL', @Verify = 'Y', @Compress = 'Y', @CleanupTime = 192, @CheckSum = 'Y'" -b
===========================================================
BACKUP DATABASE [ReportServerTempDB] TO  DISK = N'C:\BACKUP REPORT\ReportServerTempDB1.bak' WITH NOFORMAT, NOINIT,  NAME = N'ReportServer-Full Database Backup', SKIP, NOREWIND, NOUNLOAD,  STATS = 10
GO



-- RESTORE KROZ JOB SA SQLCMD
sqlcmd -E -S $(ESCAPE_SQUOTE(SRVR)) -d master -Q "RESTORE DATABASE [satest] FROM DISK = N'\\DIDI195192\sqluat\NEW DATASOURCE\DATASOURCE\satest\FULL\DATASOURCE_satest_FULL_20230101_124437.bak' WITH NOUNLOAD, NORECOVERY, REPLACE, STATS = 5;" -b

-- POWERSHELL RESTORE DB
-----------------------------------------------------------------------------------------------------------------
/*
###### START OF POWERSHELL SCRIPT USED FOR DSOLA RESTORE CBS DATABASES
$databaselist = Get-Content "C:\RESTORE\dsola_cbs.txt"
$server = 'dsola'
$RestoreTime = Get-Date('03:30 01/07/2022')


foreach ($database in $databaselist) {

$full = '\\DIDI195192\SQLuat\NEW DATASOURCE\20220630\'+$database+'\FULL'
$diff = '\\DIDI195192\SQLuat\NEW DATASOURCE\20220630\'+$database+'\DIFF'
$log = '\\DIDI195192\SQLuat\NEW DATASOURCE\20220630\'+$database+''

Restore-DbaDatabase -SqlInstance $server -DatabaseName $database -path $full -NoRecovery -WithReplace
Restore-DbaDatabase -SqlInstance $server -DatabaseName $database -path $diff -NoRecovery
Restore-DbaDatabase -SqlInstance $server -DatabaseName $database -path $log -Continue -RestoreTime $RestoreTime
}

###END OF POWERSHELL SCRIPT
*/
-----------------------------------------------------------------------------------------------------------------
-- SA PROCEDUROM BACKUP
EXECUTE [dbo].[DatabaseBackup] @Databases = 'GG30_live', 
@Directory = N'\\DIDI195192\DIDI195', 
@BackupType = 'FULL', 
@Verify = 'Y', 
@Compress = 'Y',
@CheckSum = 'Y'
-----------------------------------------------------------------------------------------------------------------




--Neke powershell komande
-----------------------------------------------------------------------------------------------------------------
$computer = $env:COMPUTERNAME
$port = "5022"
Test-NetConnection -ComputerName "t24db1" -Port $port
-----------------------------------------------------------------------------------------------------------------








--PROVJERA LOG READER AGENT REPLIKACIJA
-----------------------------------------------------------------------------------------------------------------
  SELECT SERVER
    ,[command]
    ,sj.job_id
    ,[NAME]
FROM msdb.dbo.sysjobs sj
INNER JOIN msdb.dbo.sysjobsteps sjs ON sjs.job_id = sj.job_id
    AND subsystem = 'logreader'
-----------------------------------------------------------------------------------------------------------------













-- PROMJENA DATA BAZE
-----------------------------------------------------------------------------------------------------------------
--PROVJERA i STATUS BAZE
SELECT name, physical_name AS NewLocation, state_desc AS OnlineStatus
FROM sys.master_files  
WHERE database_id = DB_ID(N'tieto')  
GO

--PREBACIVANJE FAJLOVA NA NOVU LOKACIJU
ALTER DATABASE tieto
    MODIFY FILE ( NAME = tieto,   
                  FILENAME = 'I:\MSSQL\Data\tieto.mdf');  
GO

ALTER DATABASE tieto
    MODIFY FILE ( NAME = tieto_log,   
                  FILENAME = 'H:\MSSQL\Data\tieto_log.ldf');  
GO

--BAZU PREBACITI U OFFLINE - Sljedeca skripta
execute msdb..usp_killusers 'tieto'
execute msdb..usp_killusers 'tieto'
execute msdb..usp_killusers 'tieto'
ALTER DATABASE tieto SET OFFLINE;  
GO


--PREBACIT FAJLOVE RUCNO NA NOVU LOKACIJU
--VRATITI BAZU ONLINE
ALTER DATABASE tieto SET ONLINE;  
GO
-----------------------------------------------------------------------------------------------------------------






--promjena
USE master;
GO
ALTER DATABASE KremaSds SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
GO
ALTER DATABASE KremaSds MODIFY NAME = KremaSds_20240806;
GO
ALTER DATABASE KremaSds_20240806 SET MULTI_USER;
GO











--REPAIR
-----------------------------------------------------------------------------------------------------------------
ALTER DATABASE sqlcentric 
SET single_user WITH 
ROLLBACK immediate; 
go 
DBCC checkdb (sqlcentric, repair_allow_data_loss); 
go
/*
REPAIR_ALLOW_DATA_LOSS - Tries to repair all reported errors. These repairs can cause some data loss.
REPAIR_FAST - Maintains syntax for backward compatibility only. No repair actions are performed.
REPAIR_REBUILD */

dbcc opentran()
DBCC CHECKDB (N ’Dbtesting’, REPAIR_ALLOW_DATA_LOSS) WITH ALL_ERRORMSGS, NO_INFOMSGS; 
dbcc traceon (1204, -1)
dbcc traceon (1222, -1)
DBCC TRACEON (1204, -1);
DBCC TRACESTATUS (1204);
DBCC TRACEOFF (1204, -1);


dbcc shrinkfile(DB_log, 3)
-----------------------------------------------------------------------------------------------------------------





-- PROVJERA DISKA MISO
-----------------------------------------------------------------------------------------------------------------
    SELECT Drive
    ,   TotalSpaceGB
    ,   FreeSpaceGB
    ,   PctFree
    ,   PctFreeExact
    FROM
    (SELECT DISTINCT
        SUBSTRING(dovs.volume_mount_point, 1, 10) AS Drive
    ,   CONVERT(INT, dovs.total_bytes / 1024.0 / 1024.0 / 1024.0) AS TotalSpaceGB
    ,   CONVERT(INT, dovs.available_bytes / 1048576.0) / 1024 AS FreeSpaceGB
    ,   CAST(ROUND(( CONVERT(FLOAT, dovs.available_bytes / 1048576.0) / CONVERT(FLOAT, dovs.total_bytes / 1024.0 /
                         1024.0) * 100 ), 2) AS NVARCHAR(50)) + '%' AS PctFree
    ,   CONVERT(FLOAT, dovs.available_bytes / 1048576.0) / CONVERT(FLOAT, dovs.total_bytes / 1024.0 / 1024.0) * 100 AS PctFreeExact                
    FROM    sys.master_files AS mf
    CROSS APPLY sys.dm_os_volume_stats(mf.database_id, mf.file_id) AS dovs) AS DE
-----------------------------------------------------------------------------------------------------------------












--CHECK JOBS SA PACKET SSISDB NA AGENTU <<<<<<<<<<<<<<<<<<<<<<<
-----------------------------------------------------------------------------------------------------------------
SELECT j.name,
 js.step_id,
 js.step_name,
 js.command

FROM msdb.dbo.sysjobs j,

 msdb.dbo.sysjobsteps js

WHERE j.job_id = js.job_id
AND js.subsystem = 'SSIS'
and js.command like '%%'
-----------------------------------------------------------------------------------------------------------------

--PROVJERA PALIH SSIS PAKETA
-----------------------------------------------------------------------------------------------------------------
select count(*)
FROM   (
       SELECT  em.*
       FROM    SSISDB.catalog.event_messages em
       WHERE   em.operation_id IN (SELECT execution_id FROM SSISDB.catalog.executions where start_time >  CAST(getdate()-1 as date))
          and event_name NOT LIKE '%Validate%'
       )q
WHERE event_name IN('OnError','OnTaskFailed') and package_path like '\Package%'
-----------------------------------------------------------------------------------------------------------------











--CHECK JOB KOJI SU AKTIVNI
-----------------------------------------------------------------------------------------------------------------
select a.session_id, b.job_id, b.name
from sys.dm_exec_sessions a
	inner join msdb.dbo.sysjobs b on b.job_id = cast(convert( binary(16), substring(a.program_name , 30, 34), 1) as uniqueidentifier)
where program_name like 'SQLAgent - TSQL JobStep (Job % : Step %)'
-----------------------------------------------------------------------------------------------------------------












--DISABLE JOBS SKRIPTA GRUPNO  -- veoma korisno
-----------------------------------------------------------------------------------------------------------------
use [msdb]

DECLARE @name nvarchar (256)
DECLARE @status nvarchar (256)
DECLARE @job_id nvarchar (256)

DECLARE cur CURSOR FOR


select name, [enabled] from msdb..sysjobs order by name

print 'USE [msdb]';
print 'GO';

	OPEN cur

	FETCH NEXT FROM cur INTO @name, @status;
	WHILE @@FETCH_STATUS = 0
	BEGIN
		print 'exec dbo.sp_update_job @job_name = N''' + @name + ''', @enabled = 0;';
		print 'GO'
		FETCH NEXT FROM cur INTO @name, @status;
	END

	CLOSE cur;
	DEALLOCATE cur;
-----------------------------------------------------------------------------------------------------------------
















--CHECK JOBS SA RDL SSRS NA AGENTU
-----------------------------------------------------------------------------------------------------------------
SELECT distinct
sj.[name] AS [Job Name],
rs.SubscriptionID,
c.[Name] AS [Report Name],
c.[Path]


FROM msdb..sysjobs AS sj 

INNER JOIN ReportServer..ReportSchedule AS rs
ON sj.[name] = CAST(rs.ScheduleID AS NVARCHAR(128)) 

INNER JOIN ReportServer..Subscriptions AS su
ON rs.SubscriptionID = su.SubscriptionID

INNER JOIN ReportServer..[Catalog] c
ON su.Report_OID = c.ItemID

USE [msdb]
EXEC sp_start_job @job_name = '[Job Name] from above query'
-----------------------------------------------------------------------------------------------------------------














--POREKTANJE JOBA
-----------------------------------------------------------------------------------------------------------------
USE msdb;
GO
-- Create a user to run the job.
CREATE USER CanRunAnyJob WITHOUT LOGIN;
-- Add user CanRunAnyJob to the role SQLAgentOperatorRole
ALTER ROLE SQLAgentOperatorRole ADD MEMBER CanRunAnyJob;
GO
-- Create a stored procedure that executes the job
CREATE PROCEDURE sp_RunThisJob
WITH EXECUTE AS 'CanRunAnyJob'
AS
EXEC sp_start_job 'ThisJob';
GO
-----------------------------------------------------------------------------------------------------------------

















--SLANJE MAILA POMOCU AGENT JOBA:
-----------------------------------------------------------------------------------------------------------------
EXEC msdb.dbo.sp_send_dbmail
    @profile_name = 'MAIL',
    @recipients = 'amilatest.ba',
    @body = 'Postovani, procedure uspjesno zavrsene.',
    @subject = 'Success JOB: cd_daily_one' ;
-----------------------------------------------------------------------------------------------------------------














--PRETRAGA RDL SUBSCRIPTION CATALOG
-----------------------------------------------------------------------------------------------------------------
SELECT c.Path AS RDLPath,c.Name AS RDLName,s.Description AS SubscriptionName,s.LastStatus,s.LastRunTime 
FROM [ReportServer].[dbo].[Catalog] AS c
left join [ReportServer].[dbo].[Subscriptions] AS s
on c.ItemID = s.Report_OID
where s.Report_OID = '25114B6A-6FBC-4B10-BEC3-3FD9A1009D6D'
-----------------------------------------------------------------------------------------------------------------














--CHECK RDL FAILS
-----------------------------------------------------------------------------------------------------------------

TRUNCATE TABLE RDLSubs.Logs;

INSERT INTO RDLSubs.Logs (RDLPath,RDLName,SubscriptionName,LastStatus,LastRunTime)
SELECT c.Path AS RDLPath,c.Name AS RDLName,s.Description AS SubscriptionName,s.LastStatus,s.LastRunTime 
FROM OPENROWSET('SQLNCLI', 'Server=reportservice\report;Trusted_Connection=yes;',
                 [ReportServer].[dbo].[Catalog]) AS c
left join OPENROWSET('SQLNCLI', 'Server=reportservice\report;Trusted_Connection=yes;',
                 [ReportServer].[dbo].[Subscriptions]) AS s
on c.ItemID = s.Report_OID
where CONVERT(INT,SUBSTRING(s.LastStatus,CHARINDEX(';', s.LastStatus)+1,(LEN(s.LastStatus)-CHARINDEX(';', s.LastStatus)-8))) > 0
and s.LastStatus like '%errors.%'
order by s.LastRunTime desc, c.Name


-----------------------------------------------------------------------------------------------------------------
exec dbo.insertReport '/_DBMGM_/Ostalo/DBA Reports/FailedReports', 'FailedReports', 'Send e-mail to dbarequest@TAMBURAgroup.ba'



-- KADA PRAVIM RDL. DATASOURCE
-----------------------------------------------------------------------------------------------------------------
SELECT 
RDLPath,
RDLName,
SubscriptionName,
LastStatus,
LastRunTime
FROM RDLSubs.Logs
WHERE LastRunTime > CAST(CAST(GETDATE() AS DATE) AS DATETIME)
ORDER BY LastRunTime DESC
-----------------------------------------------------------------------------------------------------------------









-- PREGLED JOB
-----------------------------------------------------------------------------------------------------------------
SELECT j.job_id,
s.srvname,
j.name,
js.step_id,
js.command,
j.enabled
FROM msdb.dbo.sysjobs j
JOIN msdb.dbo.sysjobsteps js
ON js.job_id = j.job_id
JOIN master.dbo.sysservers s
ON s.srvid = j.originating_server_id
WHERE js.command LIKE N'%KEYWORD_SEARCH%'
-----------------------------------------------------------------------------------------------------------------














--MOJA PREPRAVKA RDL JOBA ZA PRACENJE
-----------------------------------------------------------------------------------------------------------------
TRUNCATE TABLE RDLSubs.Logs;

INSERT INTO RDLSubs.Logs (RDLPath,RDLName,SubscriptionName,LastStatus,LastRunTime)
SELECT c.Path AS RDLPath,c.Name AS RDLName,s.Description AS SubscriptionName,s.LastStatus,s.LastRunTime 
FROM OPENROWSET('SQLNCLI', 'Server=reportservice\report;Trusted_Connection=yes;',
                 [ReportServer].[dbo].[Catalog]) AS c 
left join OPENROWSET('SQLNCLI', 'Server=reportservice\report;Trusted_Connection=yes;',
                 [ReportServer].[dbo].[Subscriptions]) AS s 
on c.ItemID = s.Report_OID 
    where CONVERT(INT,SUBSTRING(s.LastStatus,CHARINDEX(';', s.LastStatus)+1,(LEN(s.LastStatus)-CHARINDEX(';', s.LastStatus)-8))) > 0
    and s.LastStatus like '%errors.%' 
UNION ALL 
SELECT c.Path AS RDLPath,c.Name AS RDLName,s.Description AS SubscriptionName,CONCAT(SUBSTRING(s.LastStatus,1,40), '...'),s.LastRunTime 
FROM OPENROWSET('SQLNCLI', 'Server=reportservice\report;Trusted_Connection=yes;',
                 [ReportServer].[dbo].[Catalog]) AS c 
left join OPENROWSET('SQLNCLI', 'Server=reportservice\report;Trusted_Connection=yes;',
                 [ReportServer].[dbo].[Subscriptions]) AS s 
on c.ItemID = s.Report_OID 
     where s.LastStatus like 'Error%'
order by s.LastRunTime desc, c.Name
-----------------------------------------------------------------------------------------------------------------
TRUNCATE TABLE RDLSubs.Logs;
INSERT INTO RDLSubs.Logs (RDLPath,RDLName,SubscriptionName,LastStatus,LastRunTime)
	  SELECT c.Path AS RDLPath,c.Name AS RDLName,s.Description AS SubscriptionName,s.LastStatus,s.LastRunTime 
      FROM OPENROWSET('SQLNCLI', 'Server=reportservice\report;Trusted_Connection=yes;',
                       [ReportServer].[dbo].[Catalog]) AS c 
      left join OPENROWSET('SQLNCLI', 'Server=reportservice\report;Trusted_Connection=yes;',
                       [ReportServer].[dbo].[Subscriptions]) AS s 
      on c.ItemID = s.Report_OID 
          where CONVERT(INT,SUBSTRING(s.LastStatus,CHARINDEX(';', s.LastStatus)+1,(LEN(s.LastStatus)-CHARINDEX(';', s.LastStatus)-8))) > 0
          and s.LastStatus like '%errors.%' 
UNION ALL 
	  SELECT c.Path AS RDLPath,c.Name AS RDLName,s.Description AS SubscriptionName,CONCAT(SUBSTRING(s.LastStatus,1,40), '...'),s.LastRunTime 
      FROM OPENROWSET('SQLNCLI', 'Server=reportservice\report;Trusted_Connection=yes;',
                       [ReportServer].[dbo].[Catalog]) AS c 
      left join OPENROWSET('SQLNCLI', 'Server=reportservice\report;Trusted_Connection=yes;',
                       [ReportServer].[dbo].[Subscriptions]) AS s 
      on c.ItemID = s.Report_OID 
		   WHERE (s.LastStatus like 'Error%')
		   AND 
		   (
		   (c.Name NOT LIKE 'LCR_C74-all') AND 
		   (c.Name NOT LIKE 'C72_izvjestaj') AND 
		   (c.Name NOT LIKE 'LCR_C74_sumarni') AND 
		   (c.Name NOT LIKE 'C72_sumarni') AND 	   
		   (c.Name NOT LIKE 'LCR_C73_sumarni') AND 
		   (c.Name NOT LIKE 'LCR_C73-all') AND 
		   (c.Name NOT LIKE 'LCR_C73-biljeske') AND
		   (s.Description NOT LIKE '%mjesecni%')
		   )
UNION ALL 
           SELECT c.Path AS RDLPath,c.Name AS RDLName,s.Description AS SubscriptionName,CONCAT(SUBSTRING(s.LastStatus,1,40), '...'),s.LastRunTime
           FROM OPENROWSET('SQLNCLI', 'Server=reportservice\report;Trusted_Connection=yes;',
                            [ReportServer].[dbo].[Catalog]) AS c 
           left join OPENROWSET('SQLNCLI', 'Server=reportservice\report;Trusted_Connection=yes;',
                            [ReportServer].[dbo].[Subscriptions]) AS s 
           on c.ItemID = s.Report_OID 
           where
           CONVERT(DATE, s.LastRunTime) = EOMONTH(CONVERT(DATE, GETDATE()))
           AND (s.LastStatus like 'Error%')
order by s.LastRunTime desc, c.Name



--MOJA PREPRAVKA REPLIKACIAJ TABELA ZA PRACENJE
-----------------------------------------------------------------------------------------------------------------

SELECT * INTO #dbrep_monitor FROM OPENROWSET('SQLNCLI', 'Server=sarajdbprim;Trusted_Connection=yes;',
     'set fmtonly off; EXEC [distribution].dbo.sp_replmonitorhelppublication with result sets 
	 (
	 (
		publisher_db sysname 
		,publication sysname 
		,publication_id int 
		,publication_type int
		,status int
		,warning int
		,worst_latency int
		,best_latency int
		,avg_latency int
		,last_distsync datetime	
		,retention int
        ,latencythreshold int
        ,expirationthreshold int
		,agentnotrunningthreshold int
        ,subscriptioncount int
        ,runningdistagentcount int
		,snapshot_agentname sysname null
		,logreader_agentname sysname null
		,qreader_agentname sysname null
		,worst_runspeedPerf int
		,best_runspeedPerf int
		,average_runspeedPerf int 
		,retention_period_unit tinyint
        ,publisher sysname null
	 )
	 )')

;WITH QueryALL AS(
            SELECT DISTINCT
            	CAST(temp.last_distsync AS smalldatetime)    AS [Last Sync],
                p.[publication]                           AS [Publication Name],
                 g.[publisher] + '.' + a.[publisher_db]     AS [Source db]
                ,a.[source_owner] + '.' + a.[article]       AS [Table]
            	,CASE
                   WHEN x.subscriber_db = 'test' THEN 'SARAJDB'+ '.'+ x.[subscriber_db]
            	   WHEN x.subscriber_db = 'GRAS' THEN 'BOSNADB'+ '.'+ x.[subscriber_db]
            	   WHEN x.subscriber_db = 'FON' THEN 'pisarDB'+ '.'+ x.[subscriber_db]
            	   WHEN x.subscriber_db = 'CM' THEN 'ITAPPDB'+ '.'+ x.[subscriber_db]
            	   ELSE 'NN'
            	END AS [Destination db]
                --,CAST(G.last_distsync AS smalldatetime)    AS [Last Sync2]
                ,CASE
                    WHEN g.status = 1 and temp.status != 6 and temp.last_distsync IS NOT NULL THEN 'Started'
                    WHEN g.status = 2 and temp.status != 6 and temp.last_distsync IS NOT NULL  THEN 'ok'
                    WHEN g.status = 3 and temp.status != 6 and temp.last_distsync IS NOT NULL  THEN 'ok' --In progress
                    WHEN g.status = 4 and temp.status != 6 and temp.last_distsync IS NOT NULL  THEN 'ok'
                    WHEN g.status = 5 and temp.status != 6 and temp.last_distsync IS NOT NULL  THEN 'Retrying'
                    WHEN g.status = 6 or temp.status = 6 OR temp.last_distsync IS NULL THEN 'Failed'
            		WHEN g.status = 0  THEN 'ok'
                 END AS [Status]
            FROM
                OPENROWSET('SQLNCLI', 'Server=sarajdb;Trusted_Connection=yes;',
                [distribution].[dbo].[MSarticles]) AS a
                INNER JOIN OPENROWSET('SQLNCLI', 'Server=sarajdb;Trusted_Connection=yes;',
            	[distribution].[dbo].[MSpublications]) AS p
                    ON (a.[publication_id] = p.[publication_id])
                INNER JOIN OPENROWSET('SQLNCLI', 'Server=sarajdb;Trusted_Connection=yes;',
            	[distribution].[dbo].MSreplication_monitordata) AS g
                    ON (a.[publication_id] = g.[publication_id])
                INNER JOIN OPENROWSET('SQLNCLI', 'Server=sarajdb;Trusted_Connection=yes;',
            	[distribution].[dbo].[MSsubscriptions]) AS x
            	ON (a.[publication_id] = x.[publication_id])
            	right join #dbrep_monitor as temp
                on x.[publication_id] = temp.[publication_id]
            WHERE (p.[publication] NOT LIKE '%OffloadArea%') AND (x.subscriber_db IN ('test', 'GRAS', 'FON', 'CM'))
)





--INSERT INTO #ReplLogs_prod ([Publication Name], [Source db], [Table], [Destination db], [Last Sync], [Status])
SELECT 
             [Publication Name]
        	,CASE
                WHEN [Source db] = 'pisarDBPRIM\CODE.FON' THEN REPLACE([Source db],'pisarDBPRIM\CODE.FON', 'pisarDB.FON')
                WHEN [Source db] = 'SARAJDBPRIM.test' THEN REPLACE([Source db],'SARAJDBPRIM.test', 'SARAJDB.test')
                WHEN [Source db] = 'SARAJDBPRIM.test99' THEN REPLACE([Source db],'SARAJDBPRIM.test99', 'SARAJDB.test99')
        		WHEN [Source db] = 'BOSNADBPRIM\HEAD.GRAS' THEN REPLACE([Source db],'BOSNADBPRIM\HEAD.GRAS', 'BOSNADB.GRAS')
             END AS [Source db]
            ,[Table]
            ,[Destination db]
            ,MAX([Last Sync]) AS Sync
            ,CASE
                WHEN [Status] = 'The job succeeded.' THEN REPLACE([Status],'The job succeeded.', 'ok')
        		ELSE [Status] END AS [Status]
into #ReplLogs_prod-- ([Publication Name], [Source db], [Table], [Destination db], [Last Sync], [Status])
             FROM QueryALL
             GROUP BY 
                  [Publication Name]
                 ,[Source db]
                 ,[Table]
                 ,[Destination db]
                 ,[Status]
             ORDER BY 2,4


select * from #ReplLogs_prod


-----------------------------------------------------------------------------------------------------------------









--PROMJENA AVAILABILITY GROUP KROZ SQLCMD
-----------------------------------------------------------------------------------------------------------------
ALTER AVAILABILITY GROUP ime MODIFY REPLICA ON 'DMSDB2\DMS' WITH (AVAILABILITY_MODE = SYNCHRONOUS_COMMIT);
go
-----------------------------------------------------------------------------------------------------------------

























-----------------------------------------------------------------------------------------------------------------
USE [master]
RESTORE DATABASE [ReportServer] FROM  DISK = N'C:\UPDATE\BACKUP REPORT\ReportServer.bak' WITH  FILE = 1,  MOVE N'ReportServer' TO N'D:\MSSQL\Data\ReportServer.mdf',  MOVE N'ReportServer_log' TO N'E:\MSSQL\Data\ReportServer_log.ldf',  NOUNLOAD,  STATS = 5, REPLACE
GO
-----------------------------------------------------------------------------------------------------------------
execute msdb..usp_killusers 'test'
EXECUTE sp_restoreDatabase 'test', '\\DIDI195192\DIDI195\SARAJDB\test\FULL', '\\DIDI195192\DIDI195\SARAJDB\test\DIFF'
-----------------------------------------------------------------------------------------------------------------























--permisije za ssrs 
-----------------------------------------------------------------------------------------------------------------
USE master
GO
GRANT EXECUTE ON master.dbo.xp_sqlagent_notify TO RSExecRole
GO
GRANT EXECUTE ON master.dbo.xp_sqlagent_enum_jobs TO RSExecRole
GO
GRANT EXECUTE ON master.dbo.xp_sqlagent_is_starting TO RSExecRole
GO

USE msdb
GO
GRANT EXECUTE ON msdb.dbo.sp_help_category TO RSExecRole
GO
GRANT EXECUTE ON msdb.dbo.sp_add_category TO RSExecRole
GO
GRANT EXECUTE ON msdb.dbo.sp_add_job TO RSExecRole
GO
GRANT EXECUTE ON msdb.dbo.sp_add_jobserver TO RSExecRole
GO
GRANT EXECUTE ON msdb.dbo.sp_add_jobstep TO RSExecRole
GO
GRANT EXECUTE ON msdb.dbo.sp_add_jobschedule TO RSExecRole
GO
GRANT EXECUTE ON msdb.dbo.sp_help_job TO RSExecRole
GO
GRANT EXECUTE ON msdb.dbo.sp_delete_job TO RSExecRole
GO
GRANT EXECUTE ON msdb.dbo.sp_help_jobschedule TO RSExecRole
GO
GRANT EXECUTE ON msdb.dbo.sp_verify_job_identifiers TO RSExecRole
GO
GRANT SELECT ON msdb.dbo.sysjobs TO RSExecRole
GO
GRANT SELECT ON msdb.dbo.syscategories TO RSExecRole
GO
-----------------------------------------------------------------------------------------------------------------















--LAST QUERY CHECK:
-----------------------------------------------------------------------------------------------------------------
SELECT
deqs.last_execution_time AS [Time],
dest.TEXT AS [Query]
FROM
sys.dm_exec_query_stats AS deqs
CROSS APPLY sys.dm_exec_sql_text(deqs.sql_handle) AS dest
ORDER BY
deqs.last_execution_time DESC
-----------------------------------------------------------------------------------------------------------------










--CHECK FOR PERMISIONS DENY
-----------------------------------------------------------------------------------------------------------------
SELECT l.name as grantee_name, p.state_desc, p.permission_name, o.name
FROM sys.database_permissions AS p JOIN sys.database_principals AS l 
ON   p.grantee_principal_id = l.principal_id
JOIN sys.sysobjects O 
ON p.major_id = O.id 
WHERE p.state_desc ='DENY'
-----------------------------------------------------------------------------------------------------------------















--CONFIG ZA OPENQUERY, OPENROWSET
-----------------------------------------------------------------------------------------------------------------
select staro_konto, novo_konto FROM OPENROWSET('SQLNCLI', 'Server=uatsarajdb\sar;Trusted_Connection=yes;', [test].[dbo].[konta_import])
select staro_konto, novo_konto from OPENQUERY([uatsarajdb\sar], 'select staro_konto, novo_konto FROM test.dbo.konta_import')
select TOP 5 * from OPENQUERY(BO, 'SELECT * FROM ISSUING2_0.PRPA_ACCNT_AC FETCH NEXT 10 ROWS ONLY')

SP_CONFIGURE 'advance',1
reconfigure
SP_CONFIGURE 'Ad Hoc Distributed Queries',1
reconfigure

Select * from sys.servers 
EXEC sp_serveroption 'YourServer', 'DATA ACCESS', TRUE
-----------------------------------------------------------------------------------------------------------------
















--BATCH PRIMJER
-----------------------------------------------------------------------------------------------------------------
DECLARE @batch INT,
		@count INT 
SELECT 
@count = COUNT(1),
@batch = 1000
FROM ServiceUsage_T 
WHERE UsageEnd<={ts '2022-12-07 23:59:59'}

WHILE (@count + @batch > @batch)
BEGIN

DELETE TOP (@batch) 
FROM ServiceUsage_T 
WHERE UsageEnd<={ts '2022-12-07 23:59:59'}

SET @count -= @batch
PRINT(@count)
END

/*provjera
select COUNT(1) 
FROM ServiceUsage_T WITH (NOLOCK)
WHERE UsageEnd<={ts '2022-12-07 23:59:59'}
*/
-----------------------------------------------------------------------------------------------------------------

--jos jedan primjer

USE istorija

SET NOCOUNT ON;
DECLARE @rows INT, @count INT, @message VARCHAR(100);
SET @rows = 1;
SET @count = 0;

WHILE @rows > 0
BEGIN
    BEGIN TRAN
	UPDATE TOP(100000) test SET LimitDisable = IIF(Stage='1','Y','N') WHERE COALESCE(LimitDisable,'') = ''
	SET @rows = @@ROWCOUNT
	SET @count = @count + @rows
        RAISERROR('COUNT %d', 0, 1, @count) WITH NOWAIT
    COMMIT TRAN
END

ALTER TABLE PartijeKarticaH DROP COLUMN Stage
GO

-----------------------------------------------------------------------------------------------------------------









--Primjer bacic funkcije
-----------------------------------------------------------------------------------------------------------------
CREATE FUNCTION [dbo].[IsMBOnWhiteListForAccount06]  
(
	@MB varchar(20)
)
RETURNS bit
AS
BEGIN

	DECLARE @ReturnValue as bit

	IF exists(select MatBroj from test (nolock) where MatBroj = @MB and Status = 'A' )
		SET @ReturnValue = 1
	ELSE
		SET @ReturnValue = 0

	RETURN @ReturnValue 

END
-----------------------------------------------------------------------------------------------------------------














--SP_WHO2
-----------------------------------------------------------------------------------------------------------------
CREATE TABLE #sp_who2 (SPID INT, Status VARCHAR(255),
      Login  VARCHAR(255), HostName  VARCHAR(255),
      BlkBy  VARCHAR(255), DBName  VARCHAR(255),
      Command VARCHAR(255), CPUTime INT,
      DiskIO INT, LastBatch VARCHAR(255),
      ProgramName VARCHAR(255), SPID1 INT,
      REQUESTID INT);
INSERT INTO #sp_who2 
EXEC sp_who2

SELECT      *
FROM        #sp_who2
WHERE       DBName = 'test'
ORDER BY    SPID ASC;

DROP TABLE #sp_who2;
-----------------------------------------------------------------------------------------------------------------












--ROLE NA AGENT 
-----------------------------------------------------------------------------------------------------------------
use [msdb]
go

EXEC sp_addrolemember 'SQLAgentOperatorRole', [TODO\dzan.jasarevic]
go
-----------------------------------------------------------------------------------------------------------------














--KAKO PISATI QUERY UPUSTVO
-----------------------------------------------------------------------------------------------------------------
;with imetabele as (
select a.matbroj,a.jmbg,a.konto, a.partija,a.sifval,
            (case when a.sdin<0 then sdin*-1 else 0 end) as dindug, (case when a.sdin>0 then sdin else 0 end) as dinpot,
            (case when a.sdev<0 then sdev*-1 else 0 end) as devdug, (case when a.sdev>0 then sdev else 0 end) as devpot,
            d.novo_novo_konto as NOVO_KONTO, LEFT(d.[Staro_konto],1) as prva_cifra      
        from analstanja a with(nolock)
        inner join #temp2 d on a.konto=d.novo_konto and a.sdin<>0
        where staro_konto NOT like '0%'
        union all   
        select a.matbroj,a.jmbg,a.konto, a.partija,a.sifval,
            (case when sum(dinpot-dindug)<0 then sum(dinpot-dindug)*-1 else 0 end) as dindug,
            (case when sum(dinpot-dindug)>0 then sum(dinpot-dindug) else 0 end) as dinpot,
            (case when sum(devpot-devdug)<0 then sum(devpot-devdug)*-1 else 0 end) as devdug,
            (case when sum(devpot-devdug)>0 then sum(devpot-devdug) else 0 end)  as devpot,
            d.novo_novo_konto as Novo_konto, LEFT(d.[Staro_konto],1) as prva_cifra
        from promkred a with(nolock)
        inner join #temp2 d on a.konto=d.novo_konto 
        group by a.matbroj,a.jmbg,a.konto, a.partija,a.sifval,d.novo_novo_konto, LEFT(d.[Staro_konto],1) 
        having sum(dinpot-dindug)<>0
        union all
        select a.matbroj,a.jmbg,a.konto, a.partija,a.sifval,
            (case when sum(dinpot-dindug)<0 then sum(dinpot-dindug)*-1 else 0 end) as dindug,
            (case when sum(dinpot-dindug)>0 then sum(dinpot-dindug) else 0 end) as dinpot,
            (case when sum(devpot-devdug)<0 then sum(devpot-devdug)*-1 else 0 end) as devdug,
            (case when sum(devpot-devdug)>0 then sum(devpot-devdug) else 0 end)  as devpot,
            d.novo_novo_konto as Novo_konto , LEFT(d.[Staro_konto],1) as prva_cifra
        from promet a with(nolock)
        inner join #temp2 d on a.konto=d.novo_konto 
        group by a.matbroj,a.jmbg,a.konto, a.partija,a.sifval,d.novo_novo_konto, LEFT(d.[Staro_konto],1)
        having sum(dinpot-dindug)<>0
)
select T.matbroj, T.jmbg, T.konto, T.partija, T.sifval, sum(T.dindug) as dindug, sum(T.dinpot) as dinpot, 
    sum(T.devdug) as devdug, sum(T.devpot) as devpot, T.[Novo_konto], T.prva_cifra
        into #Priprema
    from imetabele T
-----------------------------------------------------------------------------------------------------------------


















--UPALITI STATISTIKU ZA QUERY - PROFILE CHECK PROFIL
-----------------------------------------------------------------------------------------------------------------
SET STATISTICS PROFILE ON;  
GO  

--Run this in a different session than the session in which your query is running. 
--Note that you may need to change session id 54 below with the session id you want to monitor.
SELECT node_id,physical_operator_name, SUM(row_count) row_count, 
  SUM(estimate_row_count) AS estimate_row_count, 
  CAST(SUM(row_count)*100 AS float)/SUM(estimate_row_count)  
FROM sys.dm_exec_query_profiles   
WHERE session_id=54
GROUP BY node_id,physical_operator_name  
ORDER BY node_id;  
-----------------------------------------------------------------------------------------------------------------

















--MS AGENT OPERATOR
-----------------------------------------------------------------------------------------------------------------
USE msdb;
DECLARE @DropExistingOperator bit = 0;
DECLARE @OperatorName sysname = N'Amar Abaz';
DECLARE @EmailAddress sysname = N'amar.abaz@TAMBURAgroup.ba';
DECLARE @ReturnCode int = 0;
DECLARE @CategoryID int;
 
BEGIN TRANSACTION;
IF @DropExistingOperator = 1
BEGIN
    EXEC dbo.sp_delete_operator @OperatorName;
END
 
IF NOT EXISTS (
    SELECT 1
    FROM dbo.syscategories sc
    WHERE sc.name = N'DBA'
        AND sc.category_type = 3 --OPERATOR
    )
BEGIN
    EXEC @ReturnCode = dbo.sp_add_category @class = 'OPERATOR'
        , @name = N'DBA'
        , @type = 'NONE';
    IF @ReturnCode = 0
    BEGIN
        PRINT N'Created DBAs operator category';
    END
    ELSE
    BEGIN
        GOTO failed
    END
END
SET @CategoryID = COALESCE(
    (
        SELECT sc.category_id
        FROM dbo.syscategories sc
        WHERE sc.name = N'DBA'
    ), 0);
 
IF NOT EXISTS (
    SELECT 1
    FROM dbo.sysoperators so 
    WHERE so.name = @OperatorName
        AND so.category_id = @CategoryID
    )
BEGIN
    EXEC dbo.sp_add_operator @name = @OperatorName, @enabled = 1, @email_address = @EmailAddress, @category_name = N'DBA';
    PRINT N'Added ' + @OperatorName + N' to the list of operators, with email address: ' + @EmailAddress + N'.';
END
 
GOTO success
failed:
IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
PRINT N'Failed';
GOTO end_of_script
 
success:
PRINT N'Succeeded';
COMMIT TRANSACTION;
SELECT OperatorName = so.name
    , so.email_address
    , CategoryName = sc.name
FROM dbo.sysoperators so
    INNER JOIN dbo.syscategories sc ON so.category_id = sc.category_id
ORDER BY sc.name
    , so.name;
 
end_of_script:
-----------------------------------------------------------------------------------------------------------------























--PROCEDURA ZA PUSTANJE RDL-A KROZ T-SQL BY BENJAMIN
-----------------------------------------------------------------------------------------------------------------

CREATE PROCEDURE [dbo].[starttes_subscription] ( @_ListID INT = null, @_itemNum INT = null)
AS
DECLARE @_updateValue NVARCHAR(MAX);
DECLARE @_path NVARCHAR(MAX),
		@_jobID UNIQUEIDENTIFIER;
SET @_path = '\\benjamin\SHARE';

SELECT TOP 1
@_jobID = sj.job_id
FROM msdb..sysjobs AS sj 
INNER JOIN ReportServer..ReportSchedule AS rs
ON sj.[name] = CAST(rs.ScheduleID AS NVARCHAR(128)) 
INNER JOIN ReportServer..Subscriptions AS su
ON rs.SubscriptionID = su.SubscriptionID
INNER JOIN ReportServer..[Catalog] ct
ON su.Report_OID = c.ItemID
INNER JOIN msdb..sysjobactivity sja
ON sj.job_id = sja.job_id
where c.name = 'AIKSObavijesti'
and LastStatus like 'The file %'
GROUP BY sj.job_id;
IF(@_jobID IS NOT NULL)
EXEC msdb..sp_start_job @job_id = @_jobID;
GO
-----------------------------------------------------------------------------------------------------------------

















-- DODAVANJE CERTIFIKATA NA USER
-----------------------------------------------------------------------------------------------------------------
--adhocdb
USE [master]
GO
CREATE LOGIN [TODO\test] FROM WINDOWS WITH DEFAULT_DATABASE=[master]
ALTER USER [TODO\test] WITH LOGIN [TODO\test]

--sarajdb
CREATE CERTIFICATE [SQL-Compliance-Cert]
ENCRYPTION BY PASSWORD='test'
WITH SUBJECT='Cert to be used for processes that need a higher perms';
GO

ADD SIGNATURE TO OBJECT::[dbo].[test]
BY CERTIFICATE [SQL-Compliance-Cert]
WITH PASSWORD='test';

CREATE USER [test-Izvjestaj-Cert]
FROM CERTIFICATE [SQL-Compliance-Cert]
GO
 
GRANT EXECUTE 
   ON [dbo].[teset]  
   TO [TODO\SQL-Compliance-Analitika-Izvjestaj] 
GO  
--------------------------------
CREATE USER [TODO\SQL-Compliance-Analitika-Izvjestaj] 
GRANT EXEC ON [dbo].[AnalKartica_stanja_AH] TO [TODO\SQL-Compliance-Analitika-Izvjestaj]; 
ALTER ROLE [db_datareader] ADD MEMBER [TODO\SQL-Compliance-Analitika-Izvjestaj]
GO
-----------------------------------------------------------------------------------------------------------------
-----------------------------------------------------------------------------------------------------------------
-----------------------------------------------------------------------------------------------------------------
--primjer 2 koji sam radio za reporte alisi
--sve ovo da bi radilo pokretanje joba kroz proceduru u njihovoj bazi 
--execute [FinRep_db].[dbo].[create_report_depozit_klijenata]

--target
CREATE CERTIFICATE [SQL-Execute-Cert]
ENCRYPTION BY PASSWORD='tets'
WITH SUBJECT='Cert to be used for processes that need a higher perms';
GO

ADD SIGNATURE TO OBJECT::[dbo].[create_report_depozit_klijenata]
BY CERTIFICATE [SQL-Execute-Cert]
WITH PASSWORD='test';

CREATE USER [SQL-create_report]
FROM CERTIFICATE [SQL-Execute-Cert]
GO
 
 use FinRep_db
 go

GRANT EXECUTE 
   ON [dbo].[create_report_depozit_klijenata]  
   TO [TODO\test] 
GO  

BACKUP CERTIFICATE [SQL-Execute-Cert]
TO FILE = 'c:\UPDATE\SQL-Execute-Cert.cer';
GO


--source
USE [msdb]
GO
CREATE CERTIFICATE [SQL-Execute-Cert]
FROM FILE = 'c:\UPDATE\SQL-Execute-Cert.cer';
GO
-- Create a login from the certificate
CREATE LOGIN [SQL-create_report]
FROM CERTIFICATE [SQL-Execute-Cert];
GO
grant execute on sp_start_job to [SQL-create_report]
 
--master
USE [master]
GO
CREATE CERTIFICATE [SQL-Execute-Cert]
FROM FILE = 'c:\UPDATE\SQL-Execute-Cert.cer';
GO
CREATE LOGIN [SQL-create_report]
FROM CERTIFICATE [SQL-Execute-Cert]
GO

GRANT AUTHENTICATE SERVER TO [SQL-create_report]
GO 


--Sa ovim dodajem na druge baze isti certifikat:
CREATE CERTIFICATE [SQL-Execute-Cert]       
FROM FILE = 'c:\UPDATE\SQL-Execute-Cert.cer'   
WITH PRIVATE KEY (
DECRYPTION BY PASSWORD = 'test')


DROP SIGNATURE from OBJECT::[dbo].[lock] BY CERTIFICATE [SQL-Execute-Cert]

ADD SIGNATURE TO OBJECT::[dbo].[lock]
BY CERTIFICATE [SQL-Execute-Cert]
WITH SIGNATURE = 0x0100070204000000B83E7267F5F600C2029412C1D4375CDE667B83335818A0E723D7D833B443175DE75CF7CB22140B8AE38817D82CE2339590A7A542E3ACAB440CAB36870F1522257F19A5CD223CC163074718C3FAEE330A1E89024DB8231D1AE1279FB756D0330370E479C129FA995D44F2301F54D11DB0C6A6F081E7D118556B915B651EE06AC0A711ADC3D292E367A2D60568B226BC5FADC25C4C4002A37878C9A36256EC905304ED955D7A9A5A8E8C043986C14013CA88DD558F036231144836652810DDD52F8904401C90F9F19BD189A7A906B0A05B7CEEF24BE5A24C0A53A2C3753A52FFA99B76B1E8D15BA152D3FF08070FBEF13B770557D5A76515EB2313711F4BEF8379B961F562672021B8D6363B082353D21B82E2EE98BFE1760E9601B219CBA3617EF2316B9B0A17FBBF75B2BD8E4EFBB662E63EE71007DFCEA66722C1EC9D2005D14D067AC668FB5B875A2CE5184AEC96C0F1109ED9E02B7D30C36A3C185ED7756530D6533649E3F7A91480AE4B9040B5570D3B5C0C61F425F3022E5A8DEDD88B83

CREATE USER [SQL-certUser]
FROM CERTIFICATE [SQL-Execute-Cert]
GO
 
 SELECT cp.crypt_property  
    FROM sys.crypt_properties AS cp  
    JOIN sys.certificates AS cer  
        ON cp.thumbprint = cer.thumbprint  
    WHERE cer.name = 'SQL-Execute-Cert' ;  
GO  


-----------------------------------------------------------------------------------------------------------------
1.
USE [Statements]

--create i backup certifikata
CREATE CERTIFICATE [SQL-Execute-Cert]
ENCRYPTION BY PASSWORD='twaws'
WITH SUBJECT='Cert to be used for processes that need a higher perms';
GO
BACKUP CERTIFICATE [SQL-Execute-Cert]
TO FILE = 'd:\MSSQL\SQL-Execute-Cert.cer';
GO
-- potpis na njihovu proceduru 
ADD SIGNATURE TO OBJECT::[dbo].[executeStatementSSIS]
BY CERTIFICATE [SQL-Execute-Cert]
WITH PASSWORD='********************';

2.
USE [msdb]
GO
CREATE CERTIFICATE [SQL-Execute-Cert]
FROM FILE = 'd:\MSSQL\SQL-Execute-Cert.cer';
GO
CREATE USER [sqlCert]
FROM CERTIFICATE [SQL-Execute-Cert];
GO
grant execute on sp_start_job to [sqlCert] 

3.
USE [master]
GO
CREATE CERTIFICATE [SQL-Execute-Cert]
FROM FILE = 'd:\MSSQL\SQL-Execute-Cert.cer';
GO
CREATE LOGIN [sqlCert]
FROM CERTIFICATE [SQL-Execute-Cert]
GO
GRANT AUTHENTICATE SERVER TO [sqlCert]
GO
-----------------------------------------------------------------------------------------------------------------













CREATE EVENT SESSION [errors_on_Corporate_Security_Mailbox_BH] ON SERVER 
ADD EVENT sqlserver.error_reported(
    ACTION(sqlserver.client_app_name,sqlserver.client_hostname,sqlserver.database_id,sqlserver.database_name,sqlserver.sql_text)
    WHERE ([package0].[greater_than_int64]([severity],(10)) AND ([sqlserver].[equal_i_sql_unicode_string]([sqlserver].[database_name],N'iBankingCorporate') OR [sqlserver].[equal_i_sql_unicode_string]([sqlserver].[database_name],N'SecurityServerCorporate') OR [sqlserver].[equal_i_sql_unicode_string]([sqlserver].[database_name],N'Mailbox')) AND [error_number]<>(50000)))
ADD TARGET package0.event_file(SET filename=N'D:\rdsdbdata\Log\tester',max_file_size=(32))
WITH (MAX_MEMORY=4096 KB,EVENT_RETENTION_MODE=ALLOW_SINGLE_EVENT_LOSS,MAX_DISPATCH_LATENCY=30 SECONDS,MAX_EVENT_SIZE=0 KB,MEMORY_PARTITION_MODE=NONE,TRACK_CAUSALITY=ON,STARTUP_STATE=OFF)
GO









@with_norecovery=0|1, [@type='DIFFERENTIAL|FULL'];



EXEC msdb..sp_delete_job @job_name = '$DBA - Maintenance - INDEX OPTIMIZE - EveryDay';

-- AWS AMAZON SERVICES
-----------------------------------------------------------------------------------------------------------------

--AWS COMMANDS
--restore from bucket
exec msdb.dbo.rds_restore_database
@restore_db_name='rbbh_geolocations',
@s3_arn_to_restore_from='arn:aws:s3:::rbbh-gl-test-restore-backup/rbbh_geolocations_test.bak';

exec msdb.dbo.rds_restore_database
 @restore_db_name='financeforecast2',
 @s3_arn_to_restore_from='arn:aws:s3:::cmap-rbal-forecast/FinanceForecast2_onprem.bak';
 
 
 exec msdb.dbo.rds_restore_database
@restore_db_name='AdventureWorksDW2019',
@s3_arn_to_restore_from='arn:aws:s3:::rds-sql-server-pitr-bucket/AdventureWorksDW2019.bak';

------------------------------------------------
--zadnji 
exec msdb.dbo.rds_restore_database
 @restore_db_name='GeoLocations',
 @s3_arn_to_restore_from='arn:aws:s3:::rbbh-geolocations-test-restore-backup/GeoLocations_20231702.bak';


--check status
exec msdb..rds_task_status

--compress backups for later
exec rdsadmin..rds_set_configuration 'S3 backup compression', 'true'

--drop database
EXECUTE msdb.dbo.rds_drop_database  N'your-database-name'


-----------------------------------------------------------------------------------------------------------------
--FULL backup
exec msdb.dbo.rds_backup_database 
@source_db_name='GeoLocations', @s3_arn_to_backup_to='arn:aws:s3:::rbbh-geolocations-test-restore-backup/rbbh_geolocations_test_fin.bak', 
@overwrite_S3_backup_file=0;

--DIFF backup
exec msdb.dbo.rds_backup_database 
@source_db_name='GeoLocations', @s3_arn_to_backup_to='arn:aws:s3:::rbbh-geolocations-test-restore-backup/rbbh_geolocations_test_diff.bak', 
@overwrite_S3_backup_file=0,
@type='DIFFERENTIAL';

--provjera stanja
exec msdb.dbo.rds_task_status @task_id=8;
-------------------------------------------------------------------
--dodaj putanju za log
exec msdb.dbo.rds_tlog_copy_setup @target_s3_arn = 'arn:aws:s3:::rbbh-geolocations-test-restore-backup/LOG/'
--provjeri putanju da se primjenila
exec rdsadmin.[dbo].[rds_show_configuration] @name='target_s3_arn_for_tlog_copy'

--Provjera logova
SELECT * from msdb.dbo.rds_fn_list_tlog_backup_metadata('GeoLocations')
ORDER BY rds_backup_seq_id

--Log backup
exec msdb.dbo.rds_tlog_backup_copy_to_S3
@db_name = 'GeoLocations',
@backup_file_start_time='2023-04-5 11:59:00',
@backup_file_end_time='2023-04-5 12:01:00'
,@kms_key_arn='arn:aws:kms:eu-central-1:007982745556:key/099ceba0-9ef6-428c-b649-e6fb749873bb'

--Provjera statusa log backupa
exec msdb.dbo.rds_task_status @task_id=10;


 SELECT * from msdb.dbo.rds_fn_list_tlog_backup_metadata('ebledb');
/*
exec msdb.dbo.rds_backup_database 
@source_db_name='ebledb', @s3_arn_to_backup_to='arn:aws:s3:::rbbhdba-backups/ebledb_full.bak', 
@overwrite_S3_backup_file=0;

exec msdb.dbo.rds_backup_database 
@source_db_name='ebledb', @s3_arn_to_backup_to='arn:aws:s3:::rbbhdba-backups/ebledb_diff3.bak', 
@overwrite_S3_backup_file=0,
@type='DIFFERENTIAL';

exec msdb.dbo.rds_tlog_backup_copy_to_S3
@db_name = 'ebledb',
@backup_file_start_time='2023-11-20 15:14:02',
@backup_file_end_time='2023-11-20 15:44:02'


{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Only allow writes to my bucket with bucket owner full control",
            "Effect": "Allow",
            "Principal": {
                "Service": "backups.rds.amazonaws.com"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::rbbhdba-backups/LOG_BACKUP_RDP/*",
            "Condition": {
                "StringEquals": {
                    "aws:sourceArn": "arn:aws:rds:eu-central-1:765217435168:db:rbbh-ebledb-test",
                    "aws:sourceAccount": "765217435168",
                    "s3:x-amz-acl": "bucket-owner-full-control"
                }
            }
        }
    ]
}


*/




















--SPN
-----------------------------------------------------------------------------------------------------------------
/*
setspn -L TODO\mssqlserviceuat
setspn -a MSSQLSvc/hrdbtest:1433
klist
*/
-----------------------------------------------------------------------------------------------------------------




--POWERSHELL COPY PRAZNE FOLDERE
/*
$from = "C:\Temp"
$to = "C:\Temp3"

Get-ChildItem -Path $from -Recurse | 
?{ $_.PSIsContainer } | 
Copy-Item -Destination {Join-Path $to $_.Parent.FullName.Substring($from.length)} -Force 
*/








--SQL EXPORT SSIS PAKETE POWERSHELL
-----------------------------------------------------------------------------------------------------------------
/*
Param($SQLInstance = "test") 
add-pssnapin sqlserverprovidersnapin100 -ErrorAction SilentlyContinue
add-pssnapin sqlservercmdletsnapin100 -ErrorAction SilentlyContinue
cls
$Packages =  Invoke-Sqlcmd -MaxCharLength 10000000 -ServerInstance $SQLInstance -Query "select p.name, CAST(CAST(packagedata AS VARBINARY(MAX)) AS VARCHAR(MAX)) as pkg
FROM MSDB..sysssispackages p join
msdb..sysssispackagefolders f on p.folderid = f.folderid
where f.foldername NOT LIKE 'Data Collector%'"
Foreach ($pkg in $Packages)
{
    $pkgName = $Pkg.name
    $fullfolderPath = "C:\ssis\IS_Server2\$SQLInstance\$pkgName\"
    if(!(test-path -path $fullfolderPath))
    {
        mkdir $fullfolderPath | Out-Null
    }
    $pkg.pkg | Out-File -Force -encoding ascii -FilePath "$fullfolderPath\$pkgName.dtsx"
}
*/
-----------------------------------------------------------------------------------------------------------------







--SQL EXPORT SSRS PAKETE POWERSHELL
-----------------------------------------------------------------------------------------------------------------
/*
###################################################################################
# Download Reports and DataSources from a SSRS server and create the same folder
# structure in the local download folder.
###################################################################################
# Parameters
###################################################################################
$downloadFolder = "c:\temp\ssrs\"
$ssrsServer = "http://myssrs.westeurope.cloudapp.azure.com"
###################################################################################
# If you can't use integrated security
#$secpasswd = ConvertTo-SecureString "MyPassword!" -AsPlainText -Force
#$mycreds = New-Object System.Management.Automation.PSCredential ("MyUser", $secpasswd)
#$ssrsProxy = New-WebServiceProxy -Uri "$($ssrsServer)/ReportServer/ReportService2010.asmx?WSDL" -Credential $mycreds
 
# SSRS Webserver call
$ssrsProxy = New-WebServiceProxy -Uri "$($ssrsServer)/ReportServer/ReportService2010.asmx?WSDL" -UseDefaultCredential
 
# List everything on the Report Server, recursively, but filter to keep Reports and DataSources
$ssrsItems = $ssrsProxy.ListChildren("/", $true) | Where-Object {$_.TypeName -eq "DataSource" -or $_.TypeName -eq "Report"}
 
# Loop through reports and data sources
Foreach($ssrsItem in $ssrsItems)
{
    # Determine extension for Reports and DataSources
    if ($ssrsItem.TypeName -eq "Report")
    {
        $extension = ".rdl"
    }
    else
    {
        $extension = ".rds"
    }
     
    # Write path to screen for debug purposes
    Write-Host "Downloading $($ssrsItem.Path)$($extension)";
 
    # Create download folder if it doesn't exist (concatenate: "c:\temp\ssrs\" and "/SSRSFolder/")
    $downloadFolderSub = $downloadFolder.Trim('\') + $ssrsItem.Path.Replace($ssrsItem.Name,"").Replace("/","\").Trim() 
    New-Item -ItemType Directory -Path $downloadFolderSub -Force > $null
 
    # Get SSRS file bytes in a variable
    $ssrsFile = New-Object System.Xml.XmlDocument
    [byte[]] $ssrsDefinition = $null
    $ssrsDefinition = $ssrsProxy.GetItemDefinition($ssrsItem.Path)
 
    # Download the actual bytes
    [System.IO.MemoryStream] $memoryStream = New-Object System.IO.MemoryStream(@(,$ssrsDefinition))
    $ssrsFile.Load($memoryStream)
    $fullDataSourceFileName = $downloadFolderSub + "\" + $ssrsItem.Name +  $extension;
    $ssrsFile.Save($fullDataSourceFileName);
}
*/
-----------------------------------------------------------------------------------------------------------------










--POWERSHELL AD komande
-----------------------------------------------------------------------------------------------------------------
Get-ADGroupMember -identity "IT-T24-Migration" | Select SamAccountName
Get-ADGroupMember -identity "HR Full" | select name | Export-csv -path c:\it\filename.csv -Notypeinformation
-----------------------------------------------------------------------------------------------------------------






-- SSIS INTEGRATION PAKET OD DO PROVJERA MAX
-----------------------------------------------------------------------------------------------------------------
SELECT MAX(idH) idH from dbo.testH
SELECT * 
FROM dbo.testH 
WHERE idH > ?
-----------------------------------------------------------------------------------------------------------------











--KILL USER PROCEDURA
-----------------------------------------------------------------------------------------------------------------
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE PROCEDURE [dbo].[usp_KillUsers] @dbname varchar(50) as
SET NOCOUNT ON
DECLARE @strSQL varchar(255)

CREATE table #tmpUsers(
 spid int,
 ecid int,
 status varchar(30),
 loginname varchar(50),
 hostname varchar(50),
 blk int,
 dbname varchar(50),
 cmd varchar(30),
  request_id int)
INSERT INTO #tmpUsers EXEC SP_WHO
DECLARE LoginCursor CURSOR
READ_ONLY
FOR 
SELECT spid, dbname,loginname FROM #tmpUsers WHERE dbname = @dbname
and loginname not in ('TODO\MSSQLServiceAdmin','TODO\Administrator','sa')
DECLARE @spid varchar(10)
DECLARE @dbname2 varchar(40)
DECLARE @loginname varchar(50)
OPEN LoginCursor
FETCH NEXT FROM LoginCursor INTO @spid, @dbname2,@loginname
WHILE (@@fetch_status <> -1)
BEGIN
 IF (@@fetch_status <> -2)
 BEGIN
 SET @strSQL = 'KILL ' + @spid
 print @strSQL
 EXEC (@strSQL)

 END
 FETCH NEXT FROM LoginCursor INTO  @spid, @dbname2,@loginname
END
CLOSE LoginCursor
DEALLOCATE LoginCursor
DROP table #tmpUsers
-----------------------------------------------------------------------------------------------------------------






















--ENKRIPCIJA BAZA
-----------------------------------------------------------------------------------------------------------------
USE Master;
GO
CREATE MASTER KEY ENCRYPTION
BY PASSWORD='InsertStrongPasswordHere';
GO

CREATE CERTIFICATE TDE_Cert
WITH 
SUBJECT='Database_Encryption';
GO

USE <bazu koju encypt>
GO
CREATE DATABASE ENCRYPTION KEY
WITH ALGORITHM = AES_256
ENCRYPTION BY SERVER CERTIFICATE TDE_Cert;
GO

ALTER DATABASE <DB>
SET ENCRYPTION ON;
GO

-----------------------------------backup i restore
BACKUP CERTIFICATE TDE_Cert
TO FILE = 'C:\temp\TDE_Cert'
WITH PRIVATE KEY (file='C:\temp\TDE_CertKey.pvk',
ENCRYPTION BY PASSWORD='InsertStrongPasswordHere') 

USE Master;
GO
CREATE MASTER KEY ENCRYPTION
BY PASSWORD='InsertStrongPasswordHere';
GO

USE MASTER
GO
CREATE CERTIFICATE TDECert
FROM FILE = 'C:\Temp\TDE_Cert'
WITH PRIVATE KEY (FILE = 'C:\TDECert_Key.pvk',
DECRYPTION BY PASSWORD = 'InsertStrongPasswordHere' );
-----------------------------------------------------------------------------------------------------------------


-- ALWAYS ENCRYPT COLUMN
-----------------------------------------------------------------------------------------------------------------
/*
COMPUTER\myIISPoolUser
Set-Location -Path cert:\localMachine\my
Import-PfxCertificate –FilePath c:\AlwaysEncrypt.pfx

COMPUTER\myIISPoolUser
certmgr.msc
*/
-----------------------------------------------------------------------------------------------------------------









--MASKIRANJE
-----------------------------------------------------------------------------------------------------------------
alter table [dbo].[tester3]
ALTER COLUMN datum_pohrane ADD MASKED WITH (FUNCTION = 'partial(2,"XXXXXXX",1)')

GRANT SELECT ON tester3 TO testu1;
EXECUTE AS USER = 'testu1';
SELECT * FROM [dbo].[tester3]
REVERT;
revoke UNMASK TO testu1
-----------------------------------------------------------------------------------------------------------------











-- INSERT KROZ SSIS PAKET PRIMJER SA VARIABLE
-----------------------------------------------------------------------------------------------------------------
"INSERT INTO [DBRANCH_log] VALUES 
 ('"+@[User::Variabler]+"',convert(varchar(10), GETDATE(),126), 'ActionLog', '"+(DT_WSTR,12)@[User::table1]+"')
 ,('"+@[User::Variabler]+"',convert(varchar(10), GETDATE(),126), 'ActionLogMapper', '"+(DT_WSTR,12)@[User::table2]+"')
 ,('"+@[User::Variabler]+"',convert(varchar(10), GETDATE(),126), 'TransactionState', '"+(DT_WSTR,12)@[User::table3]+"')
 ,('"+@[User::Variabler]+"',convert(varchar(10), GETDATE(),126), 'Transactions', '"+(DT_WSTR,12)@[User::table4]+"')
 ,('"+@[User::Variabler]+"',convert(varchar(10), GETDATE(),126), 'ExternalActionLog', '"+(DT_WSTR,12)@[User::table5]+"')
 ,('"+@[User::Variabler]+"',convert(varchar(10), GETDATE(),126), 'TransactionDenominations', '"+(DT_WSTR,12)@[User::table6]+"');"
-----------------------------------------------------------------------------------------------------------------
















--POWERSHELL SKRIPTA ZA USER PERMISIJE PERMISIONS
-----------------------------------------------------------------------------------------------------------------
/*
# ========================================================================================================
 Function GetDBUserInfo($Dbase)
 {
  if ($dbase.status -eq "Normal")  # ensures the DB is online before checking
   {$users = $Dbase.users | where {$_.login -eq $SQLLogin.name}   # Ignore the account running this as it is assumed to be an admin account on all servers
     foreach ($u in $users)
     {
       if ($u)
         {
           $DBRoles = $u.enumroles()
           foreach ($role in $DBRoles) {
           if ($role -eq "db_owner") {
           write-host $role "on"$Dbase.name -foregroundcolor "red"  #if db_owner set text color to red
           }
           else {
           write-host $role "on"$Dbase.name
           }
           }
         #Get any explicitily granted permissions
         foreach($perm in $Dbase.EnumObjectPermissions($u.Name)){
         write-host  $perm.permissionstate $perm.permissiontype "on" $perm.objectname "in" $DBase.name }
      }
    } # Next user in database
   }
   #else
   #Skip to next database.
 }
 #Main portion of script start
 [reflection.assembly]::LoadWithPartialName("Microsoft.SqlServer.Smo") | out-null  #ensure we have SQL SMO available
 foreach ($SQLsvr in get-content "C:\temp\Instances.txt")  # read the instance source file to get instance names
 {
   $svr = new-object ("Microsoft.SqlServer.Management.Smo.Server") $SQLsvr
   write-host "================================================================================="
   write-host "SQL Instance: " $svr.name
   write-host "SQL Version:" $svr.VersionString
   write-host "Edition:" $svr.Edition
   write-host "Login Mode:" $svr.LoginMode
   write-host "================================================================================="
   $SQLLogins = $svr.logins
   foreach ($SQLLogin in $SQLLogins)
   {
     write-host    "Login          : " $SQLLogin.name
     write-host    "Login Type     : " $SQLLogin.LoginType
     write-host    "Created        : " $SQLLogin.CreateDate
     write-host    "Default DB     : " $SQLLogin.DefaultDatabase
     Write-Host    "Disabled       : " $SQLLogin.IsDisabled
     $SQLRoles = $SQLLogin.ListMembers()
     if ($SQLRoles) {
     if ($SQLRoles -eq "SysAdmin"){ write-host    "Server Role    : " $SQLRoles -foregroundcolor "red"}
     else { write-host    "Server Role    : " $SQLRoles
     } } else {"Server Role    :  Public"}
      If ( $SQLLogin.LoginType -eq "WindowsGroup" ) {   #get individuals in any Windows domain groups
            write-host "Group Members:"
         try {
                $ADGRoupMembers = get-adgroupmember  $SQLLogin.name.Split("\")[1] -Recursive
                foreach($member in $ADGRoupMembers){
                write-host "   Account: " $member.name "("$member.SamAccountName")"
                                                    }
            }
            catch
            {
            #Sometimes there are 'ghost' groups left behind that are no longer in the domain, this highlights those still in SQL
                write-host "Unable to locate group "  $SQLLogin.name.Split("\")[1] " in the AD Domain" -foregroundcolor Red
            }
            }
        #Check the permissions in the DBs the Login is linked to.
     if ($SQLLogin.EnumDatabaseMappings())
       {write-host "Permissions:"
       foreach ( $DB in $svr.Databases)
         {
         GetDBUserInfo($DB)
         } # Next Database
       }
     Else
       {write-host "None."
       }
     write-host "   ----------------------------------------------------------------------------"
   } # Next Login
 } # Next Server
 */
-----------------------------------------------------------------------------------------------------------------













--DODAVANJE USERA I PERMISIJA ZA SERVISNE USERE
-----------------------------------------------------------------------------------------------------------------
/*
Install-WindowsFeature AD-DOMAIN-SERVICES
Install-ADServiceAccount -Identity MSSQL-T24U-SVC
Get-ADServiceAccount -Identity MSSQL-T24U-SVC$ -Properties TrustedForDelegation,TrustedToAuthForDelegation,msDS-AllowedToDelegateTo
Set-ADAccountControl -Identity MSSQL-T24D-SVC$ -TrustedForDelegation $true -TrustedToAuthForDelegation $true
*/
-----------------------------------------------------------------------------------------------------------------











--PROVJERA JOB SQL AGENT DA LI JE ACTIVE
-----------------------------------------------------------------------------------------------------------------
IF EXISTS(SELECT 1 
          FROM msdb.dbo.sysjobs J 
          JOIN msdb.dbo.sysjobactivity A 
              ON A.job_id=J.job_id 
          WHERE J.name=N'1' 
          AND A.run_requested_date IS NOT NULL 
          AND A.stop_execution_date IS NULL
         )
    PRINT 'The job is running!'
	ELSE
    PRINT 'The job is not running.'
-----------------------------------------------------------------------------------------------------------------















--PROVJERA GROWTH BAZA PO GODINAMA
-----------------------------------------------------------------------------------------------------------------
--SECTION 1 BEGIN
WITH BackupsSize AS(
SELECT TOP 1000
      rn = ROW_NUMBER() OVER (ORDER BY DATEPART(year,[backup_start_date]) ASC, DATEPART(month,[backup_start_date]) ASC)
    , [Year]  = DATEPART(year,[backup_start_date])
    , [Month] = DATEPART(month,[backup_start_date])
    , [Backup Size GB] = CONVERT(DECIMAL(10,2),ROUND(AVG([backup_size]/1024/1024/1024),4))
    , [Compressed Backup Size GB] = CONVERT(DECIMAL(10,2),ROUND(AVG([compressed_backup_size]/1024/1024/1024),4))
FROM 
    msdb.dbo.backupset
WHERE 
    [database_name] = N'rbbh_geolocations'
AND [type] = 'D'
AND backup_start_date BETWEEN DATEADD(mm, - 13, GETDATE()) AND GETDATE()
GROUP BY 
    [database_name]
    , DATEPART(yyyy,[backup_start_date])
    , DATEPART(mm, [backup_start_date])
ORDER BY [Year],[Month]) 
--SECTION 1 END
 
--SECTION 2 BEGIN
SELECT 
   b.Year,
   b.Month,
   b.[Backup Size GB],
   0 AS deltaNormal,
   b.[Compressed Backup Size GB],
   0 AS deltaCompressed
FROM BackupsSize b
WHERE b.rn = 1
UNION
SELECT 
   b.Year,
   b.Month,
   b.[Backup Size GB],
   b.[Backup Size GB] - d.[Backup Size GB] AS deltaNormal,
   b.[Compressed Backup Size GB],
   b.[Compressed Backup Size GB] - d.[Compressed Backup Size GB] AS deltaCompressed
FROM BackupsSize b
CROSS APPLY (
   SELECT bs.[Backup Size GB],bs.[Compressed Backup Size GB]
   FROM BackupsSize bs
   WHERE bs.rn = b.rn - 1
) AS d
--SECTION 2 END
-----------------------------------------------------------------------------------------------------------------
















-----------------------------------------------------------------------------------------------------------------
--PROVJERA JOBOVA JOB KOJI SU ENABLE/DISABLE, SKRIPTA SA PRINT
use [msdb]
DECLARE @name nvarchar (256)
DECLARE @status nvarchar (256)
DECLARE @job_id nvarchar (256)

DECLARE cur CURSOR FOR

select name, [enabled] from msdb..sysjobs order by name
print 'USE [msdb]';
print 'GO';

	OPEN cur

	FETCH NEXT FROM cur INTO @name, @status;
	WHILE @@FETCH_STATUS = 0
	BEGIN
		print 'exec dbo.sp_update_job @job_name = N''' + @name + ''', @enabled = 0;';
		print 'GO'
		FETCH NEXT FROM cur INTO @name, @status;
	END

	CLOSE cur;
	DEALLOCATE cur;
---------------
  SELECT
     job.job_id,
     notify_level_email,
     name,
     enabled,
     description,
     step_name,
     command,
     server,
     database_name
FROM
    msdb.dbo.sysjobs job
INNER JOIN 
    msdb.dbo.sysjobsteps steps        
ON
    job.job_id = steps.job_id
WHERE
    job.enabled = 0
-----------------------------------------------------------------------------------------------------------------













--ON DEMAND BACKUP
-----------------------------------------------------------------------------------------------------------------
--
use msdb
execute dbo.AdhocBackup
--
create procedure dbo.AdhocBackup
with execute as owner
as
exec sp_start_job @job_name = '1'
--
GRANT EXECUTE on dbo.AdhocBackup TO [testu1]
--
-----------------------------------------------------------------------------------------------------------------























-----------------------------------------------------------------------------------------------------------------
--PROCEDURA ZA RESTORE DATABASE
-----------------------------------------------------------------------------------------------------------------
USE [dba]
GO

/****** Object:  StoredProcedure [dbo].[restoreDatabase]    Script Date: 3/22/2023 1:45:59 PM ******/
SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO






/*
EXECUTE dbo.restoreDatabase 
@serverName = 'sarajdb',
@databaseName = 'test',
@continueLogRestore = 1,
@restoreDatetime = '2023-02-06 7:00:00',
@recovery = 0,
@databaseReadonly = 0,
@print = 1;

EXECUTE dbo.restoreDatabase 
@serverName = 'bosnadb\head',
@databaseName = 'GRAS',
@continueLogRestore = 1,
@restoreDatetime = '2023-02-06 7:00:00',
@recovery = 0,
@databaseReadonly = 0,
@print = 1;

EXECUTE dbo.restoreDatabase 
@serverName = 'pisardb\code',
@databaseName = 'stuff',
@continueLogRestore = 1,
@restoreDatetime = '2023-02-06 7:00:00',
@recovery = 0,
@databaseReadonly = 0,
@print = 1;

EXECUTE dbo.restoreDatabase 
@serverName = 'sebr\retail',
@databaseName = 'apolo',
@continueLogRestore = 1,
@restoreDatetime = '2023-02-06 7:00:00',
@recovery = 0,
@databaseReadonly = 0,
@print = 1;
*/


CREATE PROCEDURE [dbo].[restoreDatabase]
@serverName NVARCHAR(100), 
@databaseName NVARCHAR(100), 
@continueLogRestore BIT = 0,
@restoreDatetime NVARCHAR(255) = NULL, 
@recovery BIT = 1,
@databaseReadonly BIT = 0,
@print BIT = 1
AS

DECLARE @_tempTable_restoreDataset TABLE 
(
backupType NVARCHAR(100),
restorePath NVARCHAR(MAX),
backupDatetime DATETIME2
);

DECLARE @_forExecution NVARCHAR(MAX),
		@_getDataset NVARCHAR(MAX),
		@_createLinkedserver NVARCHAR(MAX),
		@_dropLinkedserver NVARCHAR(MAX),
		@_linkedServer NVARCHAR(100),
		@_restoreFull NVARCHAR(MAX),
		@_restoreFullpath NVARCHAR(MAX),
		@_restoreDiff NVARCHAR(MAX),
		@_restoreDiffpath NVARCHAR(MAX),
		@_restoreLog NVARCHAR(MAX),
		@_restoreLogpath NVARCHAR(500),
		@_databaseName NVARCHAR(100),
		@_recoverDatabase NVARCHAR(MAX),
		@_setReadonly NVARCHAR(MAX),
		@_setRecoverytype NVARCHAR(MAX),
		@_restoreDatetimeFrom NVARCHAR(255),
		@_spid INT;

IF(@restoreDatetime IS NULL)
	SET @restoreDatetime = GETDATE();

SELECT @_restoreDatetimeFrom =
CONVERT(NVARCHAR, MAX(bus.backup_finish_date), 120)
FROM sys.sysdatabases sdb
LEFT OUTER JOIN msdb.dbo.backupset bus ON bus.database_name = sdb.name
WHERE sdb.name = @databaseName
GROUP BY sdb.Name

SET @_getDataset = '
SELECT 
 CASE s.[type]    
           WHEN ''D'' THEN ''FULL''    
           WHEN ''I'' THEN ''DIFF''    
           WHEN ''L'' THEN ''LOG'' 
       END AS BackupType,    
       m.physical_device_name AS BackupPath ,   
	convert(varchar,backup_finish_date, 120)  BackupDate  
FROM msdb.dbo.backupset s    
INNER JOIN msdb.dbo.backupmediafamily m ON s.media_set_id = m.media_set_id    
WHERE (s.backup_set_id IN    
         (SELECT Max(backup_set_id)    
          FROM msdb.dbo.backupset    
          WHERE TYPE = ''D''  
            AND database_name = ''@databaseName''    
            AND backup_finish_date <= ''@restoreDatetime'')    
       OR s.backup_set_id IN    
         (SELECT MAX(backup_set_id)    
          FROM msdb.dbo.backupset    
          WHERE TYPE = ''I''    
            AND database_name = ''@databaseName''  
            AND backup_finish_date >=    
              (SELECT Max(backup_finish_date)    
               FROM msdb.dbo.backupset    
               WHERE TYPE = ''D''   
                 AND database_name = ''@databaseName''    
                 AND backup_finish_date <= ''@restoreDatetime'')    
				 AND backup_finish_date <= ''@restoreDatetime'')    
       OR s.backup_set_id IN    
         (SELECT backup_set_id    
          FROM msdb.dbo.backupset    
          WHERE TYPE = ''L''  
            AND database_name = ''@databaseName''    
            AND backup_finish_date >=    
              (SELECT Max(backup_finish_date)    
               FROM msdb.dbo.backupset    
               WHERE (TYPE = ''I'' OR TYPE = ''D'')   
                 AND database_name = ''@databaseName''    
                 AND backup_finish_date <= ''@restoreDatetime'')    
				 AND backup_finish_date <= ''@restoreDatetime'')   
			 )    
  AND s.backup_finish_date <= ''@restoreDatetime''  AND backup_finish_date >= (SELECT Max(backup_finish_date)    
               FROM msdb.dbo.backupset    
               WHERE TYPE = ''D''   
                 AND database_name = ''@databaseName''   
                 AND backup_finish_date <= ''@restoreDatetime'')   
				 @restoreDatetimeFrom
  ORDER  BY s.backup_set_id;';


SELECT @_restoreDatetimeFrom = CASE 
	WHEN @continueLogRestore = 0 THEN ''
	WHEN @continueLogRestore = 1 THEN ' AND backup_finish_date > '''+@_restoreDatetimeFrom+''''
END


SET @_getDataset = REPLACE(@_getDataSet,'@restoreDatetimeFrom',@_restoreDatetimeFrom);
SET @_getDataset = REPLACE(@_getDataSet,'@restoreDatetime',@restoreDatetime);
SET @_getDataset = REPLACE(@_getDataSet,'@databaseName',@databaseName);

SET @_linkedServer = @serverName+'_temp_'+CONVERT(NVARCHAR,CAST(RAND() * 1000000 AS INT));
SET @_createLinkedserver = 
'
sp_addlinkedserver 
@server = '''+@_linkedServer+''', 
@srvproduct = '''',
@datasrc= '''+@serverName+''', 
@provider =  ''SQLNCLI''

EXEC sp_serveroption '''+@_linkedServer+''', ''rpc out'', true;  
';

SET @_dropLinkedserver =
'
EXEC sp_dropserver '''+@_linkedServer+''', ''droplogins'';
';

SET @_forExecution = '
EXECUTE (@_getDataSet) AT ['+@_linkedServer+'];';

EXECUTE (@_createLinkedserver);
INSERT INTO @_tempTable_restoreDataset
EXEC sp_executesql @_forExecution, 
N'@_getDataSet NVARCHAR(MAX)'
,@_getDataSet;
EXECUTE (@_dropLinkedServer);

SELECT @_restoreFullpath = restorePath 
FROM @_tempTable_restoreDataset
WHERE backupType = 'FULL'

SELECT @_restoreDiffpath = restorePath 
FROM @_tempTable_restoreDataset
WHERE backupType = 'DIFF'


SET @_restoreFull = '

USE [master];

IF DB_ID('''+@databaseName+''') IS NULL
            CREATE DATABASE ['+@databaseName+'];

RESTORE DATABASE ['+@databaseName+'] FROM  
DISK = N'''+@_restoreFullpath+''' 
WITH  
NOUNLOAD, 
NORECOVERY,
REPLACE,  
STATS = 5;
';

--EXEC dbo.killUsers @databaseName,0
IF(@print = 0) EXECUTE (@_restoreFull) ELSE PRINT (@_restoreFull);

SET @_restoreDiff = '
RESTORE DATABASE ['+@databaseName+'] 
FROM  DISK = N'''+@_restoreDiffpath+''' 
WITH  FILE = 1,  
NOUNLOAD, 
NORECOVERY,
STATS = 5;

'
IF(@print = 0) EXECUTE (@_restoreDiff) ELSE PRINT (@_restoreDiff);


SET @_restoreLogpath = ''
DECLARE foreach CURSOR FOR 

SELECT 
restorePath, 
@databaseName
FROM @_tempTable_restoreDataset
WHERE backupType = 'LOG'

OPEN foreach  
FETCH NEXT FROM foreach INTO 
@_restoreLogpath,
@_databaseName
	
	WHILE @@FETCH_STATUS = 0  
	BEGIN 
	
	PRINT(@_restoreLogpath)
	SET @_restoreLogpath = 'RESTORE LOG ['+@databaseName+'] FROM  DISK = N'''+@_restoreLogpath+'''  WITH  FILE = 1,  NORECOVERY, NOUNLOAD, STATS = 5;
	'
	IF(@print = 0) EXECUTE (@_restoreLogpath) ELSE PRINT (@_restoreLogpath);

	FETCH NEXT FROM foreach INTO 
	@_restoreLogpath,
	@_databaseName
	END 
  
CLOSE foreach  
DEALLOCATE foreach 	

SET @_recoverDatabase = '
RESTORE DATABASE ['+@databaseName+'] WITH RECOVERY;
'
SET @_setReadonly = '

ALTER DATABASE ['+@databaseName+'] SET READ_ONLY WITH NO_WAIT;
'

IF(@recovery = 1)
	IF(@print = 0) EXECUTE (@_recoverDatabase) ELSE PRINT (@_recoverDatabase);

IF(@databaseReadonly = 1)
BEGIN
EXEC dbo.killUsers @databaseName,0
IF(@print = 0) EXECUTE (@_setReadonly) ELSE PRINT (@_setReadonly);
END

GO
-----------------------------------------------------------------------------------------------------------------








-- SQLCMD MEMORY CHANGE
-----------------------------------------------------------------------------------------------------------------
SELECT [name], [value], [value_in_use]
FROM sys.configurations
WHERE [name] = 'max server memory (MB)'

sp_configure 'show advanced options', 1;
GO
RECONFIGURE;
GO
sp_configure 'max server memory', 11000;
GO
RECONFIGURE;
GO
-----------------------------------------------------------------------------------------------------------------










-- PROVJERA KADA JE ZADNJI PUT JOB IMAO FAIL STEP
-----------------------------------------------------------------------------------------------------------------
select TOP 1 j.name  
    ,js.step_name
    ,jh.sql_severity
    ,jh.message
    ,jh.run_date
    ,jh.run_time
	,CD.last_executed_step_date
	,CD.stop_execution_date
FROM msdb.dbo.sysjobs AS j
INNER JOIN msdb.dbo.sysjobsteps AS js
   ON js.job_id = j.job_id
INNER JOIN msdb.dbo.sysjobhistory AS jh
   ON jh.job_id = j.job_id AND jh.step_id = js.step_id
INNER JOIN msdb.dbo.sysjobactivity AS CD
   ON j.job_id = CD.job_id
WHERE jh.run_status = 0 AND 
j.name = '3' AND jh.run_date = (select convert(varchar, getdate(), 112)) 
AND (SELECT convert(varchar, CD.stop_execution_date, 112)) = (select convert(varchar, getdate(), 112)) 
AND DATEPART(HOUR, LAST_EXECUTED_STEP_DATE) > '1'
ORDER BY CD.last_executed_step_date  desc
-----------------------------------------------------------------------------------------------------------------















-- PROVJERA OWNER JOBOVA PAKETA
-----------------------------------------------------------------------------------------------------------------
select s.name,l.name
 from  msdb..sysjobs s 
 left join master.sys.syslogins l on s.owner_sid = l.sid

 select s.name,l.name 
from msdb..sysssispackages s 
 left join master.sys.syslogins l on s.ownersid = l.sid

 SELECT  s.name ,
        SUSER_SNAME(s.owner_sid) AS owner
FROM    msdb..sysjobs s 
ORDER BY name
-----------------------------------------------------------------------------------------------------------------





-----------------------------------------------------------------------------------------------------------------
-- PROVJERA SYSADMIN ROLE
SELECT 'Name' = sp.NAME
	,sp.is_disabled AS [Is_disabled]
FROM sys.server_role_members rm
	,sys.server_principals sp
WHERE rm.role_principal_id = SUSER_ID('Sysadmin')
	AND rm.member_principal_id = sp.principal_id
-----------------------------------------------------------------------------------------------------------------












-- PROVJERA KADA JE PROCEDURA IZVRSENA
-----------------------------------------------------------------------------------------------------------------
SELECT  
          SCHEMA_NAME(sysobject.schema_id) AS SchemaName,
          OBJECT_NAME(PS.object_id) AS ProcedureName, 
          execution_count AS NoOfTimesExecuted,
          PS.last_execution_time AS LastExecutedOn
    FROM   
         sys.dm_exec_procedure_stats PS
         INNER JOIN sys.objects sysobject ON sysobject.object_id =   
         PS.object_id 
    WHERE  
         sysobject.type = 'P'
    ORDER BY
         PS.last_execution_time DESC  
-----------------------------------------------------------------------------------------------------------------

-- PROVJERA KADA JE ZADNJI MODIFIKATE PROCEDURE BIO
SELECT name, create_date, modify_date
FROM sys.objects
WHERE type = 'P'
AND name = 'IZV_Pregled_broja_klijenata'
ORDER BY modify_date DESC
-----------------------------------------------------------------------------------------------------------------



--CHECK LAST EXECUTION PROCEDUR, PROVJERA ZADNJE OKINUTE PROCEDURE PROCEDURA
-----------------------------------------------------------------------------------------------------------------
SELECT  
    SCHEMA_NAME(sysobject.schema_id),
    OBJECT_NAME(stats.object_id), 
    stats.last_execution_time
FROM   
    sys.dm_exec_procedure_stats stats
    INNER JOIN sys.objects sysobject ON sysobject.object_id = stats.object_id 
WHERE  
    sysobject.type = 'P'
    and (sysobject.object_id = object_id('[dbo].[NL_OsigDep7]') 
    OR sysobject.name = 'NL_OsigDep7')
ORDER BY
       stats.last_execution_time DESC  
-----------------------------------------------------------------------------------------------------------------
SELECT TOP 10 d.object_id, d.database_id, OBJECT_NAME(object_id, database_id) 'proc name',   
    d.cached_time, d.last_execution_time, d.total_elapsed_time,  
    d.total_elapsed_time/d.execution_count AS [avg_elapsed_time],  
    d.last_elapsed_time, d.execution_count  
FROM sys.dm_exec_procedure_stats AS d  
ORDER BY [total_worker_time] DESC;  










-- CHECK ROW SIZE TABLE 
-----------------------------------------------------------------------------------------------------------------
SELECT OBJECT_NAME (id) tablename
     , COUNT (1)        nr_columns
     , SUM (length)     maxrowlength
FROM   syscolumns
GROUP BY OBJECT_NAME (id)
ORDER BY OBJECT_NAME (id)
-----------------------------------------------------------------------------------------------------------------
-- CHECK TABLE ROW SIZE

SELECT 
    t.NAME AS TableName,
    s.Name AS SchemaName,
    p.rows AS RowCounts,
    SUM(a.total_pages) * 8 AS TotalSpaceKB, 
    CAST(ROUND(((SUM(a.total_pages) * 8) / 1024.00), 2) AS NUMERIC(36, 2)) AS TotalSpaceMB,
    SUM(a.used_pages) * 8 AS UsedSpaceKB, 
    CAST(ROUND(((SUM(a.used_pages) * 8) / 1024.00), 2) AS NUMERIC(36, 2)) AS UsedSpaceMB, 
    (SUM(a.total_pages) - SUM(a.used_pages)) * 8 AS UnusedSpaceKB,
    CAST(ROUND(((SUM(a.total_pages) - SUM(a.used_pages)) * 8) / 1024.00, 2) AS NUMERIC(36, 2)) AS UnusedSpaceMB
FROM 
    sys.tables t
INNER JOIN      
    sys.indexes i ON t.OBJECT_ID = i.object_id
INNER JOIN 
    sys.partitions p ON i.object_id = p.OBJECT_ID AND i.index_id = p.index_id
INNER JOIN 
    sys.allocation_units a ON p.partition_id = a.container_id
LEFT OUTER JOIN 
    sys.schemas s ON t.schema_id = s.schema_id
INNER JOIN  sysobjects so on t.object_id = so.id
INNER JOIN  syscolumns SC on (so.id = sc.id)
INNER JOIN systypes st on (st.type = sc.type)
WHERE 
    t.NAME NOT LIKE 'dt%' 
    AND t.is_ms_shipped = 0
    AND i.OBJECT_ID > 255 
    AND so.type = 'U'
and st.name IN ('DATETIME', 'DATE', 'TIME')
GROUP BY 
    t.Name, s.Name, p.Rows
ORDER BY 
    p.rows DESC
-----------------------------------------------------------------------------------------------------------------












-- PROVJERA TABELA
--CHECK TABLES USED IN DB PROVJERA TABELA
-----------------------------------------------------------------------------------------------------------------
SELECT DB_NAME([database_id]) as DB,
       OBJECT_NAME(dm_db_index_usage_stats.[object_id], [database_id]) as tb,
       TotalReads = (user_seeks + user_scans + user_lookups),
       Writes = user_updates,
       indexes.[name],
       *
FROM sys.dm_db_index_usage_stats
LEFT JOIN sys.indexes on dm_db_index_usage_stats.[object_id] = indexes.[object_id] and dm_db_index_usage_stats.[index_id] = indexes.[index_id]
where DB_NAME([database_id]) = 'KremaTransactMig3' 
and OBJECT_NAME(dm_db_index_usage_stats.[object_id], [database_id]) = 'F_TSA_STATUS'
-----------------------------------------------------------------------------------------------------------------
-- PROVJERA OBJEKATA
SELECT DISTINCT o.name AS Object_Name,o.type_desc, *
FROM sys.sql_modules m INNER JOIN sys.objects o ON m.object_id=o.object_id WHERE m.definition LIKE '%dbo.ekspoleasing%' ORDER BY 2,1
-----------------------------------------------------------------------------------------------------------------





--PROVJERA STATUSA INDEXA                <<<<<<<<<<<<<<<<<<<<<<
-----------------------------------------------------------------------------------------------------------------
SELECT percent_complete, estimated_completion_time, reads, writes, logical_reads, text_size, *
FROM
sys.dm_exec_requests AS r
WHERE
r.session_id <> @@SPID
AND r.session_id = 155
-----------------------------------------------------------------------------------------------------------------






-- PROVJERA SESIJA amar
-- CHECK SVIH QUERY 
--CHECK DBCCC SHRINKFILE SHRINK                 <<<<<<<<<<<<<<<<<<<<<<
-----------------------------------------------------------------------------------------------------------------
select
a.session_id
,P.loginame
, command
, b.text
, percent_complete
, done_in_minutes = a.estimated_completion_time / 1000 / 60
, min_in_progress = DATEDIFF(MI, a.start_time, DATEADD(ms, a.estimated_completion_time, GETDATE() ))
, a.start_time
, estimated_completion_time = DATEADD(ms, a.estimated_completion_time, GETDATE() )
,P.program_name
from sys.dm_exec_requests a
inner join master.dbo.sysprocesses P
on P.spid = a.session_id
CROSS APPLY sys.dm_exec_sql_text(a.sql_handle) b
where command like '%dbcc%'

--DRUGI DIO
	select
    P.spid
,   right(convert(varchar, 
            dateadd(ms, datediff(ms, P.last_batch, getdate()), '1900-01-01'), 
            121), 12) as 'batch_duration'
,   P.program_name
,   P.hostname
,   P.loginame
from master.dbo.sysprocesses P
where P.spid > 50
and      P.status not in ('background', 'sleeping')
and      P.cmd not in ('AWAITING COMMAND'
                    ,'MIRROR HANDLER'
                    ,'LAZY WRITER'
                    ,'CHECKPOINT SLEEP'
                    ,'RA MANAGER')
order by batch_duration desc




-----------------------------------------------------------------------------------------------------------------


;WITH task_space_usage AS (
    -- SUM alloc/delloc pages
    SELECT session_id,
           request_id,
           SUM(internal_objects_alloc_page_count) AS alloc_pages,
           SUM(internal_objects_dealloc_page_count) AS dealloc_pages
    FROM sys.dm_db_task_space_usage WITH (NOLOCK)
    WHERE session_id <> @@SPID
    GROUP BY session_id, request_id
)
SELECT TSU.session_id,
       TSU.alloc_pages * 1.0 / 128 AS [internal object MB space],
       TSU.dealloc_pages * 1.0 / 128 AS [internal object dealloc MB space],
       EST.text,
       -- Extract statement from sql text
       ISNULL(
           NULLIF(
               SUBSTRING(
                   EST.text, 
                   ERQ.statement_start_offset / 2, 
                   CASE WHEN ERQ.statement_end_offset < ERQ.statement_start_offset THEN 0 ELSE( ERQ.statement_end_offset - ERQ.statement_start_offset ) / 2 END
               ), ''
           ), EST.text
       ) AS [statement text],
       EQP.query_plan
FROM task_space_usage AS TSU
INNER JOIN sys.dm_exec_requests ERQ WITH (NOLOCK)
    ON  TSU.session_id = ERQ.session_id
    AND TSU.request_id = ERQ.request_id
OUTER APPLY sys.dm_exec_sql_text(ERQ.sql_handle) AS EST
OUTER APPLY sys.dm_exec_query_plan(ERQ.plan_handle) AS EQP
WHERE EST.text IS NOT NULL OR EQP.query_plan IS NOT NULL
ORDER BY 3 DESC, 5 DESC






-----------------------------------------------------------------------------------------------------------------

select  top(100)
        creation_time,
        last_execution_time,
        execution_count,
        total_worker_time/1000 as CPU,
        convert(money, (total_worker_time))/(execution_count*1000)as [AvgCPUTime],
        qs.total_elapsed_time/1000 as TotDuration,
        convert(money, (qs.total_elapsed_time))/(execution_count*1000)as [AvgDur],
        total_logical_reads as [Reads],
        total_logical_writes as [Writes],
        total_logical_reads+total_logical_writes as [AggIO],
        convert(money, (total_logical_reads+total_logical_writes)/(execution_count + 0.0)) as [AvgIO],
        [sql_handle],
        plan_handle,
        statement_start_offset,
        statement_end_offset,
        plan_generation_num,
        total_physical_reads,
        convert(money, total_physical_reads/(execution_count + 0.0)) as [AvgIOPhysicalReads],
        convert(money, total_logical_reads/(execution_count + 0.0)) as [AvgIOLogicalReads],
        convert(money, total_logical_writes/(execution_count + 0.0)) as [AvgIOLogicalWrites],
        query_hash,
        query_plan_hash,
        total_rows,
        convert(money, total_rows/(execution_count + 0.0)) as [AvgRows],
        total_dop,
        convert(money, total_dop/(execution_count + 0.0)) as [AvgDop],
        total_grant_kb,
        convert(money, total_grant_kb/(execution_count + 0.0)) as [AvgGrantKb],
        total_used_grant_kb,
        convert(money, total_used_grant_kb/(execution_count + 0.0)) as [AvgUsedGrantKb],
        total_ideal_grant_kb,
        convert(money, total_ideal_grant_kb/(execution_count + 0.0)) as [AvgIdealGrantKb],
        total_reserved_threads,
        convert(money, total_reserved_threads/(execution_count + 0.0)) as [AvgReservedThreads],
        total_used_threads,
        convert(money, total_used_threads/(execution_count + 0.0)) as [AvgUsedThreads],
        case 
            when sql_handle IS NULL then ' '
            else(substring(st.text,(qs.statement_start_offset+2)/2,(
                case
                    when qs.statement_end_offset =-1 then len(convert(nvarchar(MAX),st.text))*2      
                    else qs.statement_end_offset    
                end - qs.statement_start_offset)/2  ))
        end as query_text,
        db_name(st.dbid) as database_name,
        object_schema_name(st.objectid, st.dbid)+'.'+object_name(st.objectid, st.dbid) as [object_name],
        sp.[query_plan]
from sys.dm_exec_query_stats as qs with(readuncommitted)
cross apply sys.dm_exec_sql_text(qs.[sql_handle]) as st
cross apply sys.dm_exec_query_plan(qs.[plan_handle]) as sp
WHERE st.[text] LIKE '%query%'








--PROVJERA PROCESA U TRENUTKU RADA POSTOTAK
-----------------------------------------------------------------------------------------------------------------
SELECT
r.session_id
,r.command
,CONVERT(NUMERIC(6,2), r.percent_complete) AS [Percent Complete]
,CONVERT(VARCHAR(20), DATEADD(ms,r.estimated_completion_time,GetDate()),20) AS [ETA Completion Time]
,CONVERT(NUMERIC(10,2), r.total_elapsed_time/1000.0/60.0) AS [Elapsed Min]
,CONVERT(NUMERIC(10,2), r.estimated_completion_time/1000.0/60.0) AS [ETA Min]
,CONVERT(NUMERIC(10,2), r.estimated_completion_time/1000.0/60.0/60.0) AS [ETA Hours]
,CONVERT(VARCHAR(1000)
,(SELECT SUBSTRING(text,r.statement_start_offset/2, CASE WHEN r.statement_end_offset = -1
THEN 1000
ELSE (r.statement_end_offset-r.statement_start_offset)/2
END)
FROM sys.dm_exec_sql_text(sql_handle)
)
) AS [SQL]
FROM sys.dm_exec_requests r
WHERE command IN ('RESTORE DATABASE', 'BACKUP DATABASE')
-----------------------------------------------------------------------------------------------------------------










--ozbiljne provjere RAMO
--CPU PROVJERE: <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
-----------------------------------------------------------------------------------------------------------------
WITH DB_CPU_Stats
AS
(
    SELECT DatabaseID, isnull(DB_Name(DatabaseID),case DatabaseID when 32767 then 'Internal ResourceDB' else CONVERT(varchar(255),DatabaseID)end) AS [DatabaseName], 
      SUM(total_worker_time) AS [CPU Time Ms],
      SUM(total_logical_reads)  AS [Logical Reads],
      SUM(total_logical_writes)  AS [Logical Writes],
      SUM(total_logical_reads+total_logical_writes)  AS [Logical IO],
      SUM(total_physical_reads)  AS [Physical Reads],
      SUM(total_elapsed_time)  AS [Duration MicroSec],
      SUM(total_clr_time)  AS [CLR Time MicroSec],
      SUM(total_rows)  AS [Rows Returned],
      SUM(execution_count)  AS [Execution Count],
      count(*) 'Plan Count'

    FROM sys.dm_exec_query_stats AS qs
    CROSS APPLY (
                    SELECT CONVERT(int, value) AS [DatabaseID] 
                  FROM sys.dm_exec_plan_attributes(qs.plan_handle)
                  WHERE attribute = N'dbid') AS F_DB
    GROUP BY DatabaseID
)
SELECT ROW_NUMBER() OVER(ORDER BY [CPU Time Ms] DESC) AS [Rank CPU],
       DatabaseName,
       [CPU Time Hr] = convert(decimal(15,2),([CPU Time Ms]/1000.0)/3600) ,
        CAST([CPU Time Ms] * 1.0 / SUM([CPU Time Ms]) OVER() * 100.0 AS DECIMAL(5, 2)) AS [CPU Percent],
       [Duration Hr] = convert(decimal(15,2),([Duration MicroSec]/1000000.0)/3600) , 
       CAST([Duration MicroSec] * 1.0 / SUM([Duration MicroSec]) OVER() * 100.0 AS DECIMAL(5, 2)) AS [Duration Percent],    
       [Logical Reads],
        CAST([Logical Reads] * 1.0 / SUM([Logical Reads]) OVER() * 100.0 AS DECIMAL(5, 2)) AS [Logical Reads Percent],      
       [Rows Returned],
        CAST([Rows Returned] * 1.0 / SUM([Rows Returned]) OVER() * 100.0 AS DECIMAL(5, 2)) AS [Rows Returned Percent],
       [Reads Per Row Returned] = [Logical Reads]/[Rows Returned],
       [Execution Count],
        CAST([Execution Count] * 1.0 / SUM([Execution Count]) OVER() * 100.0 AS DECIMAL(5, 2)) AS [Execution Count Percent],
       [Physical Reads],
       CAST([Physical Reads] * 1.0 / SUM([Physical Reads]) OVER() * 100.0 AS DECIMAL(5, 2)) AS [Physcal Reads Percent], 
       [Logical Writes],
        CAST([Logical Writes] * 1.0 / SUM([Logical Writes]) OVER() * 100.0 AS DECIMAL(5, 2)) AS [Logical Writes Percent],
       [Logical IO],
        CAST([Logical IO] * 1.0 / SUM([Logical IO]) OVER() * 100.0 AS DECIMAL(5, 2)) AS [Logical IO Percent],
       [CLR Time MicroSec],
       CAST([CLR Time MicroSec] * 1.0 / SUM(case [CLR Time MicroSec] when 0 then 1 else [CLR Time MicroSec] end ) OVER() * 100.0 AS DECIMAL(5, 2)) AS [CLR Time Percent],
       [CPU Time Ms],[CPU Time Ms]/1000 [CPU Time Sec],
       [Duration MicroSec],[Duration MicroSec]/1000000 [Duration Sec]
FROM DB_CPU_Stats
--WHERE DatabaseID > 4 -- system databases
--AND DatabaseID <> 32767 -- ResourceDB
ORDER BY [Rank CPU] OPTION (RECOMPILE);


-----------------------------------------------------------------------------------------------------------------
--512+((8-4)*16) = 576
SELECT max_workers_count FROM sys.dm_os_sys_info
select * from sys.dm_os_sys_info

	    SELECT cpu_count AS [Logical CPU Count], hyperthread_ratio AS Hyperthread_Ratio,
cpu_count/hyperthread_ratio AS Physical_CPU_Count,
--physical_memory_in_bytes/1048576 AS Physical_Memory_in_MB,
sqlserver_start_time, affinity_type_desc -- (affinity_type_desc is only in 2008 R2)
FROM sys.dm_os_sys_info


 SELECT TOP 1
        record_time,
        SQLProcessUtilization,
        SystemIdle,
        100 - SystemIdle - SQLProcessUtilization AS OtherProcessUtilization
    FROM (
        SELECT 
            record.value('(./Record/@id)[1]', 'int') AS record_id,
            record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int') AS SystemIdle,
            record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]', 'int') AS SQLProcessUtilization,
            record_time
        FROM (
            select 
                dateadd (ms, r.[timestamp] - sys.ms_ticks, getdate()) as record_time,  
                cast(r.record as xml) record  
            from sys.dm_os_ring_buffers r  
            cross join sys.dm_os_sys_info sys  
            where   
                ring_buffer_type='RING_BUFFER_SCHEDULER_MONITOR' 
                AND 
                record LIKE '%<SystemHealth>%'
            ) AS x
        ) AS y 

-----------------------------------------------------------------------------------------------------------------
SELECT
   [ReadLatency] =
        CASE WHEN [num_of_reads] = 0
            THEN 0 ELSE ([io_stall_read_ms] / [num_of_reads]) END,
   [WriteLatency] =
        CASE WHEN [num_of_writes] = 0
            THEN 0 ELSE ([io_stall_write_ms] / [num_of_writes]) END,
   [Latency] =
        CASE WHEN ([num_of_reads] = 0 AND [num_of_writes] = 0)
            THEN 0 ELSE ([io_stall] / ([num_of_reads] + [num_of_writes])) END,
   [Latency Desc] = 
         CASE 
            WHEN ([num_of_reads] = 0 AND [num_of_writes] = 0) THEN 'N/A' 
            ELSE 
               CASE WHEN ([io_stall] / ([num_of_reads] + [num_of_writes])) < 2 THEN 'Excellent'
                    WHEN ([io_stall] / ([num_of_reads] + [num_of_writes])) < 6 THEN 'Very good'
                    WHEN ([io_stall] / ([num_of_reads] + [num_of_writes])) < 11 THEN 'Good'
                    WHEN ([io_stall] / ([num_of_reads] + [num_of_writes])) < 21 THEN 'Poor'
                    WHEN ([io_stall] / ([num_of_reads] + [num_of_writes])) < 101 THEN 'Bad'
                    WHEN ([io_stall] / ([num_of_reads] + [num_of_writes])) < 501 THEN 'Yikes!'
               ELSE 'YIKES!!'
               END 
         END, 
   [AvgBPerRead] =
        CASE WHEN [num_of_reads] = 0
            THEN 0 ELSE ([num_of_bytes_read] / [num_of_reads]) END,
   [AvgBPerWrite] =
        CASE WHEN [num_of_writes] = 0
            THEN 0 ELSE ([num_of_bytes_written] / [num_of_writes]) END,
   [AvgBPerTransfer] =
        CASE WHEN ([num_of_reads] = 0 AND [num_of_writes] = 0)
            THEN 0 ELSE
                (([num_of_bytes_read] + [num_of_bytes_written]) /
                ([num_of_reads] + [num_of_writes])) END,
   LEFT ([mf].[physical_name], 2) AS [Drive],
   DB_NAME ([vfs].[database_id]) AS [DB],
   [mf].[physical_name]
FROM
   sys.dm_io_virtual_file_stats (NULL,NULL) AS [vfs]
   JOIN sys.master_files AS [mf]
   ON [vfs].[database_id] = [mf].[database_id]
      AND [vfs].[file_id] = [mf].[file_id]
-- WHERE [vfs].[file_id] = 2 -- log files
ORDER BY [Latency] DESC
-- ORDER BY [ReadLatency] DESC
-- ORDER BY [WriteLatency] DESC;
GO
-----------------------------------------------------------------------------------------------------------------



-----------------------------------------------------------------------------------------------------------------
-- PROVJERA SESIJA 
-----------------------------------------------------------------------------------------------------------------
USE msdb   
GO
SET NOCOUNT ON   
SET ANSI_PADDING ON
SET QUOTED_IDENTIFIER ON  
DECLARE @record_id int, @SQLProcessUtilization int, @CPU int,@EventTime datetime--,@MaxCPUAllowed int   
select  top 1  @record_id =record_id,
      @EventTime=dateadd(ms, -1 * ((SELECT ms_ticks from sys.dm_os_sys_info) - [timestamp]), GetDate()),-- as EventTime,
      @SQLProcessUtilization=SQLProcessUtilization,
      --SystemIdle,
      --100 - SystemIdle - SQLProcessUtilization as OtherProcessUtilization,
      @CPU=SQLProcessUtilization + (100 - SystemIdle - SQLProcessUtilization) --as CPU_Usage
from (
      select
            record.value('(./Record/@id)[1]', 'int') as record_id,
            record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int') as SystemIdle,
            record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]', 'int') as SQLProcessUtilization,
            timestamp
      from (
            select timestamp, convert(xml, record) as record
            from sys.dm_os_ring_buffers
            where ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
            and record like '%<SystemHealth>%') as x
      ) as y
order by record_id desc 
SELECT           
            x.session_id as [Sid],
            COALESCE(x.blocking_session_id, 0) as BSid,
            @CPU as CPU,   
            @SQLProcessUtilization as SQL,  

            x.Status,  
            x.TotalCPU as [T.CPU],
            x.Start_time,    
            CONVERT(nvarchar(30), getdate()-x.Start_time, 108) as Elap_time, --x.totalElapsedTime as ElapTime,
            x.totalReads as [T.RD], -- total reads
            x.totalWrites as [T.WR], --total writes     
            x.Writes_in_tempdb as [W.TDB],
            (
                  SELECT substring(text,x.statement_start_offset/2,
                        (case when x.statement_end_offset = -1
                        then len(convert(nvarchar(max), text)) * 2
                        else x.statement_end_offset end - x.statement_start_offset+3)/2)
                  FROM sys.dm_exec_sql_text(x.sql_handle)
                  FOR XML PATH(''), TYPE
            ) AS Sql_text,
            db_name(x.database_id) as dbName,
            (SELECT object_name(objectid) FROM sys.dm_exec_sql_text(x.sql_handle)) as object_name,           
            x.Wait_type,
            x.Login_name,
            x.Host_name,
            CASE LEFT(x.program_name,15)
            WHEN 'SQLAgent - TSQL' THEN 
            (     select top 1 'SQL Job = '+j.name from msdb.dbo.sysjobs (nolock) j
                  inner join msdb.dbo.sysjobsteps (nolock) s on j.job_id=s.job_id
                  where right(cast(s.job_id as nvarchar(50)),10) = RIGHT(substring(x.program_name,30,34),10) )
            WHEN 'SQL Server Prof' THEN 'SQL Server Profiler'
            ELSE x.program_name
            END as Program_name,
            x.percent_complete,
            x.percent_complete, 
            (
                  SELECT
                        p.text
                  FROM
                  (
                        SELECT
                             sql_handle,statement_start_offset,statement_end_offset
                        FROM sys.dm_exec_requests r2
                        WHERE
                             r2.session_id = x.blocking_session_id
                  ) AS r_blocking
                  CROSS APPLY
                  (
                  SELECT substring(text,r_blocking.statement_start_offset/2,
                        (case when r_blocking.statement_end_offset = -1
                        then len(convert(nvarchar(max), text)) * 2
                        else r_blocking.statement_end_offset end - r_blocking.statement_start_offset+3)/2)
                  FROM sys.dm_exec_sql_text(r_blocking.sql_handle)
                  FOR XML PATH(''), TYPE
                  ) p (text)
            )  as blocking_text,
            (SELECT object_name(objectid) FROM sys.dm_exec_sql_text(
            (select top 1 sql_handle FROM sys.dm_exec_requests r3 WHERE r3.session_id = x.blocking_session_id))) as blocking_obj

      FROM
      (
            SELECT
                  r.session_id,
                  s.host_name,
                  s.login_name,
                  r.start_time,
                  r.sql_handle,
                  r.database_id,
                  r.blocking_session_id,
                  r.wait_type,
                  r.status,
                  r.statement_start_offset,
                  r.statement_end_offset,
                  s.program_name,
                  r.percent_complete,               
                  SUM(cast(r.total_elapsed_time as bigint)) /1000 as totalElapsedTime, --CAST AS BIGINT to fix invalid data convertion when high activity
                  SUM(cast(r.reads as bigint)) AS totalReads,
                  SUM(cast(r.writes as bigint)) AS totalWrites,
                  SUM(cast(r.cpu_time as bigint)) AS totalCPU,
                  SUM(tsu.user_objects_alloc_page_count + tsu.internal_objects_alloc_page_count) AS writes_in_tempdb
            FROM sys.dm_exec_requests r
            JOIN sys.dm_exec_sessions s ON s.session_id = r.session_id
            JOIN sys.dm_db_task_space_usage tsu ON s.session_id = tsu.session_id and r.request_id = tsu.request_id
            WHERE r.status IN ('running', 'runnable', 'suspended')
            GROUP BY
                  r.session_id,
                  s.host_name,
                  s.login_name,
                  r.start_time,
                  r.sql_handle,
                  r.database_id,
                  r.blocking_session_id,
                  r.wait_type,
                  r.status,
                  r.statement_start_offset,
                  r.statement_end_offset,
                  s.program_name,
                  r.percent_complete
      ) x
      where x.session_id <> @@spid AND db_name(x.database_id) = 'KremaTransactPreProd'
      order by x.totalCPU desc
GO
-----------------------------------------------------------------------------------------------------------------









-- CPU BAZA PROVJERA RAMO
-----------------------------------------------------------------------------------------------------------------
WITH DB_CPU_Stats
AS
(
    SELECT DatabaseID, DB_Name(DatabaseID) AS [DatabaseName], 
      SUM(total_worker_time) AS [CPU_Time_Ms]
    FROM sys.dm_exec_query_stats AS qs
    CROSS APPLY (
                    SELECT CONVERT(int, value) AS [DatabaseID] 
                  FROM sys.dm_exec_plan_attributes(qs.plan_handle)
                  WHERE attribute = N'dbid') AS F_DB
    GROUP BY DatabaseID
)
SELECT ROW_NUMBER() OVER(ORDER BY [CPU_Time_Ms] DESC) AS [row_num],
       DatabaseName,
        [CPU_Time_Ms], 
       CAST([CPU_Time_Ms] * 1.0 / SUM([CPU_Time_Ms]) OVER() * 100.0 AS DECIMAL(5, 2)) AS [CPUPercent]
FROM DB_CPU_Stats
--WHERE DatabaseID > 4 -- system databases
--AND DatabaseID <> 32767 -- ResourceDB
ORDER BY row_num OPTION (RECOMPILE);


DECLARE @total INT
SELECT @total=sum(cpu) FROM sys.sysprocesses sp (NOLOCK)
    join sys.sysdatabases sb (NOLOCK) ON sp.dbid = sb.dbid

SELECT sb.name 'database', @total 'system cpu', SUM(cpu) 'database cpu', CONVERT(DECIMAL(4,1), CONVERT(DECIMAL(17,2),SUM(cpu)) / CONVERT(DECIMAL(17,2),@total)*100) '%'
FROM sys.sysprocesses sp (NOLOCK)
JOIN sys.sysdatabases sb (NOLOCK) ON sp.dbid = sb.dbid
--WHERE sp.status = 'runnable'
GROUP BY sb.name
ORDER BY CONVERT(DECIMAL(4,1), CONVERT(DECIMAL(17,2),SUM(cpu)) / CONVERT(DECIMAL(17,2),@total)*100) desc


-----------------------------------------------------------------------------------------------------------------


--WHERE (backup_finish_date > ''2024-10-28 21:00:00'' AND backup_finish_date <= ''@restoreDatetime'')  








--HISTORY RESTORE DATABASE PROVJERA ZADNJI LOG SHIPPING
-----------------------------------------------------------------------------------------------------------------
declare @DB sysname = 'ews';
select * from msdb.dbo.restorehistory where destination_database_name = @DB
order by restore_date desc
----------------------------------------------

-- PROVJERA RESTORE PROCESA    <<<<<<<<<
SELECT
 rh.destination_database_name AS [BAZA],
  CASE WHEN rh.restore_type = 'D' THEN 'Database'
  WHEN rh.restore_type = 'F' THEN 'File'
   WHEN rh.restore_type = 'I' THEN 'Differential'
  WHEN rh.restore_type = 'L' THEN 'Log'
    ELSE rh.restore_type 
 END AS [Restore Type],
 rh.restore_date AS [Restore Date],
 bmf.physical_device_name AS [Source], 
 rf.destination_phys_name AS [Restore File],
  rh.user_name AS [Restored By]
FROM msdb.dbo.restorehistory rh
 INNER JOIN msdb.dbo.backupset bs ON rh.backup_set_id = bs.backup_set_id
 INNER JOIN msdb.dbo.restorefile rf ON rh.restore_history_id = rf.restore_history_id
 INNER JOIN msdb.dbo.backupmediafamily bmf ON bmf.media_set_id = bs.media_set_id
ORDER BY rh.restore_history_id DESC
GO
-----------------------------------------------------------------------------------------------------------------

-- PROVJERA SVIH BACKUP PROCESA  <<<<<<<<<<<<<
SELECT 
   CONVERT(CHAR(100), SERVERPROPERTY('Servername')) AS Server, 
   msdb.dbo.backupset.database_name, 
   msdb.dbo.backupset.backup_start_date, 
   msdb.dbo.backupset.backup_finish_date, 
   msdb.dbo.backupset.expiration_date, 
   CASE msdb..backupset.type 
      WHEN 'D' THEN 'Database' 
      WHEN 'L' THEN 'Log' 
      END AS backup_type, 
   msdb.dbo.backupset.backup_size, 
   msdb.dbo.backupmediafamily.logical_device_name, 
   msdb.dbo.backupmediafamily.physical_device_name, 
   msdb.dbo.backupset.name AS backupset_name, 
   msdb.dbo.backupset.description 
FROM 
   msdb.dbo.backupmediafamily 
   INNER JOIN msdb.dbo.backupset ON msdb.dbo.backupmediafamily.media_set_id = msdb.dbo.backupset.media_set_id 
WHERE 
   (CONVERT(datetime, msdb.dbo.backupset.backup_start_date, 102) >= GETDATE() - 100) 
   and msdb..backupset.type = 'D' and msdb.dbo.backupset.database_name = 'GG30_live' 
ORDER BY 
   msdb.dbo.backupset.database_name, 
   msdb.dbo.backupset.backup_finish_date DESC
-----------------------------------------------------------------------------------------------------------------





--PROVJERA JOBOVA
-- PROVJERA PALIH JOBOVA STEPOVA U DANU <<<<<<<<<
-----------------------------------------------------------------------------------------------------------------
DECLARE @FinalDate INT;
SET @FinalDate = CONVERT(int
    , CONVERT(varchar(10), DATEADD(DAY, -1, GETDATE()), 112)
    ) -- Yesterday's date as Integer in YYYYMMDD format

-- Final Logic 

SELECT  j.[name],  
        s.step_name,  
        h.step_id,  
        h.step_name,  
        h.run_date,  
        h.run_time,  
        h.sql_severity,  
        h.message,   
        h.server  
FROM    msdb.dbo.sysjobhistory h  
        INNER JOIN msdb.dbo.sysjobs j  
            ON h.job_id = j.job_id  
        INNER JOIN msdb.dbo.sysjobsteps s  
            ON j.job_id = s.job_id 
                AND h.step_id = s.step_id  
WHERE    h.run_status = 0 -- Failure  
         AND h.run_date > @FinalDate  
ORDER BY h.instance_id DESC;
-----------------------------------------------------------------------------------------------------------------







--PROVJERA SSIS PAKETA PALIH
-- PROVJERA PALIH paketa U DANU <<<<<<<<<
-----------------------------------------------------------------------------------------------------------------
DECLARE @DATE DATE = GETDATE() - 7

SELECT O.Operation_Id -- Not much of use 
,E.Folder_Name AS Project_Name 
,E.Project_name AS SSIS_Project_Name 
,EM.Package_Name 
,CONVERT(DATETIME, O.start_time) AS Start_Time 
,CONVERT(DATETIME, O.end_time) AS End_Time 
,OM.message as [Error_Message] 
,EM.Event_Name 
,EM.Message_Source_Name AS Component_Name 
,EM.Subcomponent_Name AS Sub_Component_Name 
,E.Environment_Name 
,CASE E.Use32BitRunTime 
WHEN 1 
THEN 'Yes' 
ELSE 'NO' 
END Use32BitRunTime 
,EM.Package_Path 
,E.Executed_as_name AS Executed_By 

FROM [SSISDB].[internal].[operations] AS O 
INNER JOIN [SSISDB].[internal].[event_messages] AS EM 
ON o.start_time >= @date -- Restrict data by date 
AND EM.operation_id = O.operation_id 

INNER JOIN [SSISDB].[internal].[operation_messages] AS OM
ON EM.operation_id = OM.operation_id 

INNER JOIN [SSISDB].[internal].[executions] AS E 
ON OM.Operation_id = E.EXECUTION_ID 

WHERE OM.Message_Type = 120
AND EM.event_name = 'OnError' 
AND ISNULL(EM.subcomponent_name, '') <> 'SSIS.Pipeline'  and E.Folder_Name = 'RRR'
ORDER BY EM.operation_id DESC 





-----------------------------------------------------------------------------------------------------------------

-- PROVJERA AUDITA
SELECT		audit_id, 
		a.name as audit_name, 
		s.name as server_specification_name,
		d.audit_action_name,
		s.is_state_enabled,
		d.is_group,
		d.audit_action_id,	
		s.create_date,
		s.modify_date
FROM sys.server_audits AS a
JOIN sys.server_audit_specifications AS s
ON a.audit_guid = s.audit_guid
JOIN sys.server_audit_specification_details AS d
ON s.server_specification_id = d.server_specification_id
WHERE s.is_state_enabled = 1

-- List enabled database specifications
SELECT	a.audit_id,
		a.name as audit_name,
		s.name as database_specification_name,
		d.audit_action_name,
		s.is_state_enabled,
		d.is_group,
		s.create_date,
		s.modify_date,
		d.audited_result
FROM sys.server_audits AS a
JOIN sys.database_audit_specifications AS s
ON a.audit_guid = s.audit_guid
JOIN sys.database_audit_specification_details AS d
ON s.database_specification_id = d.database_specification_id
WHERE s.is_state_enabled = 1
-----------------------------------------------------------------------------------------------------------------












-- PROVJERA TABELA        <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
-----------------------------------------------------------------------------------------------------------------
SELECT 
    t.name AS TableName,
    s.name AS SchemaName, f.[name] AS [FileGroup],
    p.rows,
    SUM(a.total_pages) * 8 AS TotalSpaceKB, 
    CAST(ROUND(((SUM(a.total_pages) * 8) / 1024.00), 2) AS NUMERIC(36, 2)) AS TotalSpaceMB,
    SUM(a.used_pages) * 8 AS UsedSpaceKB, 
    CAST(ROUND(((SUM(a.used_pages) * 8) / 1024.00), 2) AS NUMERIC(36, 2)) AS UsedSpaceMB, 
    (SUM(a.total_pages) - SUM(a.used_pages)) * 8 AS UnusedSpaceKB,
    CAST(ROUND(((SUM(a.total_pages) - SUM(a.used_pages)) * 8) / 1024.00, 2) AS NUMERIC(36, 2)) AS UnusedSpaceMB
FROM 
    sys.tables t
INNER JOIN      
    sys.indexes i ON t.object_id = i.object_id
INNER JOIN 
    sys.partitions p ON i.object_id = p.object_id AND i.index_id = p.index_id
INNER JOIN 
    sys.allocation_units a ON p.partition_id = a.container_id
INNER JOIN [sys].[filegroups] f ON a.[data_space_id] = f.[data_space_id]
LEFT OUTER JOIN 
    sys.schemas s ON t.schema_id = s.schema_id
WHERE 
    t.name NOT LIKE 'dt%' 
    AND t.is_ms_shipped = 0
    AND i.object_id > 255 
GROUP BY 
    t.name, s.name, p.rows, f.[name] 
ORDER BY 
    TotalSpaceMB DESC, t.name
-----------------------------------------------------------------------------------------------------------------

-- PROVJERA BAZA        <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
   with fs
    as
    (
        select database_id, type, size * 8.0 / 1024 size, physical_name
        from sys.master_files
        where state_desc = 'ONLINE' -- ONLINE,RESTORING,RECOVERING,RECOVERY_PENDING,SUSPECT,,OFFLINE, DEFUNCT
    )
    select 
        name,
        (select sum(size) from fs where type = 0 and fs.database_id = db.database_id) DataFileSizeMB,
        (select sum(size) from fs where type = 1 and fs.database_id = db.database_id) LogFileSizeMB,
        (select sum(size) from fs where fs.database_id = db.database_id) TotalFileSizeMB
    from sys.databases db
    --order by database_id
    order by TotalFileSizeMB desc

-----------------------------------------------------------------------------------------------------------------










-- PROVJERA FILEGROUP TABELA
-----------------------------------------------------------------------------------------------------------------
SELECT 
   DS.name AS DataSpaceName 
  ,OBJ.name AS ObjectName
  ,d.[physical_name] AS [DatabaseFileName]
  ,AU.type_desc AS AllocationDesc 
  ,AU.total_pages / 128 AS TotalSizeMB 
  ,AU.used_pages / 128 AS UsedSizeMB 
  ,AU.data_pages / 128 AS DataSizeMB 
  ,SCH.name AS SchemaName 
  ,OBJ.type_desc AS ObjectType       
  ,IDX.type_desc AS IndexType 
  ,IDX.name AS IndexName 
FROM sys.data_spaces AS DS 
 INNER JOIN sys.allocation_units AS AU 
     ON DS.data_space_id = AU.data_space_id 
 INNER JOIN sys.partitions AS PA 
     ON (AU.type IN (1, 3)  
         AND AU.container_id = PA.hobt_id) 
        OR 
        (AU.type = 2 
         AND AU.container_id = PA.partition_id) 
 INNER JOIN sys.objects AS OBJ 
     ON PA.object_id = OBJ.object_id 
 INNER JOIN sys.schemas AS SCH 
     ON OBJ.schema_id = SCH.schema_id 
 LEFT JOIN sys.indexes AS IDX 
     ON PA.object_id = IDX.object_id 
	 AND PA.index_id = IDX.index_id 
 INNER JOIN [sys].[database_files] d
 ON IDX.[data_space_id] = d.[data_space_id]
		WHERE OBJECTPROPERTY(OBJ.[object_id], 'IsUserTable') = 1 and OBJ.name in ('TraceLog_old22', 'TraceLog_old23', '2023_GL_TRANSACTIONS') --and IDX.type_desc = 'HEAP'
ORDER BY DS.name 
    ,SCH.name 
    ,OBJ.name 
    ,IDX.name

-----------------------------------------------------------------------------------------------------------------












-----------------------------------------------------------------------------------------------------------------
-- dodavanje kolone kroz skriptu u ssis paket
    int i = 0;
    public override void Input0_ProcessInputRow(Input0Buffer Row)
    {
        Row.ID = i + 1;
        i = i + 1;
    }

-----------------------------------------------------------------------------------------------------------------





-----------------------------------------------------------------------------------------------------------------
INSERT INTO [dbo].[EWS_ZA_BANKU_H]
SELECT *  FROM [dbo].[EWS_ZA_BANKU_temp] 
WHERE [Ugovor] NOT IN
 (SELECT DISTINCT ([Ugovor]) FROM [dbo].[EWS_ZA_BANKU])

IF OBJECT_ID('dbo.EWS_ZA_BANKU_temp','U') IS NOT NULL
drop table dbo.EWS_ZA_BANKU_temp;

BEGIN
SELECT 
   [ID],[MB/JMBG],[Ugovor],[Partner],[Za plaćanje],[Već plaćeno],[Još duguje],[% plać.],[Teč. lista ugov.1],[Valuta ugov.1],[Br. ugovora],[Br. rata - cijeli dug]
  ,[Br. rata/ug. - cijeli dug],[Otv. glavnice],[Otv. kamate],[Otv. marža],[Otv. porez],[Otv. ost. neto],[Otv. porez ost.]
  ,[Šif. partnera],[Bonitet],[Rata],[Trajanje],[Zadnji dospjeli],[Buduća glavnica],[Buduće kamate],[Buduća marža],[Buduće dodatne usluge],[Bud. fin. porez],[Sklopljen],[Grupa],[Predmet]
  ,[Telefon1],[Odobravatelj],[Svih rata],[Status],[Vr. osobe],[Tip financ.],[Ukupno],[Mj.troška],[Bud.potr.bruto],[Dobavljač],[Naziv dobavljača],[Otv. dod.usluge]
  ,[Iznos financiranja],[Otkup],[Dat.otkupa],[Aneks],[Vrijed.VAL],[Kred.ugov.],[Indeks],[Dat.dosp. najstarijeg neplaćenog potraž.],[Maks.br.dana dugovanja],[Datum 1. opomene],[Datum 2. opomene],[Datum 3. opomene],[Kategorija]
  ,[Naziv kateg.],[Iznos man. troškova],[Trajanje ug.],[Teč. lista ugov.2],[Valuta ugov2.],[Tržišna vrijednost neto],[Plaćanje]
  ,[Maturity],[Telefon],[pošta],[Mjesto],[Outstanding],[Vrsta opreme],[NACE],[Cocunut],[TIGER],[ID Skrbnik1],[Skrbnik1]
  ,[ID Skrbnik2],[Skrbnik2],[Kategorija2],[Kategorija2b],[VRIJEME_PREPISA]
   ,cast(HASHBYTES('SHA1',isnull(cast(ltrim(rtrim([MB/JMBG])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Ugovor])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Partner])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Za plaćanje])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Već plaćeno])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Još duguje])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([% plać.])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Teč. lista ugov.1])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Valuta ugov.1])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Br. ugovora])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Br. rata - cijeli dug])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Br. rata/ug. - cijeli dug])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Otv. glavnice])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Otv. kamate])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Otv. marža])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Otv. porez])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Otv. ost. neto])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Otv. porez ost.])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Šif. partnera])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Bonitet])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Rata])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Trajanje])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Zadnji dospjeli])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Buduća glavnica])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Buduće kamate])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Buduća marža])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Buduće dodatne usluge])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Bud. fin. porez])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Sklopljen])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Grupa])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Predmet])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Telefon1])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Odobravatelj])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Svih rata])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Status])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Vr. osobe])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Tip financ.])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Ukupno])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Mj.troška])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Bud.potr.bruto])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Dobavljač])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Naziv dobavljača])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Otv. dod.usluge])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Iznos financiranja])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Otkup])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Dat.otkupa])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Aneks])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Vrijed.VAL])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Kred.ugov.])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Indeks])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Dat.dosp. najstarijeg neplaćenog potraž.])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Maks.br.dana dugovanja])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Datum 1. opomene])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Datum 2. opomene])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Datum 3. opomene])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Kategorija])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Naziv kateg.])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Iznos man. troškova])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Trajanje ug.])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Teč. lista ugov.2])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Valuta ugov2.])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Tržišna vrijednost neto])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Plaćanje])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Maturity])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Telefon])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([pošta])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Mjesto])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Outstanding])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Vrsta opreme])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([NACE])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Cocunut])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([TIGER])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([ID Skrbnik1])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Skrbnik1])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([ID Skrbnik2])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Skrbnik2])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Kategorija2])) as nvarchar),0)
   +isnull(cast(ltrim(rtrim([Kategorija2b])) as nvarchar),0))
    AS VARBINARY(600)) as HashValue 
 INTO dbo.EWS_ZA_BANKU_temp
 FROM [dbo].[EWS_ZA_BANKU] with (nolock) 
 WHERE [MB/JMBG]=[MB/JMBG] AND [UGOVOR]=[UGOVOR];
END

BEGIN
--SET IDENTITY_INSERT [dbo].[EWS_ZA_BANKU_H_test] ON

MERGE INTO [dbo].[EWS_ZA_BANKU_H] A         
 USING (SELECT [ID],[MB/JMBG],[Ugovor],[Partner],[Za plaćanje],[Već plaćeno],[Još duguje],[% plać.],[Teč. lista ugov.1],[Valuta ugov.1],[Br. ugovora],[Br. rata - cijeli dug]
           ,[Br. rata/ug. - cijeli dug],[Otv. glavnice],[Otv. kamate],[Otv. marža],[Otv. porez],[Otv. ost. neto],[Otv. porez ost.]
           ,[Šif. partnera],[Bonitet],[Rata],[Trajanje],[Zadnji dospjeli],[Buduća glavnica],[Buduće kamate],[Buduća marža],[Buduće dodatne usluge],[Bud. fin. porez],[Sklopljen],[Grupa],[Predmet]
           ,[Telefon1],[Odobravatelj],[Svih rata],[Status],[Vr. osobe],[Tip financ.],[Ukupno],[Mj.troška],[Bud.potr.bruto],[Dobavljač],[Naziv dobavljača],[Otv. dod.usluge]
           ,[Iznos financiranja],[Otkup],[Dat.otkupa],[Aneks],[Vrijed.VAL],[Kred.ugov.],[Indeks],[Dat.dosp. najstarijeg neplaćenog potraž.],[Maks.br.dana dugovanja],[Datum 1. opomene],[Datum 2. opomene],[Datum 3. opomene],[Kategorija]
           ,[Naziv kateg.],[Iznos man. troškova],[Trajanje ug.],[Teč. lista ugov.2],[Valuta ugov2.],[Tržišna vrijednost neto],[Plaćanje]
           ,[Maturity],[Telefon],[pošta],[Mjesto],[Outstanding],[Vrsta opreme],[NACE],[Cocunut],[TIGER],[ID Skrbnik1],[Skrbnik1]
           ,[ID Skrbnik2],[Skrbnik2],[Kategorija2],[Kategorija2b],[VRIJEME_PREPISA],[HashValue]
 FROM dbo.EWS_ZA_BANKU_temp with (nolock)) B       
 ON (A.HashValue = B.hashValue)     
 WHEN NOT MATCHED BY TARGET 
 THEN      
 INSERT ([ID],[MB/JMBG],[Ugovor],[Partner],[Za plaćanje],[Već plaćeno],[Još duguje],[% plać.],[Teč. lista ugov.1],[Valuta ugov.1],[Br. ugovora],[Br. rata - cijeli dug]
            ,[Br. rata/ug. - cijeli dug],[Otv. glavnice],[Otv. kamate],[Otv. marža],[Otv. porez],[Otv. ost. neto],[Otv. porez ost.]
            ,[Šif. partnera],[Bonitet],[Rata],[Trajanje],[Zadnji dospjeli],[Buduća glavnica],[Buduće kamate],[Buduća marža],[Buduće dodatne usluge],[Bud. fin. porez],[Sklopljen],[Grupa],[Predmet]
            ,[Telefon1],[Odobravatelj],[Svih rata],[Status],[Vr. osobe],[Tip financ.],[Ukupno],[Mj.troška],[Bud.potr.bruto],[Dobavljač],[Naziv dobavljača],[Otv. dod.usluge]
            ,[Iznos financiranja],[Otkup],[Dat.otkupa],[Aneks],[Vrijed.VAL],[Kred.ugov.],[Indeks],[Dat.dosp. najstarijeg neplaćenog potraž.],[Maks.br.dana dugovanja],[Datum 1. opomene],[Datum 2. opomene],[Datum 3. opomene],[Kategorija]
            ,[Naziv kateg.],[Iznos man. troškova],[Trajanje ug.],[Teč. lista ugov.2],[Valuta ugov2.],[Tržišna vrijednost neto],[Plaćanje]
            ,[Maturity],[Telefon],[pošta],[Mjesto],[Outstanding],[Vrsta opreme],[NACE],[Cocunut],[TIGER],[ID Skrbnik1],[Skrbnik1]
            ,[ID Skrbnik2],[Skrbnik2],[Kategorija2],[Kategorija2b],[VRIJEME_PREPISA],[HashValue])      
 VALUES ([ID],[MB/JMBG],[Ugovor],[Partner],[Za plaćanje],[Već plaćeno],[Još duguje],[% plać.],[Teč. lista ugov.1],[Valuta ugov.1],[Br. ugovora],[Br. rata - cijeli dug]
            ,[Br. rata/ug. - cijeli dug],[Otv. glavnice],[Otv. kamate],[Otv. marža],[Otv. porez],[Otv. ost. neto],[Otv. porez ost.]
            ,[Šif. partnera],[Bonitet],[Rata],[Trajanje],[Zadnji dospjeli],[Buduća glavnica],[Buduće kamate],[Buduća marža],[Buduće dodatne usluge],[Bud. fin. porez],[Sklopljen],[Grupa],[Predmet]
            ,[Telefon1],[Odobravatelj],[Svih rata],[Status],[Vr. osobe],[Tip financ.],[Ukupno],[Mj.troška],[Bud.potr.bruto],[Dobavljač],[Naziv dobavljača],[Otv. dod.usluge]
            ,[Iznos financiranja],[Otkup],[Dat.otkupa],[Aneks],[Vrijed.VAL],[Kred.ugov.],[Indeks],[Dat.dosp. najstarijeg neplaćenog potraž.],[Maks.br.dana dugovanja],[Datum 1. opomene],[Datum 2. opomene],[Datum 3. opomene],[Kategorija]
            ,[Naziv kateg.],[Iznos man. troškova],[Trajanje ug.],[Teč. lista ugov.2],[Valuta ugov2.],[Tržišna vrijednost neto],[Plaćanje]
            ,[Maturity],[Telefon],[pošta],[Mjesto],[Outstanding],[Vrsta opreme],[NACE],[Cocunut],[TIGER],[ID Skrbnik1],[Skrbnik1]
            ,[ID Skrbnik2],[Skrbnik2],[Kategorija2],[Kategorija2b],[VRIJEME_PREPISA],[HashValue]) ;  
 
--SET IDENTITY_INSERT [dbo].[EWS_ZA_BANKU_H_test] OFF
END
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------







-----------------------------------------------------------------------------------------------------------------
-- razlike u tabelama MISO
SELECT [C_LOGID]
  FROM [SEB].[SEB].[C_LOG]
  where c_logid in
  (SELECT 
  [C_LOGID]
  FROM [SEB].[SEB].[C_LOG]
  group by c_logid
  having count(*) > 1)
-----------------------------------------------------------------------------------------------------------------























--PROVJERA QUERIJA HIG CPU   RONALDO <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
-----------------------------------------------------------------------------------------------------------------
SELECT TOP(50) qs.execution_count AS [Execution Count],
(qs.total_logical_reads)/1000.0 AS [Total Logical Reads in ms],
(qs.total_logical_reads/qs.execution_count)/1000.0 AS [Avg Logical Reads in ms],
(qs.total_worker_time)/1000.0 AS [Total Worker Time in ms],
(qs.total_worker_time/qs.execution_count)/1000.0 AS [Avg Worker Time in ms],
(qs.total_elapsed_time)/1000.0 AS [Total Elapsed Time in ms],
(qs.total_elapsed_time/qs.execution_count)/1000.0 AS [Avg Elapsed Time in ms],
qs.creation_time AS [Creation Time]
,t.text AS [Complete Query Text], qp.query_plan AS [Query Plan]
FROM sys.dm_exec_query_stats AS qs WITH (NOLOCK)
CROSS APPLY sys.dm_exec_sql_text(plan_handle) AS t
CROSS APPLY sys.dm_exec_query_plan(plan_handle) AS qp
WHERE t.dbid = DB_ID()
--ORDER BY qs.execution_count DESC OPTION (RECOMPILE);-- for frequently ran query
-- ORDER BY [Avg Logical Reads in ms] DESC OPTION (RECOMPILE);-- for High Disk Reading query
-- ORDER BY [Avg Worker Time in ms] DESC OPTION (RECOMPILE);-- for High CPU query
ORDER BY [Avg Elapsed Time in ms] DESC OPTION (RECOMPILE);-- for Long Running query


SELECT eqs.query_hash, qsp.query_plan_hash, eqs.last_compile_batch_sql_handle,
qsp.query_id, qsp.plan_id,*
FROM sys.query_store_query eqs INNER JOIN sys.query_store_plan qsp
ON eqs.query_id = qsp.query_id
WHERE qsp.plan_id = 2812


-----------------------------------------------------------------------------------------------------------------
select
db_name(database_id) DB,
object_name
(object_id) Obj,
row_lock_count, page_lock_count,
row_lock_count
+ page_lock_count No_Of_Locks,
row_lock_wait_count, page_lock_wait_count,
row_lock_wait_count
+ page_lock_wait_count No_Of_Blocks,
row_lock_wait_in_ms, page_lock_wait_in_ms,
row_lock_wait_in_ms
+ page_lock_wait_in_ms Block_Wait_Time,
index_id
from
sys.dm_db_index_operational_stats(NULL,NULL,NULL,NULL)
WHERE db_name(database_id) = 'KremaTransactPreProd'
order by 
--Block_Wait_Time desc
No_Of_Blocks desc



-- <<<<<<<<<<<<<<<<<<<<<<<<<<<<<< LATENCY CHECK
-----------------------------------------------------------------------------------------------------------------
SELECT
   [ReadLatency] =
        CASE WHEN [num_of_reads] = 0
            THEN 0 ELSE ([io_stall_read_ms] / [num_of_reads]) END,
   [WriteLatency] =
        CASE WHEN [num_of_writes] = 0
            THEN 0 ELSE ([io_stall_write_ms] / [num_of_writes]) END,
   [Latency] =
        CASE WHEN ([num_of_reads] = 0 AND [num_of_writes] = 0)
            THEN 0 ELSE ([io_stall] / ([num_of_reads] + [num_of_writes])) END,
   [Latency Desc] = 
         CASE 
            WHEN ([num_of_reads] = 0 AND [num_of_writes] = 0) THEN 'N/A' 
            ELSE 
               CASE WHEN ([io_stall] / ([num_of_reads] + [num_of_writes])) < 2 THEN 'Excellent'
                    WHEN ([io_stall] / ([num_of_reads] + [num_of_writes])) < 6 THEN 'Very good'
                    WHEN ([io_stall] / ([num_of_reads] + [num_of_writes])) < 11 THEN 'Good'
                    WHEN ([io_stall] / ([num_of_reads] + [num_of_writes])) < 21 THEN 'Poor'
                    WHEN ([io_stall] / ([num_of_reads] + [num_of_writes])) < 101 THEN 'Bad'
                    WHEN ([io_stall] / ([num_of_reads] + [num_of_writes])) < 501 THEN 'Yikes!'
               ELSE 'YIKES!!'
               END 
         END, 
   [AvgBPerRead] =
        CASE WHEN [num_of_reads] = 0
            THEN 0 ELSE ([num_of_bytes_read] / [num_of_reads]) END,
   [AvgBPerWrite] =
        CASE WHEN [num_of_writes] = 0
            THEN 0 ELSE ([num_of_bytes_written] / [num_of_writes]) END,
   [AvgBPerTransfer] =
        CASE WHEN ([num_of_reads] = 0 AND [num_of_writes] = 0)
            THEN 0 ELSE
                (([num_of_bytes_read] + [num_of_bytes_written]) /
                ([num_of_reads] + [num_of_writes])) END,
   LEFT ([mf].[physical_name], 2) AS [Drive],
   DB_NAME ([vfs].[database_id]) AS [DB],
   [mf].[physical_name]
FROM
   sys.dm_io_virtual_file_stats (NULL,NULL) AS [vfs]
   JOIN sys.master_files AS [mf]
   ON [vfs].[database_id] = [mf].[database_id]
      AND [vfs].[file_id] = [mf].[file_id]
-- WHERE [vfs].[file_id] = 2 -- log files
ORDER BY [Latency] DESC
-- ORDER BY [ReadLatency] DESC
-- ORDER BY [WriteLatency] DESC;
GO
-----------------------------------------------------------------------------------------------------------------





-----------------------------------------------------------------------------------------------------------------
--- PROVJERA BLOCK 
SELECT TL.resource_type AS ResType 
      ,TL.resource_description AS ResDescr 
      ,TL.request_mode AS ReqMode 
      ,TL.request_type AS ReqType 
      ,TL.request_status AS ReqStatus 
      ,TL.request_owner_type AS ReqOwnerType 
      ,TAT.[name] AS TransName 
      ,TAT.transaction_begin_time AS TransBegin 
      ,DATEDIFF(ss, TAT.transaction_begin_time, GETDATE()) AS TransDura 
      ,ES.session_id AS S_Id 
      ,ES.login_name AS LoginName 
      ,COALESCE(OBJ.name, PAROBJ.name) AS ObjectName 
      ,PARIDX.name AS IndexName 
      ,ES.host_name AS HostName 
      ,ES.program_name AS ProgramName 
FROM sys.dm_tran_locks AS TL 
     INNER JOIN sys.dm_exec_sessions AS ES 
         ON TL.request_session_id = ES.session_id 
     LEFT JOIN sys.dm_tran_active_transactions AS TAT 
         ON TL.request_owner_id = TAT.transaction_id 
            AND TL.request_owner_type = 'TRANSACTION' 
     LEFT JOIN sys.objects AS OBJ 
         ON TL.resource_associated_entity_id = OBJ.object_id 
            AND TL.resource_type = 'OBJECT' 
     LEFT JOIN sys.partitions AS PAR 
         ON TL.resource_associated_entity_id = PAR.hobt_id 
            AND TL.resource_type IN ('PAGE', 'KEY', 'RID', 'HOBT') 
     LEFT JOIN sys.objects AS PAROBJ 
         ON PAR.object_id = PAROBJ.object_id 
     LEFT JOIN sys.indexes AS PARIDX 
         ON PAR.object_id = PARIDX.object_id 
            AND PAR.index_id = PARIDX.index_id 
WHERE TL.resource_database_id  = DB_ID() 
      AND ES.session_id <> @@Spid -- Exclude "my" session 
      -- optional filter  
      AND TL.request_mode <> 'S' -- Exclude simple shared locks 
ORDER BY TL.resource_type 
        ,TL.request_mode 
        ,TL.request_type 
        ,TL.request_status 
        ,ObjectName 
        ,ES.login_name;
-----------------------------------------------------------------------------------------------------------------















--- PROVJERA INDEXA
-----------------------------------------------------------------------------------------------------------------

SELECT
 object_name(i.object_id) AS object_name,
 i.name AS index_name, s.index_id,
 user_seeks + user_scans + user_lookups AS user_reads,
 system_seeks + system_scans + system_lookups AS
system_reads,
 user_updates,
 system_updates
FROM sys.dm_db_index_usage_stats s JOIN sys.indexes i
 ON s.index_id = i.index_id AND s.object_id = i.object_id
WHERE s.database_id = db_id()
 AND i.type <> 0 and i.name like '%IX%'
ORDER BY user_reads DESC

-----------------------------------------------------------------------------------------------------------------

select i.[name] as index_name,
    substring(column_names, 1, len(column_names)-1) as [columns],
    case when i.[type] = 1 then 'Clustered index'
        when i.[type] = 2 then 'Nonclustered unique index'
        when i.[type] = 3 then 'XML index'
        when i.[type] = 4 then 'Spatial index'
        when i.[type] = 5 then 'Clustered columnstore index'
        when i.[type] = 6 then 'Nonclustered columnstore index'
        when i.[type] = 7 then 'Nonclustered hash index'
        end as index_type,
    case when i.is_unique = 1 then 'Unique'
        else 'Not unique' end as [unique],
    schema_name(t.schema_id) + '.' + t.[name] as table_view, 
    case when t.[type] = 'U' then 'Table'
        when t.[type] = 'V' then 'View'
        end as [object_type]
from sys.objects t
    inner join sys.indexes i
        on t.object_id = i.object_id
    cross apply (select col.[name] + ', '
                    from sys.index_columns ic
                        inner join sys.columns col
                            on ic.object_id = col.object_id
                            and ic.column_id = col.column_id
                    where ic.object_id = t.object_id
                        and ic.index_id = i.index_id
                            order by key_ordinal
                            for xml path ('') ) D (column_names)
where t.is_ms_shipped <> 1 and i.[name] like 'IX%'
order by i.[name]








SELECT  1
    ja.job_id,  
    j.name AS job_name,  
    ja.start_execution_date,        
    ISNULL(last_executed_step_id,0)+1 AS current_executed_step_id
FROM msdb.dbo.sysjobactivity ja   
LEFT JOIN msdb.dbo.sysjobhistory jh ON ja.job_history_id = jh.instance_id  
JOIN msdb.dbo.sysjobs j ON ja.job_id = j.job_id  
JOIN msdb.dbo.sysjobsteps js  
    ON ja.job_id = js.job_id  
    AND ISNULL(ja.last_executed_step_id,0)+1 = js.step_id  
WHERE  
  ja.session_id = (  
    SELECT TOP 1 session_id FROM msdb.dbo.syssessions ORDER BY agent_start_date DESC  
  )  
AND start_execution_date is not null  
AND stop_execution_date is null
and j.name = '$DBA - FULL - DatabaseBackup ON DEMAND T24-ADHOC'; 


-- PROVJERA LOCKOVA
-----------------------------------------------------------------------------------------------------------------
SELECT * 
FROM sys.dm_exec_requests
WHERE blocking_session_id <> 0;
GO

SELECT session_id, wait_duration_ms, wait_type, blocking_session_id 
FROM sys.dm_os_waiting_tasks 
WHERE blocking_session_id <> 0
GO
-----------------------------------------------------------------------------------------------------------------





--PROVJERA FOREIGN KEYA NA TABELAMA
-----------------------------------------------------------------------------------------------------------------
SELECT 
   OBJECT_NAME(f.parent_object_id) TableName,
   COL_NAME(fc.parent_object_id,fc.parent_column_id) ColName
FROM 
   sys.foreign_keys AS f
INNER JOIN 
   sys.foreign_key_columns AS fc 
      ON f.OBJECT_ID = fc.constraint_object_id
INNER JOIN 
   sys.tables t 
      ON t.OBJECT_ID = fc.referenced_object_id
WHERE 
   OBJECT_NAME (f.referenced_object_id) = 'accountsbpms'
-----------------------------------------------------------------------------------------------------------------






-- PROVJERA password change
-----------------------------------------------------------------------------------------------------------------
SELECT name, LOGINPROPERTY([name], 'PasswordLastSetTime') AS 'PasswordChanged'
FROM sys.sql_logins  
WHERE LOGINPROPERTY([name], 'PasswordLastSetTime') < DATEADD(dd, -60, GETDATE());
-----------------------------------------------------------------------------------------------------------------








--XML PRETVARANJE TABELE
-----------------------------------------------------------------------------------------------------------------
    
if object_id('tempdb..#FBNK_RE_CRF_RBBHGL') is not null drop table #FBNK_RE_CRF_RBBHGL;

DECLARE @eod_date smalldatetime; --#v3

SELECT @eod_date = convert(smalldatetime, convert(date, id2, 23))
FROM test.dbo.sif_sifarnika WITH(nolock)
WHERE grupa = 'JOB_PARAMETERS' and id = 'EOD_DATE'; --#v3


select --prvo razbijamo kolonu recid
    recid,
	"SEQ_NO" = nullif(JSON_VALUE(JS1,'$[0]'),'') --nullif() optional otherwise empty string
    ,"CONTRACT_ID" = nullif(JSON_VALUE(JS1,'$[3]'),'')
    ,"ASSET_TYPE" = nullif(JSON_VALUE(JS1,'$[4]'),''),
	a.XMLRECORD as THE_RECORD
    ,"CURRENCY" = nullif(JSON_VALUE(JS,'$[0]'),'') --nullif() optional otherwise empty string
    ,"DESC_1" = nullif(JSON_VALUE(JS,'$[1]'),'')
    ,"DESC_2" = nullif(JSON_VALUE(JS,'$[2]'),'')
    ,"LOCAL_BALANCE" = nullif(JSON_VALUE(JS,'$[3]'),'')
    ,"FOREIGN_BALANCE" = nullif(JSON_VALUE(JS,'$[4]'),'')
    ,"MATURITY_DATE" = nullif(JSON_VALUE(JS,'$[5]'),'')
    ,"SCHED_LOCAL_BAL" = nullif(JSON_VALUE(JS,'$[6]'),'')
    ,"SCHED_FCY_BAL" = nullif(JSON_VALUE(JS,'$[7]'),'')
    ,"CONSOL_KEY" = nullif(JSON_VALUE(JS,'$[8]'),'')
    ,"CUSTOMER_NO" = nullif(JSON_VALUE(JS,'$[9]'),'')
    ,"DEAL_BALANCE" = nullif(JSON_VALUE(JS,'$[10]'),'')
    ,"DEAL_LCY_BALANCE" = nullif(JSON_VALUE(JS,'$[11]'),'')
    ,"DEAL_RATE" = nullif(JSON_VALUE(JS,'$[12]'),'')
    ,"DEAL_VALUE_DATE" = nullif(JSON_VALUE(JS,'$[13]'),'')
    ,"DEAL_MAT_DATE" = nullif(JSON_VALUE(JS,'$[14]'),'')
    ,"DEAL_REMAINING_DAYS" = nullif(JSON_VALUE(JS,'$[15]'),'')
    ,"INT_RATE_BASIS" = nullif(JSON_VALUE(JS,'$[16]'),'')
    ,"LINE_TOTAL" = nullif(JSON_VALUE(JS,'$[17]'),'')
    ,"DEBIT_MVMT" = nullif(JSON_VALUE(JS,'$[18]'),'')
    ,"DEBIT_LCY_MVMT" = nullif(JSON_VALUE(JS,'$[19]'),'')
    ,"CREDIT_MVMT" = nullif(JSON_VALUE(JS,'$[21]'),'')
    ,"CREDIT_LCY_MVMT" = nullif(JSON_VALUE(JS,'$[22]'),''),
   	GETDATE() AS "REPORT_TIME",
	@eod_date as EOD_DATE
into #FBNK_RE_CRF_RBBHGL
from dbo.FBNK_RE_CRF_RBBHGL A 
	cross apply (values ('["'+replace(string_escape([Recid],'json'),'*','","')+'"]') ) B(JS1)
	Cross Apply (values ('["'+replace(string_escape([XMLRECORD],'json'),nchar(63742),'","')+'"]') ) c(JS);--1:27 603506
-----------------------------------------------------------------------------------------------------------------





-- PROVJERA RESTORE LOG ILI LSN LAST BACKUP
-----------------------------------------------------------------------------------------------------------------
SELECT 
s.database_name,
CAST(CAST(s.backup_size / 1000000 AS INT) AS VARCHAR(14)) + ' ' + 'MB' AS bkSize,
CAST(DATEDIFF(second, s.backup_start_date,
s.backup_finish_date) AS VARCHAR(4)) + ' ' + 'Seconds' TimeTaken,
s.backup_start_date,
CAST(s.first_lsn AS VARCHAR(50)) AS first_lsn,
CAST(s.last_lsn AS VARCHAR(50)) AS last_lsn,
CAST(s.database_backup_lsn AS VARCHAR(50)) AS database_backup_lsn,
CAST(s.checkpoint_lsn AS VARCHAR(50)) AS checkpoint_lsn,
CASE s.[type] WHEN 'D' THEN 'Full'
WHEN 'I' THEN 'Differential'
WHEN 'L' THEN 'Transaction Log'
END AS BackupType,
s.recovery_model
FROM msdb.dbo.backupset s
INNER JOIN msdb.dbo.backupmediafamily m ON s.media_set_id = m.media_set_id
WHERE s.database_name = DB_NAME() 
ORDER BY backup_start_date DESC, backup_finish_date
GO
-----------------------------------------------------------------------------------------------------------------









--provjera cpu RAMO
-----------------------------------------------------------------------------------------------------------------
select * from sys.dm_os_sys_info

	SELECT *
FROM sys.dm_os_tasks sdot
LEFT JOIN sys.dm_os_schedulers sdos
ON sdot.scheduler_id = sdos.scheduler_id
WHERE session_id is not null
ORDER BY sdot.session_id

DECLARE @cputime_threshold_microsec INT = 200*1000
DECLARE @execution_count INT = 1000

SELECT 
    qs.total_worker_time/1000 total_cpu_time_ms,
      qs.max_worker_time/1000 max_cpu_time_ms,
      (qs.total_worker_time/1000)/execution_count average_cpu_time_ms,
      qs.execution_count,
     q.[text]
FROM
  sys.dm_exec_query_stats qs
   CROSS APPLY sys.dm_exec_sql_text(plan_handle) AS q
WHERE (qs.total_worker_time/execution_count > @cputime_threshold_microsec  
       OR        qs.max_worker_time > @cputime_threshold_microsec )

-----------------------------------------------------------------------------------------------------------------












-----------------------------------------------------------------------------------------------------------------
--POLYBASE ORACLE:
exec sp_configure @configname = 'polybase enabled', @configvalue = 1;
--POLYBASE CHECK POWERSHELL
Get-Process mpdwsvc -FileVersionInfo | Format-Table -AutoSize
SELECT SERVERPROPERTY ('IsPolyBaseInstalled') AS IsPolyBaseInstalled;
-----------------------------------------------------------------------------------------------------------------
use  polyORA
go
CREATE MASTER KEY ENCRYPTION BY PASSWORD = '1C0nn3P0ly0RA@'

--provjeriti da li spasava se:
SELECT * FROM sys.symmetric_keys
SELECT * FROM sys.database_scoped_credentials
SELECT * FROM sys.external_data_sources

--provjera query prema devcards, cmsdev
select * from OPENQUERY(BO, 'SELECT * FROM ISSUING2_0.PRPA_ACCNT_AC WHERE ROWNUM < 6')

--cred i data source za oracle
--CREATE DATABASE SCOPED CREDENTIAL OracleCredentialTest WITH IDENTITY = 'RBBH_CORE', Secret = 'ACU=P9VevjDTszTn';
CREATE DATABASE SCOPED CREDENTIAL ORACLE_RBBH_CORE WITH IDENTITY = 'RBBH_CORE', Secret = 'ACU=P9VevjDTszTn';

--cred na bazi prema oracle
--CREATE EXTERNAL DATA SOURCE [ORACLE]  WITH ( location = 'Oracle://10.234.149.70:1521',CREDENTIAL = OracleCredentialTest)
CREATE EXTERNAL DATA SOURCE [ORACLE_BO]  WITH ( location = 'Oracle://10.234.149.70:1521',CREDENTIAL = ORACLE_RBBH_CORE)

CREATE EXTERNAL TABLE [DBO].[PRPA_ACC_PARAM] (
[ACCOUNT_NO] DECIMAL(20) NOT NULL, 
[CARD_ACCT] VARCHAR(136) COLLATE Latin1_General_100_BIN2_UTF8 NOT NULL, 
[CYCLE] VARCHAR(40) COLLATE Latin1_General_100_BIN2_UTF8 NOT NULL, 
[MIN_BAL] DECIMAL(10) NOT NULL, 
[COND_SET] VARCHAR(12) COLLATE Latin1_General_100_BIN2_UTF8 NOT NULL, 
[STATUS] VARCHAR(4) COLLATE Latin1_General_100_BIN2_UTF8 NOT NULL, 
[CCY] VARCHAR(12) COLLATE Latin1_General_100_BIN2_UTF8 NOT NULL, 
[STAT_CHANGE] VARCHAR(4) COLLATE Latin1_General_100_BIN2_UTF8 NOT NULL, 
[CRD] DECIMAL(12) NOT NULL, 
[STOP_DATE] DATETIME2(0), 
[PAY_CODE] VARCHAR(4) COLLATE Latin1_General_100_BIN2_UTF8 NOT NULL, 
[PAY_FREQ] VARCHAR(4) COLLATE Latin1_General_100_BIN2_UTF8 NOT NULL, 
[CALCUL_MODE] VARCHAR(8) COLLATE Latin1_General_100_BIN2_UTF8 NOT NULL, 
[PAY_AMNT] DECIMAL(12) NOT NULL, 
[PAY_INTR] DECIMAL(6, 2) NOT NULL, 
[LIM_AMNT] DECIMAL(12) NOT NULL, 
[LIM_INTR] DECIMAL(6, 2) NOT NULL, 
[STA_COMENT] VARCHAR(160) COLLATE Latin1_General_100_BIN2_UTF8, 
[AUTH_BONUS] DECIMAL(12), 
[AB_EXPIRITY] DATETIME2(0), 
[DEPOSIT] DECIMAL(12), 
[DEPOSIT_COMENT] VARCHAR(120) COLLATE Latin1_General_100_BIN2_UTF8, 
[CREATED_DATE] DATETIME2(0) NOT NULL, 
[PROC_ID] DECIMAL(14) NOT NULL, 
[BANKC] VARCHAR(8) COLLATE Latin1_General_100_BIN2_UTF8, 
[USRID] VARCHAR(24) COLLATE Latin1_General_100_BIN2_UTF8 NOT NULL, 
[CTIME] DATETIME2(0) NOT NULL, 
[GROUPC] VARCHAR(8) COLLATE Latin1_General_100_BIN2_UTF8 NOT NULL, 
[BANK_C] VARCHAR(8) COLLATE Latin1_General_100_BIN2_UTF8 NOT NULL, 
[DEPOSIT_ACCOUNT] VARCHAR(80) COLLATE Latin1_General_100_BIN2_UTF8, 
[ATM_LIMIT] DECIMAL(12) NOT NULL, 
[NON_REDUCE_BAL] DECIMAL(12) NOT NULL, 
[ADJUST_FLAG] VARCHAR(4) COLLATE Latin1_General_100_BIN2_UTF8, 
[MESSAGE] VARCHAR(480) COLLATE Latin1_General_100_BIN2_UTF8, 
[U_COD7] VARCHAR(12) COLLATE Latin1_General_100_BIN2_UTF8, 
[U_COD8] VARCHAR(24) COLLATE Latin1_General_100_BIN2_UTF8, 
[UFIELD_5] VARCHAR(80) COLLATE Latin1_General_100_BIN2_UTF8, 
[U_FIELD6] VARCHAR(100) COLLATE Latin1_General_100_BIN2_UTF8, 
[IN_FILE_NUM] DECIMAL(20), 
[B_BR_ID] DECIMAL(7), 
[AGR_AMOUNT] DECIMAL(12), 
[DEP_EXP_DATE] DATETIME2(0), 
[DEP_OPEN_F] VARCHAR(4) COLLATE Latin1_General_100_BIN2_UTF8, 
[DEP_FRONT_F] VARCHAR(4) COLLATE Latin1_General_100_BIN2_UTF8, 
[CRD_CHANGE_DATE] DATETIME2(0), 
[CRD_EXPIRY] DATETIME2(0), 
[DEP_OPER_ACCT] DECIMAL(20), 
[DEP_OPER_ACCTB] DECIMAL(1), 
[DEP_OPER_BACCT] VARCHAR(136) COLLATE Latin1_General_100_BIN2_UTF8, 
[OVERDR_RECALC_DATE] DATETIME2(0), 
[OVERDR_LAST_TRY_DATE] DATETIME2(0), 
[OVERDR_FLAG] VARCHAR(4) COLLATE Latin1_General_100_BIN2_UTF8, 
[SHADOW_LIMIT] DECIMAL(10), 
[INTEREST_CALC_FLAG] VARCHAR(4) COLLATE Latin1_General_100_BIN2_UTF8 NOT NULL, 
[AGREEMENT_KEY] DECIMAL(15), 
[IN_ORDER_NUMBER] DECIMAL(20), 
[OPEN_INSTL] FLOAT(53), 
[INSTL_LINE] FLOAT(53), 
[INSTL_CONDSET] VARCHAR(12) COLLATE Latin1_General_100_BIN2_UTF8, 
[PAY_OFF_STATUS] DECIMAL(1) NOT NULL, 
[RELINK_DATE] DATETIME2(0), 
[KS_DATE] DATETIME2(0), 
[FIRST_TRX_DATE] DATETIME2(0), 
[ALGO] VARCHAR(4) COLLATE Latin1_General_100_BIN2_UTF8, 
[SEIZURE_FEE] DECIMAL(3), 
[KS_DATE_FLAG] VARCHAR(4) COLLATE Latin1_General_100_BIN2_UTF8, 
[LAST_TRX_DATE] DATETIME2(0), 
[AB_TYPE] VARCHAR(40) COLLATE Latin1_General_100_BIN2_UTF8, 
[AB_FEE] DECIMAL(10), 
[ACTION_FEE] DECIMAL(10), 
[ACTION_START] DATETIME2(0), 
[ACTION_END] DATETIME2(0))
   WITH (
    LOCATION='[cmsdev.TODO.rbbh].ISSUING2_0.PRPA_ACC_PARAM',
    DATA_SOURCE= ORACLE
   );
   
   SELECT * FROM NOAH_ARHIVA.RI_TR_ENTITY_ID fetch  first 10 rows only;
 -----------------------------------------------------------------------------------------------------------------
 
 
 
 
 
 SELECT cpu_count AS [Logical CPU Count], hyperthread_ratio AS Hyperthread_Ratio,
cpu_count/hyperthread_ratio AS Physical_CPU_Count,
--physical_memory_in_bytes/1048576 AS Physical_Memory_in_MB,
sqlserver_start_time, affinity_type_desc -- (affinity_type_desc is only in 2008 R2)
FROM sys.dm_os_sys_info

 
 
 
 
 
 
 
 
 
 
 
 --FILEGROUPS OFFLINE FILGROUP
  -----------------------------------------------------------------------------------------------------------------
ALTER DATABASE [tester] MODIFY FILE (NAME = 'tester2t', OFFLINE)

use [master]
RESTORE DATABASE [tester] FILEGROUP = 'data2' WITH RECOVERY

USE [Archives]
Select DS.name, state_desc
from sys.data_spaces DS
inner join sys.database_files F
   on F.data_space_id = DS.data_space_id
order by DS.name, F.file_id
 -----------------------------------------------------------------------------------------------------------------
 -- FILE GROUP PROVJERA:
 SELECT [databasefile].NAME      AS [FileName],
  [filegroup].NAME       AS [File_Group_Name],
  [filegroup].type_desc,
  physical_name [Data File Location],
  size / 128    AS [Size_in_MB],
  state_desc    [State of FILE],
  growth        [Data file growth]
FROM   sys.database_files [databasefile]
  INNER JOIN sys.filegroups [filegroup]
          ON [databasefile].data_space_id = [filegroup].data_space_id  
 --PROVJERA FILE GROUPA LOKACIJA
-----------------------------------------------------------------------------------------------------------------
SELECT OBJECT_NAME(i.[object_id]) AS [ObjectName]
	,i.[index_id] AS [IndexID]
	,i.[name] AS [IndexName]
	,i.[type_desc] AS [IndexType]
	,i.[data_space_id] AS [DatabaseSpaceID]
	,f.[name] AS [FileGroup]
	,d.[physical_name] AS [DatabaseFileName]
FROM [sys].[indexes] i
INNER JOIN [sys].[filegroups] f
	ON f.[data_space_id] = i.[data_space_id]
INNER JOIN [sys].[database_files] d
	ON f.[data_space_id] = d.[data_space_id]
INNER JOIN [sys].[data_spaces] s
	ON f.[data_space_id] = s.[data_space_id]
WHERE OBJECTPROPERTY(i.[object_id], 'IsUserTable') = 1
ORDER BY OBJECT_NAME(i.[object_id])
	,f.[name]
	,i.[data_space_id]
GO
-----------------------------------------------------------------------------------------------------------------
 
 
 
 
 






  -----------------------------------------------------------------------------------------------------------------
Kreiranje partition funkcije za godine 2024-2028.:
     CREATE PARTITION FUNCTION [hist_archive] (datetime2(7))
                                                                        AS RANGE RIGHT FOR VALUES
                                                                                                (N'2024-12-31T00:00:00.000', N'2025-12-31T00:00:00.000', N'2026-12-31T00:00:00.000',
                                                                              N'2027-12-31T00:00:00.000',N'2028-12-31T00:00:00.000')

    CREATE PARTITION SCHEME [hist_archive]
                                                                        AS PARTITION [hist_archive]
                                                                        TO (HIST_24, HIST_25, HIST_26, HIST_27, HIST_28, HIST_27)

Kreiranje tabele:
CREATE TABLE [dbo].[customerInfo](
                        [customerInfoID] [int] IDENTITY(1,1) NOT NULL,
                        [CustomerID] [varchar](10) NULL,
                        [JMBG] [varchar](16) NULL,
                        [name] [nvarchar](100) NULL,
                        [surname] [nvarchar](100) NULL,
                        [state] [nvarchar](100) NULL,
                        [postalCode] [varchar](10) NULL,
                        [city] [nvarchar](100) NULL,
                        [street] [nvarchar](100) NULL,
                        [employmentCode] [char](1) NULL,
                        [createdDateTime] [datetime2](7) NOT NULL default getdate(),
CONSTRAINT [PK_cid] PRIMARY KEY CLUSTERED 
(
                         [createdDateTime] ASC,
                         [customerInfoID] ASC
)) ON [hist_archive]([createdDateTime])
  -----------------------------------------------------------------------------------------------------------------
 
 
 
 
 
 
 
 
  
 -- QUERY UPITI RAMO
  -----------------------------------------------------------------------------------------------------------------
--Running Transaction trenutne:
use master
SELECT
SPID,ER.percent_complete,
CAST(((DATEDIFF(s,start_time,GetDate()))/3600) as varchar) + ' hour(s), '
+ CAST((DATEDIFF(s,start_time,GetDate())%3600)/60 as varchar) + 'min, '
+ CAST((DATEDIFF(s,start_time,GetDate())%60) as varchar) + ' sec' as running_time,
CAST((estimated_completion_time/3600000) as varchar) + ' hour(s), '
+ CAST((estimated_completion_time %3600000)/60000 as varchar) + 'min, '
+ CAST((estimated_completion_time %60000)/1000 as varchar) + ' sec' as est_time_to_go,
DATEADD(second,estimated_completion_time/1000, getdate()) as est_completion_time,
ER.command,ER.blocking_session_id, SP.DBID,LASTWAITTYPE,
DB_NAME(SP.DBID) AS DBNAME,
SUBSTRING(est.text, (ER.statement_start_offset/2)+1,
((CASE ER.statement_end_offset
WHEN -1 THEN DATALENGTH(est.text)
ELSE ER.statement_end_offset
END - ER.statement_start_offset)/2) + 1) AS QueryText,
TEXT,CPU,HOSTNAME,LOGIN_TIME,LOGINAME,
SP.status,PROGRAM_NAME,NT_DOMAIN, NT_USERNAME
FROM SYSPROCESSES SP
INNER JOIN
sys.dm_exec_requests ER
ON sp.spid = ER.session_id
CROSS APPLY SYS.DM_EXEC_SQL_TEXT(er.sql_handle) EST


--Longest running query:
SELECT DISTINCT TOP 3
t.TEXT QueryName,
s.execution_count AS ExecutionCount,
s.max_elapsed_time AS MaxElapsedTime,
--ISNULL(s.total_elapsed_time / s.execution_count, 0) AS AvgElapsedTime,
s.creation_time AS LogCreatedOn--,
--ISNULL(s.execution_count / DATEDIFF(s, s.creation_time, GETDATE()), 0) AS FrequencyPerSec
FROM sys.dm_exec_query_stats s
CROSS APPLY sys.dm_exec_sql_text( s.sql_handle ) t
ORDER BY
s.max_elapsed_time DESC
GO

--ansaction causing log space fill most 
SELECT tst.[session_id],
s.[login_name] AS [Login Name],
DB_NAME (tdt.database_id) AS [Database],
tdt.[database_transaction_begin_time] AS [Begin Time],
tdt.[database_transaction_log_record_count] AS [Log Records],
tdt.[database_transaction_log_bytes_used] AS [Log Bytes Used],
tdt.[database_transaction_log_bytes_reserved] AS [Log Bytes Rsvd],
SUBSTRING(st.text, (r.statement_start_offset/2)+1,
((CASE r.statement_end_offset
WHEN -1 THEN DATALENGTH(st.text)
ELSE r.statement_end_offset
END - r.statement_start_offset)/2) + 1) AS statement_text,
st.[text] AS [Last T-SQL Text],
qp.[query_plan] AS [Last Plan]
FROM sys.dm_tran_database_transactions tdt
JOIN sys.dm_tran_session_transactions tst
ON tst.[transaction_id] = tdt.[transaction_id]
JOIN sys.[dm_exec_sessions] s
ON s.[session_id] = tst.[session_id]
JOIN sys.dm_exec_connections c
ON c.[session_id] = tst.[session_id]
LEFT OUTER JOIN sys.dm_exec_requests r
ON r.[session_id] = tst.[session_id]
CROSS APPLY sys.dm_exec_sql_text (c.[most_recent_sql_handle]) AS st
OUTER APPLY sys.dm_exec_query_plan (r.[plan_handle]) AS qp
where DB_NAME (tdt.database_id) = 'tempdb'
ORDER BY [Log Bytes Used] DESC;
  -----------------------------------------------------------------------------------------------------------------
  
  
  
  
  
  
  
  -- SNAPSHOT ISOLATION 
  SELECT name
, s.snapshot_isolation_state
, snapshot_isolation_state_desc
, is_read_committed_snapshot_on
FROM sys.databases s
  
  
  
  
  
  
  
  
  
  
  
--PROMJENE I REPAIR BAZE
-----------------------------------------------------------------------------------------------------------------
ALTER DATABASE [MaintDB] SET EMERGENCY;
GO
ALTER DATABASE [MaintDB] set single_user
GO
DBCC CHECKDB ([MaintDB], REPAIR_ALLOW_DATA_LOSS) WITH ALL_ERRORMSGS;
GO
ALTER DATABASE [MaintDB] set multi_user
GO
-----------------------------------------------------------------------------------------------------------------











--DOBRA UPUSTVA SSIS:
-- https://drive.google.com/drive/folders/1ukcO6ZLGdeGzwyUipZbq0DQF7ycV8_xc
/*
"INSERT INTO [DBRANCH_log] VALUES 
 ('55',convert(varchar(10), GETDATE(),126), 'TABELA2', ?);"
"INSERT INTO [DBRANCH_log] VALUES 
 ('55',convert(varchar(10), GETDATE(),126), 'TABELA2', '"+(DT_WSTR,12)@[User::brojan]+"');"
 */
 
 
 
 
 
 
 
 
 
 
 EXECUTE dbo.pCheckJobStatusForSuccess @jobName = 'Tieto_Data_Load';
 
 
 
 
 
 
 
 
 
 
 
 -- POSALJI MAIL SA HTML TABELU
 -----------------------------------------------------------------------------------------------------------------
 	BEGIN TRY
			IF (EXISTS
			(SELECT * FROM test..kursl with (nolock) WHERE datum = convert(varchar(10), getdate(),126) + ' 00:00:00')) 
		BEGIN
	        DECLARE @tableHTML  NVARCHAR(MAX) ;  
			DECLARE @subjectMail varchar(100)='Kursna lista na dan ' + CONVERT(char(10),GETDATE(),104) + ' u ' + SUBSTRING(convert(varchar, getdate(), 24),0,6)
						SET @tableHTML =  
				N'<head>
			<style>
			table, th, td {
				border: 2px solid #474747;
				border-collapse: collapse;
			}
			th, td {
				padding: 3px;
			}
			table {
				background-color: #FFFFFF;
			}
			table th {
				border: 2px solid #474747;
				background-color: #FFF788; 
				color: #474747;
			}
			</style>
			</head>' +
				N'<p>' + 'Postovani, ' + @subjectMail + ' :'+ '</p>' +  
				N'<table>' +  
				N'<tr><th>datum</th><th>datums</th>' +  
				N'<th>sifval</th><th>kupovni</th><th>srednji</th>' +   
				N'<th>prodajni</th><th>broj_kl</th>' + 
				N'<th>jedinica</th></tr>' + 
				CAST ( ( SELECT td = datum, '',  
								td = datums, '',  
								td = sifval, '',
								td = kupovni, '',
								td = srednji, '',
								td = prodajni, '', 
								td = broj_kl, '',
								td = jedinica, ''					              
				FROM test..kursl with (nolock) WHERE datum = convert(varchar(10), getdate(),126) + ' 00:00:00'
						  FOR XML PATH('tr'), TYPE   
				) AS NVARCHAR(MAX) ) +
				N'</table>' ; 
			--salji mail
			EXEC msdb.dbo.sp_send_dbmail
						@profile_name = 'MAIL',
                                                                                                @recipients = 'amar.abaz@TAMBURAgroup.ba',
						@body = @tableHTML,
						@subject = @subjectMail,
						@attach_query_result_as_file=0,
						@query_result_separator='',
						@exclude_query_output=0,
						@query_result_header=1,
	    				@body_format = 'HTML';		
		END

		ELSE 
                 BEGIN
				 DECLARE @subjectMail2 varchar(100)='Kursna lista na dan ' + CONVERT(char(10),GETDATE(),104) + ' u ' + SUBSTRING(convert(varchar, getdate(), 24),0,6)
                     EXEC msdb.dbo.sp_send_dbmail
                        @profile_name = 'MAIL',
                        @recipients = 'amar.abaz@TAMBURAgroup.ba',
                        @body = 'Nema rezultata kursne liste za ovaj dan.',
                        @subject = @subjectMail2;
                 END

		END TRY  
		BEGIN CATCH  
					select @@ERROR
		END CATCH
 -----------------------------------------------------------------------------------------------------------------
 
 
 
 -- POSALJI MAIL EXCEL TABELU:
  -----------------------------------------------------------------------------------------------------------------
 declare @qry varchar(8000)
declare @column1name varchar(50)
-- Create the column name with the instrucation in a variable
SET @Column1Name = '[sep=,' + CHAR(13) + CHAR(10) + 'id]'
 
-- Create the query, concatenating the column name as an alias
select @qry='set nocount on;select id ' + @column1name + 
             ' ,namedb,namedb from [maintdb].[dbo].[check_db]'
 
-- Send the e-mail with the query results in attach
exec msdb.dbo.sp_send_dbmail 
@profile_name = 'MAIL',
@recipients='amar.abaz@TAMBURAgroup.ba',
@query=@qry,
@subject='Client list',
@attach_query_result_as_file = 1,
@query_attachment_filename = 'result.csv',
@query_result_separator=',',@query_result_width =32767,
@query_result_no_padding=1
 -----------------------------------------------------------------------------------------------------------------
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 --PROVJERA JOB DA LI TRAJE:
  -----------------------------------------------------------------------------------------------------------------
 SELECT
ja.job_id,
j.name AS job_name,
ja.start_execution_date,      
ISNULL(last_executed_step_id,0)+1 AS current_executed_step_id,
Js.step_name
FROM msdb.dbo.sysjobactivity ja 
LEFT JOIN msdb.dbo.sysjobhistory jh 
ON ja.job_history_id = jh.instance_id
JOIN msdb.dbo.sysjobs j 
ON ja.job_id = j.job_id
JOIN msdb.dbo.sysjobsteps js
ON ja.job_id = js.job_id
AND ISNULL(ja.last_executed_step_id,0)+1 = js.step_id
WHERE ja.session_id = (SELECT TOP 1 session_id FROM msdb.dbo.syssessions ORDER BY agent_start_date DESC)
AND run_requested_date IS NOT NULL 
AND start_execution_date >= GETDATE()-5
AND j.name = '1'
AND stop_execution_date is null;
   -----------------------------------------------------------------------------------------------------------------
 
 
 
 
 
 
 
 
 
 
 
 -- PROVJERA TRIGGERA TRIGGER 
  -----------------------------------------------------------------------------------------------------------------
 select trg.name as trigger_name,
    schema_name(tab.schema_id) + '.' + tab.name as [table],
    case when is_instead_of_trigger = 1 then 'Instead of'
        else 'After' end as [activation],
    (case when objectproperty(trg.object_id, 'ExecIsUpdateTrigger') = 1
                then 'Update ' else '' end 
    + case when objectproperty(trg.object_id, 'ExecIsDeleteTrigger') = 1
                then 'Delete ' else '' end
    + case when objectproperty(trg.object_id, 'ExecIsInsertTrigger') = 1
                then 'Insert' else '' end
    ) as [event],
    case when trg.parent_class = 1 then 'Table trigger'
        when trg.parent_class = 0 then 'Database trigger' 
    end [class], 
    case when trg.[type] = 'TA' then 'Assembly (CLR) trigger'
        when trg.[type] = 'TR' then 'SQL trigger' 
        else '' end as [type],
    case when is_disabled = 1 then 'Disabled'
        else 'Active' end as [status],
    object_definition(trg.object_id) as [definition]
from sys.triggers trg
    left join sys.objects tab
        on trg.parent_id = tab.object_id
order by trg.name;
 -----------------------------------------------------------------------------------------------------------------
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 -- PROVJERA PASSWORD LEAK 
  -----------------------------------------------------------------------------------------------------------------
-- check password leak
CREATE TABLE tbCheck(
	CheckValue NVARCHAR(128)
)
----Drop table for testing
/*
DROP TABLE tbCheck
*/
INSERT INTO tbCheck VALUES ('A3Qw5PuTSYqL7#Nu')
INSERT INTO tbCheck VALUES ('RTQw5PuTSYqL7#Nu')

 
INSERT INTO tbCheck 
SELECT CAST([name] + '1' AS NVARCHAR(128)) 
FROM sys.sql_logins

;WITH ReturnUsedLogins AS(
	SELECT 
		[name] LoginName
		, password_hash PasswordHash
		, CheckValue
		, PWDCOMPARE(CheckValue, password_hash) Compare
	FROM sys.sql_logins 
		CROSS APPLY tbCheck
)
SELECT LoginName
	, CheckValue
FROM ReturnUsedLogins
WHERE Compare = 1
 -----------------------------------------------------------------------------------------------------------------
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 -- HISTORY TABELE 
 -- TEMPORALNE TABELE
  -----------------------------------------------------------------------------------------------------------------
 alter table dbo.Person 
    add [SysStartTime] datetime2 generated always as row start hidden not null
            constraint DF_SysStart default sysutcdatetime(),
        [SysEndTime] datetime2 generated always as row end hidden not null
            constraint DF_SysEnd default convert(datetime2, '9999-12-31 23:59:59.9999999'),
        Period for system_time (SysStartTime, SysEndTime)
go
alter table dbo.Person set (system_versioning = on (history_table = dbo.Person_History))


select * from dbo.Person_History

select *
from Person
for system_time as of   '2023-09-07 11:27:48.0139021'
order by PersonId;

select *
from Person
for system_time as of   '2023-09-07 11:30:25.3731299'
order by PersonId;


create table dbo.Person
(
    [PersonId] int not null primary key clustered identity(1,1),
    [Name] nvarchar(max) not null,
    [Email] nvarchar(max) null,
    [Address] nvarchar(max) null,
    [PhoneNumber] nvarchar(max) null
)
--go
--alter table dbo.Person 
--    add [SysStartTime] datetime2 generated always as row start hidden not null
--            constraint DF_SysStart default sysutcdatetime(),
--        [SysEndTime] datetime2 generated always as row end hidden not null
--            constraint DF_SysEnd default convert(datetime2, '9999-12-31 23:59:59.9999999'),
--        Period for system_time (SysStartTime, SysEndTime)
--go
--alter table dbo.Person set (system_versioning = on (history_table = dbo.Person_History))
--go

ALTER TABLE [dbo].Person SET ( SYSTEM_VERSIONING = OFF)
GO
DROP TABLE [dbo].Person
GO
DROP TABLE [dbo].Person_History
GO

ALTER TABLE [CBS].komitent SET ( SYSTEM_VERSIONING = OFF)
GO
ALTER table CBS.komitent set (system_versioning = on (history_table = HIST.komitentH))

 -----------------------------------------------------------------------------------------------------------------
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 /*
 --BRZI CURSOR
  -----------------------------------------------------------------------------------------------------------------
 BEGIN
DECLARE @baza VARCHAR(50), @path VARCHAR(200), @print BIT, @file VARCHAR(50), @filepath VARCHAR(200)
set @print = 1

DECLARE foreach CURSOR FOR 
select DB_NAME(DATABASE_ID), name,physical_name from sys.master_files
where name like '%test%' and name like '%log%';

OPEN foreach  
FETCH NEXT FROM foreach INTO @baza, @file, @filepath
	WHILE @@FETCH_STATUS = 0  
BEGIN 
        	SET @path = 'ALTER DATABASE ['+@baza+']
                 MODIFY FILE ( NAME = ' +@file+  ',
                 FILENAME = ''N:\MSSQL\Data\' +right(@filepath,15)+ ''');
                 GO'
IF(@print = 0) EXECUTE (@path) ELSE PRINT (@path);   

FETCH NEXT FROM foreach INTO @baza, @file, @filepath
END 

CLOSE foreach  
DEALLOCATE foreach 	
END
  -----------------------------------------------------------------------------------------------------------------
  */
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  -- BRZI EXECUTE SQL
    -----------------------------------------------------------------------------------------------------------------
DECLARE @SQLSTRING nvarchar(2000)
SET @SQLSTRING = N'insert into #PRPA_acc_param select * 
from openquery(BO, ''select distinct person_code,tranz_acct,p.crd_change_date as datum_promjene_limita, 
p.crd as limit,i.created_date as datumupisa,ac_status as status_racuna,stop_date
from issuing2_0.PRPA_acc_param p 
join issuing2_0.PRPA_acc_info i on i.account_no=p.account_no
where p.crd_change_date<= to_date(''''' + @datumdo + ''''',''''yyyy-mm-dd'''') 
and ltrim(tranz_acct) like ''''07%'''' '')'
EXECUTE sp_executesql @SQLSTRING
    -----------------------------------------------------------------------------------------------------------------
  
  
  
  
  
  
  
  
  
  
  -- DISTRIBUTEDAG
     -----------------------------------------------------------------------------------------------------------------
   select ag.name, ag.is_distributed, ar.replica_server_name, ar.availability_mode_desc, ars.connected_state_desc, ars.role_desc, 
 ars.operational_state_desc, ars.synchronization_health_desc from sys.availability_groups ag  
 join sys.availability_replicas ar on ag.group_id=ar.group_id
 left join sys.dm_hadr_availability_replica_states ars
 on ars.replica_id=ar.replica_id
 where ag.is_distributed=1
 GO


-- Na source
CREATE AVAILABILITY GROUP [distributedag]  
   WITH (DISTRIBUTED)   
   AVAILABILITY GROUP ON  
      'T24DBAG' WITH    
      (   
         LISTENER_URL = 'tcp://T24DB.TODO.rbbh:5022',    
         AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,   
         FAILOVER_MODE = MANUAL,   
         SEEDING_MODE = AUTOMATIC   
      ),   
      'T24DBPPAG' WITH    
      (   
         LISTENER_URL = 'tcp://T24DBPP.TODO.rbbh:5022',   
         AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,   
         FAILOVER_MODE = MANUAL,   
         SEEDING_MODE = AUTOMATIC   
      );    
GO   
-- Na destinaciji
alter AVAILABILITY GROUP [distributedag]  
   join   
   AVAILABILITY GROUP ON  
      'T24DBAG' WITH    
      (   
         LISTENER_URL = 'tcp://T24DB.TODO.rbbh:5022',    
         AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,   
         FAILOVER_MODE = MANUAL,   
         SEEDING_MODE = AUTOMATIC   
      ),   
      'T24DBPPAG' WITH    
      (   
         LISTENER_URL = 'tcp://T24DBPP.TODO.rbbh:5022',   
         AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,   
         FAILOVER_MODE = MANUAL,   
         SEEDING_MODE = AUTOMATIC   
      );    
GO  

--DROP AVAILABILITY GROUP DISTRIBUTEDAG
ALTER AVAILABILITY GROUP T24DBPP GRANT CREATE ANY DATABASE


ALTER AVAILABILITY GROUP [DATAPOOLAG_SEC] 
MODIFY REPLICA ON 'DATAB' WITH (ENDPOINT_URL = 'TCP://DATAb.TODO.rbbh:5019');
   -----------------------------------------------------------------------------------------------------------------
   
   
   
   
   
   
   
   
   -- SCRUMBLE TABELA
      -----------------------------------------------------------------------------------------------------------------
   -	Scramble imena i prezimena:
--Tabela za target
select * from [SEB].[C_USER_2]

Declare @id int

DECLARE CURS CURSOR FOR 
select C_USERID from [SEB].[C_USER_2]
OPEN CURS
FETCH NEXT FROM CURS INTO @id;

WHILE (@@FETCH_STATUS = 0) 
BEGIN
    UPDATE [SEB].[C_USER_2]
        SET OFX_FIRSTNAME = (SELECT TOP 1 OFX_FIRSTNAME FROM [C_USER_2] ORDER BY NEWID())
    WHERE C_USERID = @id
    UPDATE [SEB].[C_USER_2]
        SET OFX_LASTNAME = (SELECT TOP 1 OFX_LASTNAME FROM [C_USER_2] ORDER BY NEWID())
    WHERE C_USERID = @id
    FETCH NEXT FROM CURS INTO @id;
END

CLOSE CURS;
DEALLOCATE CURS;

-	Sve email adrese da budu spoj ime + prezime + gmail.com:
select
RTRIM(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(lower(COALESCE(concat(OFX_FIRSTNAME, '.', OFX_LASTNAME, '@gmail.com'),'')), 
   N'Æ','c'), N'č','c'), N'ć','c'),N'Č','c'), N'Ć','c'), N'š','s'), N'Š','s'), N'Đ','dj'),N'Ð','dj'), 'Ž','z'), 'ž','z')) AS OFX_EMAIL
from [priprema_shuffle]

      -----------------------------------------------------------------------------------------------------------------
   
   
   
   
   
   
   
   
   
   
   
   
   
   
   
   
   
   
   -- PROVJERA VREMENA JOBOVA I 24 KOJI TRAJU stepova
   -----------------------------------------------------------------------------------------------------------------   
   SELECT j.Name, jh.Step_name,

CONVERT(DATETIME, RTRIM(jh.run_date)) 
+ ((jh.run_time/10000 * 3600) 
+ ((jh.run_time%10000)/100*60) +
(jh.run_time%10000)%100 ) / (23.999999*3600 )
AS Start_DateTime
,STUFF(STUFF(STUFF(RIGHT(REPLICATE('0', 8) + CAST(jh.run_duration as varchar(8)), 8), 3, 0, ':'), 6, 0, ':'), 9, 0, ':') AS RunDuration,

CONVERT(DATETIME, 
RTRIM(jh.run_date)) + ((jh.run_time/10000 * 3600) + ((jh.run_time%10000)/100*60) +
(jh.run_time%10000)%100) / (86399.9964)
+ ((jh.run_duration/10000 * 3600) + ((jh.run_duration%10000)/100*60) + (jh.run_duration%10000)%100
) / (86399.9964) AS End_DateTime

from msdb..sysjobhistory jh, msdb..sysjobs j
where jh.job_id=j.job_id
and j.name = 'LCR'
ORDER BY run_date desc, run_time desc



SELECT sj.name
,  sja.start_execution_date,
DATEDIFF(minute,sja.start_execution_date,getdate())  as Runtime
FROM msdb.dbo.sysjobactivity AS sja
INNER JOIN msdb.dbo.sysjobs AS sj ON sja.job_id = sj.job_id
WHERE sja.start_execution_date IS NOT NULL
AND sja.stop_execution_date IS NULL
and DATEDIFF(minute,sja.start_execution_date,getdate())>=24
   -----------------------------------------------------------------------------------------------------------------
   
   
   
   
   
   
   --EVENT SESSION
CREATE EVENT SESSION [Capture_details_amar] ON SERVER 
ADD EVENT sqlserver.error_reported(
    ACTION(package0.event_sequence,package0.last_error,sqlos.worker_address,sqlserver.client_app_name,sqlserver.client_connection_id,sqlserver.client_hostname,sqlserver.client_pid,sqlserver.compile_plan_guid,sqlserver.context_info,sqlserver.database_name,sqlserver.distributed_query_id,sqlserver.distributed_request_id,sqlserver.execution_plan_guid,sqlserver.nt_username,sqlserver.plan_handle,sqlserver.query_hash,sqlserver.query_plan_hash,sqlserver.server_instance_name,sqlserver.server_principal_name,sqlserver.session_nt_username,sqlserver.session_server_principal_name,sqlserver.sql_text,sqlserver.transaction_sequence,sqlserver.tsql_frame,sqlserver.tsql_stack,sqlserver.username)
    WHERE ([sqlserver].[like_i_sql_unicode_string]([message],N'%gradjani%'))),
ADD EVENT sqlserver.sql_batch_completed(
    ACTION (package0.event_sequence,package0.last_error,sqlos.worker_address,sqlserver.client_app_name,sqlserver.client_connection_id,sqlserver.client_hostname,sqlserver.client_pid,sqlserver.compile_plan_guid,sqlserver.context_info,sqlserver.database_name,sqlserver.distributed_query_id,sqlserver.distributed_request_id,sqlserver.execution_plan_guid,sqlserver.nt_username,sqlserver.plan_handle,sqlserver.query_hash,sqlserver.query_plan_hash,sqlserver.server_instance_name,sqlserver.server_principal_name,sqlserver.session_nt_username,sqlserver.session_server_principal_name,sqlserver.sql_text,sqlserver.transaction_sequence,sqlserver.tsql_frame,sqlserver.tsql_stack,sqlserver.username)
    WHERE ([sqlserver].[like_i_sql_unicode_string]([sqlserver].[sql_text],N'%gradjani%'))),
ADD EVENT sqlserver.sql_statement_completed(
    ACTION (package0.event_sequence,package0.last_error,sqlos.worker_address,sqlserver.client_app_name,sqlserver.client_connection_id,sqlserver.client_hostname,sqlserver.client_pid,sqlserver.compile_plan_guid,sqlserver.context_info,sqlserver.database_name,sqlserver.distributed_query_id,sqlserver.distributed_request_id,sqlserver.execution_plan_guid,sqlserver.nt_username,sqlserver.plan_handle,sqlserver.query_hash,sqlserver.query_plan_hash,sqlserver.server_instance_name,sqlserver.server_principal_name,sqlserver.session_nt_username,sqlserver.session_server_principal_name,sqlserver.sql_text,sqlserver.transaction_sequence,sqlserver.tsql_frame,sqlserver.tsql_stack,sqlserver.username)
	WHERE ([sqlserver].[like_i_sql_unicode_string]([sqlserver].[sql_text],N'%gradjani%')))
WITH (MAX_MEMORY=4096 KB,EVENT_RETENTION_MODE=ALLOW_SINGLE_EVENT_LOSS,MAX_DISPATCH_LATENCY=30 SECONDS,MAX_EVENT_SIZE=0 KB,MEMORY_PARTITION_MODE=NONE,TRACK_CAUSALITY=OFF,STARTUP_STATE=OFF)
GO
   --     WHERE ([package0].[greater_than_int64]([severity],(16))))
   
   
   CREATE EVENT SESSION [Capture_amar] ON SERVER 
ADD EVENT sqlserver.sql_batch_completed(
    ACTION(package0.event_sequence,package0.last_error,sqlos.worker_address,sqlserver.client_app_name,sqlserver.client_connection_id,sqlserver.client_hostname,sqlserver.client_pid,sqlserver.compile_plan_guid,sqlserver.context_info,sqlserver.database_name,sqlserver.distributed_query_id,sqlserver.distributed_request_id,sqlserver.execution_plan_guid,sqlserver.nt_username,sqlserver.plan_handle,sqlserver.query_hash,sqlserver.query_plan_hash,sqlserver.server_instance_name,sqlserver.server_principal_name,sqlserver.session_nt_username,sqlserver.session_server_principal_name,sqlserver.sql_text,sqlserver.transaction_sequence,sqlserver.tsql_frame,sqlserver.tsql_stack,sqlserver.username)
    WHERE ([sqlserver].[sql_text] like N'INSERT%' OR [sqlserver].[sql_text] like N'UPDATE%' AND [sqlserver].[server_instance_name]=N'pp-T24sarajdb'))
WITH (MAX_MEMORY=4096 KB,EVENT_RETENTION_MODE=ALLOW_SINGLE_EVENT_LOSS,MAX_DISPATCH_LATENCY=30 SECONDS,MAX_EVENT_SIZE=0 KB,MEMORY_PARTITION_MODE=NONE,TRACK_CAUSALITY=OFF,STARTUP_STATE=ON)
GO



   --EVENT SESSION blocking
CREATE EVENT SESSION [Capture_details_amar] 
ON SERVER
ADD EVENT sqlserver.blocked_process_report(
    ACTION(sqlserver.sql_text, sqlserver.tsql_stack, sqlserver.session_id))
ADD TARGET package0.event_file(SET filename=N'C:\UPDATE\BlockingSessions.xel', max_file_size=(10), max_rollover_files=(5))
WITH (MAX_MEMORY=4096 KB, EVENT_RETENTION_MODE=ALLOW_SINGLE_EVENT_LOSS, MAX_DISPATCH_LATENCY=30 SECONDS, MAX_EVENT_SIZE=0 KB, MEMORY_PARTITION_MODE=NONE, TRACK_CAUSALITY=OFF, STARTUP_STATE=ON);
GO

ALTER EVENT SESSION [Capture_details_amar] ON SERVER STATE = START;
GO



-- Create the Extended Event session
CREATE EVENT SESSION BlockingSessions
ON SERVER
ADD EVENT sqlserver.blocked_process_report(
    ACTION(sqlserver.sql_text, sqlserver.tsql_stack, sqlserver.session_id, sqlserver.client_app_name, sqlserver.database_id, sqlserver.client_hostname)
    WHERE ([sqlserver].[database_id] > 4)) -- Exclude system databases
ADD TARGET package0.event_file(SET filename=N'C:\UPDATE\BlockingSessions2.xel', max_file_size=(50), max_rollover_files=(10))
WITH (
    MAX_MEMORY=4096 KB, 
    EVENT_RETENTION_MODE=ALLOW_SINGLE_EVENT_LOSS, 
    MAX_DISPATCH_LATENCY=5 SECONDS, 
    MAX_EVENT_SIZE=0 KB, 
    MEMORY_PARTITION_MODE=NONE, 
    TRACK_CAUSALITY=ON, 
    STARTUP_STATE=ON
);
GO

-- Start the session
ALTER EVENT SESSION BlockingSessions ON SERVER STATE = START;
GO




create procedure [dbo].[readExEventFile]
(
	 @filePath varchar(1024)
	,@eventName varchar(128)
)
as
begin

	/*
		exec [dbo].[readExEventFile] @filePath = 'i:\temp\*.xel', @eventName = 'error_reported'
	*/

	if @eventName = 'error_reported'
	select
		n.value('(@name)[1]', 'varchar(50)') as [event_name],
		n.value('(@package)[1]', 'varchar(50)') AS [package_name],
		n.value('(@timestamp)[1]', 'datetime2') AS [utc_timestamp],
		n.value('(data[@name="error_number"]/value)[1]', 'int') as [errorNo],
		n.value('(data[@name="severity"]/value)[1]', 'int') as [severity],
		n.value('(data[@name="state"]/value)[1]', 'int') as [state],
		n.value('(data[@name="user_defined"]/value)[1]', 'varchar(128)') as [userDefined],
		n.value('(data[@name="category"]/text)[1]', 'varchar(128)') as [category],
		n.value('(data[@name="destination"]/text)[1]', 'varchar(128)') as [destination],
		n.value('(data[@name="message"]/value)[1]', 'varchar(128)') as [message]
	from (select cast(event_data as XML) as event_data
	from sys.fn_xe_file_target_read_file(@filePath, null, null, null)
			where object_name = 'error_reported') ed
	cross apply ed.event_data.nodes('event') as q(n)

end
GO


   
   
   
   
   
   --CONVERT TIME 
   -----------------------------------------------------------------------------------------------------------------
DECLARE @DEF NVARCHAR(30), @PRC NVARCHAR(30)
SELECT @DEF = convert(varchar,Default_vrijeme, 120) FROM #vrijeme_prepisa

print @DEF + '_sa_vremena_prepisa'


IF (SELECT DATEPART(HOUR, GETDATE())) <= '10'
   SET @PRC = convert(varchar(10), getdate(),126) + ' 4:15:00'
ELSE 
   SET @PRC = convert(varchar(10), getdate(),126) + ' 23:59:00'
IF CONVERT(varchar(20),CONVERT(datetime, @DEF), 114) > CONVERT(varchar(20),CONVERT(datetime, @PRC), 114)
   SET @PRC = @DEF
PRINT @PRC
   -----------------------------------------------------------------------------------------------------------------
   --Nova logika datasource
SELECT TOP 1
CONVERT(DATETIME, 
RTRIM(jh.run_date)) + ((jh.run_time/10000 * 3600) + ((jh.run_time%10000)/100*60) +
(jh.run_time%10000)%100) / (86399.9964)
+ ((jh.run_duration/10000 * 3600) + ((jh.run_duration%10000)/100*60) + (jh.run_duration%10000)%100
) / (86399.9964) AS [Default_vrijeme]
INTO #vrijeme_default_joba
FROM
    OPENROWSET('SQLNCLI', 'Server=bosnadbprim\head;Trusted_Connection=yes;',
    [msdb].[dbo].[sysjobhistory]) AS  jh
INNER JOIN 
    OPENROWSET('SQLNCLI', 'Server=bosnadbprim\head;Trusted_Connection=yes;',
    [msdb].[dbo].[sysjobs]) AS  j
       ON (jh.[job_id] = j.[job_id])
where jh.job_id=j.job_id
and j.name = 'BIH.C.2.1. Defaults' and jh.step_name = '(Job outcome)'
ORDER BY run_date desc, run_time desc

DECLARE @DEF NVARCHAR(30), @PRC NVARCHAR(30)
SELECT @DEF = convert(varchar,Default_vrijeme, 120) FROM #vrijeme_default_joba

IF (SELECT DATEPART(HOUR, GETDATE())) <= '10'
   SET @PRC = convert(varchar(10), getdate(),126) + ' 4:15:00'
ELSE 
   SET @PRC = convert(varchar(10), getdate(),126) + ' 23:59:00'
IF CONVERT(varchar(20),CONVERT(datetime, @DEF), 114) > CONVERT(varchar(20),CONVERT(datetime, @PRC), 114)
   SET @PRC = convert(varchar,DATEADD(minute,10,@DEF), 120)
      -----------------------------------------------------------------------------------------------------------------
	  
	  
	  
	  
	  
	  
	  
	  
	  
      -----------------------------------------------------------------------------------------------------------------	  
	  --PROVJERA PARTICIJA
	    SELECT 
    partition_number,
    row_count
FROM sys.dm_db_partition_stats
WHERE object_id = OBJECT_ID('dbo.F_DATA_EVENTS');


SELECT 
	PF.name AS PartitionFunction,
	ds.name AS PartitionScheme,
    OBJECT_SCHEMA_NAME(si.object_id) as SchemaName,
	OBJECT_NAME(si.object_id) AS PartitionedTable, 
	si.name as IndexName
FROM sys.indexes AS si
JOIN sys.data_spaces AS ds
	ON ds.data_space_id = si.data_space_id
JOIN sys.partition_schemes AS PS
	ON PS.data_space_id = si.data_space_id
JOIN sys.partition_functions AS PF
	ON PF.function_id = PS.function_id
WHERE ds.type = 'PS'
AND OBJECTPROPERTYEX(si.object_id, 'BaseType') = 'U'
ORDER BY PartitionFunction, PartitionScheme, SchemaName, PartitionedTable;
      -----------------------------------------------------------------------------------------------------------------	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  --insert iz procedure
	  SELECT * INTO #dbrepl_monitor1 FROM OPENROWSET('SQLNCLI', 'Server=sarajdbprim;Trusted_Connection=yes;',
     'set fmtonly off; EXEC [distribution].dbo.sp_replmonitorhelppublication with result sets 
	 (
	 (
		publisher_db sysname 
		,publication sysname 
		,publication_id int 
		,publication_type int
		,status int
		,warning int
		,worst_latency int
		,best_latency int
		,avg_latency int
		,last_distsync datetime	
		,retention int
        ,latencythreshold int
        ,expirationthreshold int
		,agentnotrunningthreshold int
        ,subscriptioncount int
        ,runningdistagentcount int
		,snapshot_agentname sysname null
		,logreader_agentname sysname null
		,qreader_agentname sysname null
		,worst_runspeedPerf int
		,best_runspeedPerf int
		,average_runspeedPerf int 
		,retention_period_unit tinyint
        ,publisher sysname null
	 )
	 )')
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
---------------------------------- VRACANJE PERMISIJA USERS ROLE
	 DECLARE 
    @sql VARCHAR(2048)
    ,@sort INT
 
DECLARE tmp CURSOR FOR
 
 
/*********************************************/
/*********   DB CONTEXT STATEMENT    *********/
/*********************************************/
SELECT '-- [-- DB CONTEXT --] --' AS [-- SQL STATEMENTS --],
        1 AS [-- RESULT ORDER HOLDER --]
UNION
SELECT  'USE' + SPACE(1) + QUOTENAME(DB_NAME()) AS [-- SQL STATEMENTS --],
        1 AS [-- RESULT ORDER HOLDER --]
 
UNION
 
SELECT '' AS [-- SQL STATEMENTS --],
        2 AS [-- RESULT ORDER HOLDER --]
 
UNION
 
/*********************************************/
/*********     DB USER CREATION      *********/
/*********************************************/
 
SELECT '-- [-- DB USERS --] --' AS [-- SQL STATEMENTS --],
        3 AS [-- RESULT ORDER HOLDER --]
UNION
SELECT  'IF NOT EXISTS (SELECT [name] FROM sys.database_principals WHERE [name] = ' + SPACE(1) + '''' + [name] + '''' + ') BEGIN CREATE USER ' + SPACE(1) + QUOTENAME([name]) + ' FOR LOGIN ' + QUOTENAME([name]) + ' WITH DEFAULT_SCHEMA = ' + QUOTENAME([default_schema_name]) + SPACE(1) + 'END; ' AS [-- SQL STATEMENTS --],
        4 AS [-- RESULT ORDER HOLDER --]
FROM    sys.database_principals AS rm
WHERE [type] IN ('U', 'S', 'G') -- windows users, sql users, windows groups
 
UNION
 
/*********************************************/
/*********    DB ROLE PERMISSIONS    *********/
/*********************************************/
SELECT '-- [-- DB ROLES --] --' AS [-- SQL STATEMENTS --],
        5 AS [-- RESULT ORDER HOLDER --]
UNION
SELECT  'EXEC sp_addrolemember @rolename ='
    + SPACE(1) + QUOTENAME(USER_NAME(rm.role_principal_id), '''') + ', @membername =' + SPACE(1) + QUOTENAME(USER_NAME(rm.member_principal_id), '''') AS [-- SQL STATEMENTS --],
        6 AS [-- RESULT ORDER HOLDER --]
FROM    sys.database_role_members AS rm
WHERE   USER_NAME(rm.member_principal_id) IN (  
                                                --get user names on the database
                                                SELECT [name]
                                                FROM sys.database_principals
                                                WHERE [principal_id] > 4 -- 0 to 4 are system users/schemas
                                                and [type] IN ('G', 'S', 'U') -- S = SQL user, U = Windows user, G = Windows group
                                              )
--ORDER BY rm.role_principal_id ASC
 
 
UNION
 
SELECT '' AS [-- SQL STATEMENTS --],
        7 AS [-- RESULT ORDER HOLDER --]
 
UNION
 
/*********************************************/
/*********  OBJECT LEVEL PERMISSIONS *********/
/*********************************************/
SELECT '-- [-- OBJECT LEVEL PERMISSIONS --] --' AS [-- SQL STATEMENTS --],
        8 AS [-- RESULT ORDER HOLDER --]
UNION
SELECT  CASE 
            WHEN perm.state <> 'W' THEN perm.state_desc 
            ELSE 'GRANT'
        END
        + SPACE(1) + perm.permission_name + SPACE(1) + 'ON ' + QUOTENAME(SCHEMA_NAME(obj.schema_id)) + '.' + QUOTENAME(obj.name) --select, execute, etc on specific objects
        + CASE
                WHEN cl.column_id IS NULL THEN SPACE(0)
                ELSE '(' + QUOTENAME(cl.name) + ')'
          END
        + SPACE(1) + 'TO' + SPACE(1) + QUOTENAME(USER_NAME(usr.principal_id)) COLLATE database_default
        + CASE 
                WHEN perm.state <> 'W' THEN SPACE(0)
                ELSE SPACE(1) + 'WITH GRANT OPTION'
          END
            AS [-- SQL STATEMENTS --],
        9 AS [-- RESULT ORDER HOLDER --]
FROM    
    sys.database_permissions AS perm
        INNER JOIN
    sys.objects AS obj
            ON perm.major_id = obj.[object_id]
        INNER JOIN
    sys.database_principals AS usr
            ON perm.grantee_principal_id = usr.principal_id
        LEFT JOIN
    sys.columns AS cl
            ON cl.column_id = perm.minor_id AND cl.[object_id] = perm.major_id
--WHERE usr.name = @OldUser
--ORDER BY perm.permission_name ASC, perm.state_desc ASC
 
 
UNION
 
SELECT '' AS [-- SQL STATEMENTS --],
    10 AS [-- RESULT ORDER HOLDER --]
 
UNION
 
/*********************************************/
/*********    DB LEVEL PERMISSIONS   *********/
/*********************************************/
SELECT '-- [--DB LEVEL PERMISSIONS --] --' AS [-- SQL STATEMENTS --],
        11 AS [-- RESULT ORDER HOLDER --]
UNION
SELECT  CASE 
            WHEN perm.state <> 'W' THEN perm.state_desc --W=Grant With Grant Option
            ELSE 'GRANT'
        END
    + SPACE(1) + perm.permission_name --CONNECT, etc
    + SPACE(1) + 'TO' + SPACE(1) + '[' + USER_NAME(usr.principal_id) + ']' COLLATE database_default --TO <user name>
    + CASE 
            WHEN perm.state <> 'W' THEN SPACE(0) 
            ELSE SPACE(1) + 'WITH GRANT OPTION' 
      END
        AS [-- SQL STATEMENTS --],
        12 AS [-- RESULT ORDER HOLDER --]
FROM    sys.database_permissions AS perm
    INNER JOIN
    sys.database_principals AS usr
    ON perm.grantee_principal_id = usr.principal_id
--WHERE usr.name = @OldUser
 
WHERE   [perm].[major_id] = 0
    AND [usr].[principal_id] > 4 -- 0 to 4 are system users/schemas
    AND [usr].[type] IN ('G', 'S', 'U') -- S = SQL user, U = Windows user, G = Windows group
 
UNION
 
SELECT '' AS [-- SQL STATEMENTS --],
        13 AS [-- RESULT ORDER HOLDER --]
 
UNION
 
SELECT '-- [--DB LEVEL SCHEMA PERMISSIONS --] --' AS [-- SQL STATEMENTS --],
        14 AS [-- RESULT ORDER HOLDER --]
UNION
SELECT  CASE
            WHEN perm.state <> 'W' THEN perm.state_desc --W=Grant With Grant Option
            ELSE 'GRANT'
            END
                + SPACE(1) + perm.permission_name --CONNECT, etc
                + SPACE(1) + 'ON' + SPACE(1) + class_desc + '::' COLLATE database_default --TO <user name>
                + QUOTENAME(SCHEMA_NAME(major_id))
                + SPACE(1) + 'TO' + SPACE(1) + QUOTENAME(USER_NAME(grantee_principal_id)) COLLATE database_default
                + CASE
                    WHEN perm.state <> 'W' THEN SPACE(0)
                    ELSE SPACE(1) + 'WITH GRANT OPTION'
                    END
            AS [-- SQL STATEMENTS --],
        15 AS [-- RESULT ORDER HOLDER --]
from sys.database_permissions AS perm
    inner join sys.schemas s
        on perm.major_id = s.schema_id
    inner join sys.database_principals dbprin
        on perm.grantee_principal_id = dbprin.principal_id
WHERE class = 3 --class 3 = schema
 
 
ORDER BY [-- RESULT ORDER HOLDER --]
 
 
OPEN tmp
FETCH NEXT FROM tmp INTO @sql, @sort
WHILE @@FETCH_STATUS = 0
BEGIN
        PRINT @sql
        FETCH NEXT FROM tmp INTO @sql, @sort    
END
 
CLOSE tmp
DEALLOCATE tmp
---------------------------------------------------------------------------
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	 
	--provjera connection
SELECT        spm.class_desc, spm.permission_name, spm.state_desc, spr.name, spr.type_desc, spr.is_disabled, ep.name AS EndpointName, ep.protocol_desc AS EndPointDescription, ep.state_desc AS EndPointState, ep.type_desc AS EndpointType
FROM            sys.server_permissions AS spm INNER JOIN
                         sys.server_principals AS spr ON spm.grantee_principal_id = spr.principal_id LEFT OUTER JOIN
                         sys.endpoints AS ep ON spm.major_id = ep.endpoint_id
	
	 
	 
	 
	 -- PROVJERA ZADNJE AKTIVNOSTI NA TABELI
	 SELECT t.object_id [ObjectId],
       s.name [Schema],
       t.name [TableName],
       MAX(us.last_user_update) [LastUpdate]
FROM sys.dm_db_index_usage_stats us
    JOIN sys.tables t
        ON t.object_id = us.object_id
    JOIN sys.schemas s
        ON s.schema_id = t.schema_id
WHERE us.database_id = DB_ID()
--AND t.object_id = OBJECT_ID('YourSchemaName.TableName') --Filter By Table
GROUP BY t.object_id,
         s.name,
         t.name
ORDER BY MAX(us.last_user_update) DESC;








Declare @AMAR Table 
   (SPID INT, Status VARCHAR(255),
     Login  VARCHAR(255), HostName  VARCHAR(255),
     BlkBy  VARCHAR(255), DBName  VARCHAR(255),
     Command VARCHAR(255), CPUTime INT,
     DiskIO INT, LastBatch VARCHAR(255),
     ProgramName VARCHAR(255), SPID1 INT,
     REQUESTID INT);
INSERT INTO @AMAR
EXEC sp_who2
SELECT      *
FROM       @AMAR
WHERE DBName = 'iBankingCorporate'







ALTER PROCEDURE [dbo].[lock]
@spid1 int = NULL,		/* server process id to check for locks */
@spid2 int = NULL		/* other process id to check for locks */
WITH EXECUTE AS OWNER, ENCRYPTION
AS

EXECUTE [sys].[sp_lock] @spid1, @spid2

GO








-- UZIMATI SELECT ;
declare @message NVARCHAR(MAX)
set @message = '38762521841;38762434205;edin'

BEGIN
    DECLARE @position INT,
            @length INT,
            @substring NVARCHAR(MAX),
            @delimiter CHAR(1) = ';',
            @nextDelimiterPosition INT,
			@substring2 NVARCHAR(MAX)

    -- Initialize the cursor
    DECLARE string_cursor CURSOR FOR
    SELECT value
    FROM STRING_SPLIT(@message, @delimiter)

    -- Open the cursor
    OPEN string_cursor

    -- Fetch the first value
    FETCH NEXT FROM string_cursor INTO @substring

    -- Loop through the cursor
    WHILE @@FETCH_STATUS = 0




    BEGIN
        -- Process each parsed value
        PRINT @substring  -- Replace this line with your desired logic
		set @substring2 = @substring + 'test'+'test'
		 PRINT @substring2
        -- Fetch the next value
        FETCH NEXT FROM string_cursor INTO @substring
    END



    -- Close and deallocate the cursor
    CLOSE string_cursor
    DEALLOCATE string_cursor
END
--------------------------------








-- Kreiranje indexa na View
----------------------------------------------------------------
create or ALTER VIEW dbo.TestView
    WITH SCHEMABINDING
AS

SELECT PERSONID, [NAME], VRIJEME, HashValue
  FROM [dbo].[Person_6]
  where [address] in ('zagreb', 'blagaj')

GO

--Create an index on the view.
CREATE UNIQUE CLUSTERED INDEX IDX_V1 ON dbo.TestView (
    Vrijeme,
	PersonId
);
GO
----------------------------------------------------------------



--Pregled joba start
----------------------------------------------------------------
select 
    jobs.name
    ,jobs.description
    ,steps.step_id
    ,steps.step_name
    ,steps.last_run_outcome
    ,last_run_time = stuff(stuff(right('00000' + cast(steps.last_run_time as varchar),6),3,0,':'),6,0,':')
    ,last_run_duration = stuff(stuff(right('00000' + cast(steps.last_run_duration as varchar),6),3,0,':'),6,0,':')
from [msdb].[dbo].[sysjobs] jobs
inner join [msdb].[dbo].[sysjobsteps] steps on
steps.job_id = jobs.job_id
order by jobs.job_id, steps.step_id
----------------------------------------------------------------




--PROVJERA COLLATION:
SELECT SERVERPROPERTY('Collation') AS ServerCollation;
SELECT COLLATIONPROPERTY('Latin1_General_BIN2', 'CodePage') AS CodePage;
SELECT 
    name AS DatabaseName,
    state_desc AS State,
    recovery_model_desc AS RecoveryModel,
    compatibility_level AS CompatibilityLevel,
    collation_name AS CollationName,
    is_auto_close_on AS IsAutoCloseOn,
    is_auto_shrink_on AS IsAutoShrinkOn
FROM 
    sys.databases;

	SELECT 
    SERVERPROPERTY('IsClustered') AS IsClustered,
    CASE SERVERPROPERTY('IsClustered')
        WHEN 1 THEN 'Clustered'
        WHEN 0 THEN 'Standalone'
        ELSE 'Unknown'
    END AS ClusterStatus;
 SELECT name, collation_name FROM sys.databases;

-- Basic Server Properties
SELECT 
    SERVERPROPERTY('ProductVersion') AS ProductVersion,
    SERVERPROPERTY('ProductLevel') AS ProductLevel,
    SERVERPROPERTY('Edition') AS Edition,
    SERVERPROPERTY('EngineEdition') AS EngineEdition,
    SERVERPROPERTY('MachineName') AS MachineName,
    SERVERPROPERTY('IsClustered') AS IsClustered,
    SERVERPROPERTY('Collation') AS Collation;
	
	
	
	
	
	--restart endopint
	use master
GO
alter endpoint endpoint_name state = stopped;
GO
alter endpoint endpoint_name state = started;
GO





-------------------------memory provjera

select sum(pages_kb) from sys.dm_os_memory_clerks


select * from sys.dm_os_memory_clerks
where type = 'MEMORYCLERK_SQLBUFFERPOOL'


SELECT total_physical_memory_kb / 1024 AS MemoryMb 
FROM sys.dm_os_sys_memory

SELECT name, value_in_use FROM sys.configurations 
WHERE name LIKE 'max server memory%'

SELECT TOP(5) [type] AS [ClerkType],
SUM(pages_kb) / 1024 AS [SizeMb]
FROM sys.dm_os_memory_clerks WITH (NOLOCK)
GROUP BY [type]
ORDER BY SUM(pages_kb) DESC

SELECT session_id, requested_memory_kb / 1024 as RequestedMemMb, 
granted_memory_kb / 1024 as GrantedMemMb, text
FROM sys.dm_exec_query_memory_grants qmg
CROSS APPLY sys.dm_exec_sql_text(sql_handle)


SELECT TOP 5 DB_NAME(database_id) AS [Database Name],
COUNT(*) * 8/1024.0 AS [Cached Size (MB)]
FROM sys.dm_os_buffer_descriptors WITH (NOLOCK)
GROUP BY DB_NAME(database_id)
ORDER BY [Cached Size (MB)] DESC OPTION (RECOMPILE);















------brzi restore po custom linku---------------------------------------------------------------------------

execute msdb.dbo.usp_KillUsers 'KremaTransactMig3'

declare @db VARCHAR(20), @folder VARCHAR(150), @print BIT, @path VARCHAR(max), @fullpath VARCHAR(200)
set @folder = '\\172.21.199.126\SQLTEST\T24DB1\KremaTransact\FULL';
set @print = 0;

    CREATE TABLE #log_restore (
     Id int identity(1,1),
     SubDirectory nvarchar(255),
     Depth smallint,
     [file] smallint
    )
    INSERT INTO #log_restore (SubDirectory, Depth, [file])
    EXEC xp_dirtree @folder, 0, 1;


    set @path = (SELECT top 1 subdirectory from #log_restore order by SubDirectory desc);
    set @fullpath = @folder + '\' + @path
	--print @fullpath
	set @db = 'KremaTransactMig3'

        	SET @path = 'RESTORE database ['+@db+'] FROM  DISK = N'''+@fullpath+'''  WITH  FILE = 1,  
						 MOVE N''RBBDB'' TO N''D:\MSSQL\Data\KremaTransactMig3.mdf'',  
						 MOVE N''RBBDB_log'' TO N''E:\MSSQL\Data\KremaTransactMig3_log.ldf'', 
			             NOUNLOAD, REPLACE, STATS = 5;'
        	IF(@print = 0) EXECUTE (@path) ELSE PRINT (@path);       

DROP TABLE #log_restore
GO


begin
USE [KremaTransactMig3]
 IF NOT EXISTS (SELECT [name] FROM sys.database_principals WHERE [name] =  'Krema_mig3user') BEGIN CREATE USER  [Krema_mig3user] FOR LOGIN [Krema_mig3user] WITH DEFAULT_SCHEMA = [dbo] END; 
EXEC sp_addrolemember @rolename = 'db_owner', @membername = 'Krema_mig3user'
GRANT CONNECT TO [Krema_mig3user]
end
'
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------






replace([OPIS], char(0),'') opis_sredjen,










--RESOURCE GENERATOR
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
SELECT 
    s.session_id,
    s.login_name,
    rg.name AS resource_pool_name
FROM 
    sys.dm_exec_sessions AS s
JOIN 
    sys.dm_resource_governor_workload_groups AS wg
    ON s.group_id = wg.group_id
JOIN 
    sys.dm_resource_governor_resource_pools AS rg
    ON wg.pool_id = rg.pool_id
WHERE 
    s.is_user_process = 1;
	
	
	SELECT  OBJECT_NAME(classifier_function_id) AS classifier,
        is_enabled
FROM    sys.resource_governor_configuration;

-- View our pools
SELECT  *
FROM    sys.dm_resource_governor_resource_pools;

-- View our workload groups
SELECT  *
FROM    sys.dm_resource_governor_workload_groups;




CREATE RESOURCE POOL [LimitedResourcePool] 
        WITH
		(min_cpu_percent=30, 
		max_cpu_percent=50, 
		min_memory_percent=30, 
		max_memory_percent=50
		)
 
CREATE WORKLOAD GROUP LimitedResourceGroup
USING LimitedResourcePool;
GO
 
CREATE FUNCTION dbo.[UDFC_ResourceGroup_Users]()
RETURNS SYSNAME
WITH SCHEMABINDING
AS
BEGIN
DECLARE @WorkloadGroup AS SYSNAME
IF(SUSER_NAME() = 'LimitedUser') 
SET @WorkloadGroup = 'LimitedResourceGroup'
ELSE
SET @WorkloadGroup = 'default'
RETURN @WorkloadGroup
END
GO
 
ALTER RESOURCE GOVERNOR
WITH (CLASSIFIER_FUNCTION=dbo.[UDFC_ResourceGroup_Users]);
GO
ALTER RESOURCE GOVERNOR RECONFIGURE
GO
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
CREATE RESOURCE POOL [LimitedResourcePool] 
        WITH
		(min_cpu_percent=0, 
		max_cpu_percent=30, 
		min_memory_percent=0, 
		max_memory_percent=30
		)
CREATE WORKLOAD GROUP LimitedResourceGroup
USING LimitedResourcePool;
GO
 

CREATE FUNCTION [dbo].[ResourceGroup_Users]()
RETURNS SYSNAME
WITH SCHEMABINDING
AS
BEGIN
    DECLARE @WorkloadGroup SYSNAME;
    DECLARE @ProgramName SYSNAME;

    SET @ProgramName = CONVERT(SYSNAME, PROGRAM_NAME());
    IF @ProgramName = 'Microsoft SQL Server Management Studio - Query'
    BEGIN
        SET @WorkloadGroup = 'LimitedResourceGroup';
    END
    ELSE
    BEGIN
        SET @WorkloadGroup = 'default';
    END

    RETURN @WorkloadGroup;
END
GO


ALTER RESOURCE GOVERNOR
WITH (CLASSIFIER_FUNCTION=dbo.[ResourceGroup_Users]);
GO
ALTER RESOURCE GOVERNOR RECONFIGURE
GO












--BACTCH procedure
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
CREATE PROCEDURE [dbo].[PROC_ARHIVE_TMN_DES_EVENT] (@ArhiveBatch BIGINT,@OlderThanInHours BIGINT)
AS

DECLARE @Count BIGINT;

SELECT @Count = 
COUNT(STAGINGTIME)
FROM dbo.TMN_DES_EVENT
WHERE STAGINGTIME < DATEADD(HOUR,-@OlderThanInHours,GETDATE())

WHILE (@Count > 0)
BEGIN
	DELETE TOP(@ArhiveBatch) 
	FROM dbo.TMN_DES_EVENT
	OUTPUT DELETED.* 
	INTO dbo.TMN_DES_EVENT_ARH   
	WHERE STAGINGTIME < DATEADD(HOUR,-@OlderThanInHours,GETDATE())
SET @Count -= @ArhiveBatch;
PRINT ('ARHIVED: '+CONVERT(VARCHAR,@ArhiveBatch)+' LEFT TO BE ARHIVED: '+CONVERT(VARCHAR,@Count))
END
GO

--PRIMJER
EXECUTE dbo.PROC_ARHIVE_TMN_DES_EVENT
@ArhiveBatch = 100000,
@OlderThanInHours = 24;
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------






-- NEKA BRZA SKRIPTA
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
use test12_NEW
go

declare @print BIT, @bingo VARCHAR(max)
set @print = 0;

DECLARE foreach CURSOR FOR 
    select [name] from [test12].sys.tables
	OPEN foreach  
          FETCH NEXT FROM foreach INTO @bingo;
        	WHILE @@FETCH_STATUS = 0  
        BEGIN 
		--print @bingo
        	SET @bingo = 'CREATE VIEW dbo.[' + @bingo + '] AS SELECT * FROM arhivadb.ar12.['+ @bingo + '] GO';
			        	IF(@print = 0) EXECUTE (@bingo) ELSE PRINT (@bingo);     
				  		FETCH NEXT FROM foreach INTO @bingo;
        END 
          
        CLOSE foreach  
        DEALLOCATE foreach 	
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------






-- FILEGROUPE
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
ALTER DATABASE [Archives] MODIFY FILE (NAME = 'Archivesdb_2023', OFFLINE)

ALTER DATABASE arhivadb
    MODIFY FILE ( NAME = Archivesdb_2023,   
                  FILENAME = 'M:\MSSQL\Data\Archivesdb_2023.ndf');  
GO

use [master]
RESTORE DATABASE [Archives] FILEGROUP = 'AR_2023' WITH RECOVERY


 -- FILE GROUP PROVJERA:
 SELECT [databasefile].NAME      AS [FileName],
  [filegroup].NAME       AS [File_Group_Name],
  [filegroup].type_desc,
  physical_name [Data File Location],
  size / 128    AS [Size_in_MB],
  state_desc    [State of FILE],
  growth        [Data file growth]
FROM   sys.database_files [databasefile]
  INNER JOIN sys.filegroups [filegroup]
          ON [databasefile].data_space_id = [filegroup].data_space_id  
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------









----------------------------------------------------------------------------------------------------------------
-- deadlock simulacija
-- Create sample tables
CREATE TABLE TableA (ID INT PRIMARY KEY, ValueA VARCHAR(50));
CREATE TABLE TableB (ID INT PRIMARY KEY, ValueB VARCHAR(50));
INSERT INTO TableA (ID, ValueA) VALUES (1, 'A1'), (2, 'A2');
INSERT INTO TableB (ID, ValueB) VALUES (1, 'B1'), (2, 'B2');

-- Transaction 1
BEGIN TRANSACTION;
UPDATE TableA SET ValueA = 'A1_updated' WHERE ID = 1;
-- Transaction 2
BEGIN TRANSACTION;
UPDATE TableB SET ValueB = 'B1_updated' WHERE ID = 1;
-- Transaction 1
UPDATE TableB SET ValueB = 'B2_updated' WHERE ID = 2;
-- Transaction 2
UPDATE TableA SET ValueA = 'A2_updated' WHERE ID = 2;
--------------------------------------------------------
-- lock simulacija
  begin tran

  delete 
  FROM [test].[dbo].[Accounts]
  where id = '4800'
  
  SELECT top 5 *
  FROM [test].[dbo].[Accounts]
  ----------------------------------------------------------------------------------------------------------------
  -- deadlockEvents
--------------------------------------
CREATE EVENT SESSION deadlckCapture
ON SERVER
ADD EVENT sqlserver.xml_deadlock_report
ADD TARGET package0.event_file(SET filename=N'C:\UPDATE\dlckSessions.xel',max_file_size=(10),max_rollover_files=(5))
WITH (MAX_MEMORY=4096 KB,EVENT_RETENTION_MODE=ALLOW_SINGLE_EVENT_LOSS,MAX_DISPATCH_LATENCY=30 SECONDS,MAX_EVENT_SIZE=0 KB,MEMORY_PARTITION_MODE=NONE,TRACK_CAUSALITY=OFF,STARTUP_STATE=ON)
GO
ALTER EVENT SESSION deadlckCapture ON SERVER STATE = START;
ALTER EVENT SESSION deadlckCapture ON SERVER WITH (STARTUP_STATE=ON);
GO

SELECT
    event_data.value('(event/@timestamp)[1]', 'DATETIME') AS DeadlockStartTime,
    deadlock_node.value('@hostname', 'NVARCHAR(256)') AS Hostname,
    deadlock_node.value('@loginname', 'NVARCHAR(256)') AS LoginName,
    deadlock_node.value('@spid', 'INT') AS SPID,
    deadlock_node.value('(inputbuf)[1]', 'NVARCHAR(MAX)') AS SQLText,
    resource_node.value('@objectname', 'NVARCHAR(256)') AS ObjectName,
    CASE 
        WHEN deadlock_node.value('@id', 'NVARCHAR(256)') = victim_node.value('@id', 'NVARCHAR(256)') 
        THEN 1 
        ELSE 0 
    END AS Victim,
    CASE 
        WHEN deadlock_node.value('@id', 'NVARCHAR(256)') = victim_node.value('@id', 'NVARCHAR(256)') 
        THEN 'Yes' 
        ELSE 'No' 
    END AS Evicted
FROM (
    SELECT CAST(event_data AS XML) AS event_data
    FROM sys.fn_xe_file_target_read_file('C:\UPDATE\dlckSessions*.xel', NULL, NULL, NULL)
) AS Data
CROSS APPLY Data.event_data.nodes('//event[@name="xml_deadlock_report"]/data[@name="xml_report"]/value/deadlock/process-list/process') AS ProcessNode (deadlock_node)
CROSS APPLY Data.event_data.nodes('//event[@name="xml_deadlock_report"]/data[@name="xml_report"]/value/deadlock/resource-list/keylock') AS ResourceNode (resource_node)
CROSS APPLY Data.event_data.nodes('//event[@name="xml_deadlock_report"]/data[@name="xml_report"]/value/deadlock/victim-list/victimProcess') AS VictimNode (victim_node)
ORDER BY DeadlockStartTime DESC;


-- blockEvents
--------------------------------------
CREATE EVENT SESSION blckCapture
ON SERVER
ADD EVENT sqlserver.blocked_process_report(
    ACTION (
        sqlserver.sql_text,
        sqlserver.session_id,
        sqlserver.username,
        sqlserver.client_hostname
    )
)
ADD TARGET package0.event_file(SET filename=N'C:\UPDATE\blckSessions.xel',max_file_size=(100),max_rollover_files=(5))
WITH (MAX_MEMORY=4096 kB,EVENT_RETENTION_MODE=ALLOW_SINGLE_EVENT_LOSS,MAX_DISPATCH_LATENCY=36000 SECONDS,MAX_EVENT_SIZE=0 KB,MEMORY_PARTITION_MODE=NONE,TRACK_CAUSALITY=OFF,STARTUP_STATE=ON)
GO

ALTER EVENT SESSION blckCapture ON SERVER STATE = START;
ALTER EVENT SESSION blckCapture ON SERVER WITH (STARTUP_STATE=ON);
GO

;with blckData AS (
    SELECT
        DATEADD(HOUR, 1, event_data.value('(event/@timestamp)[1]', 'DATETIME')) AS EventTime,
        blocked_process.value('@spid', 'INT') AS BlockedSPID,
		blocking_process.value('@spid', 'INT') AS BlockingSPID,
        blocked_process.value('@hostname', 'NVARCHAR(256)') AS BlockedHostname,
        blocked_process.value('@loginname', 'NVARCHAR(256)') AS BlockedLoginName,
        blocked_process.value('(inputbuf)[1]', 'NVARCHAR(MAX)') AS BlockedSQLText,
        blocking_process.value('@hostname', 'NVARCHAR(256)') AS BlockingHostname,
        blocking_process.value('@loginname', 'NVARCHAR(256)') AS BlockingLoginName,
        blocking_process.value('(inputbuf)[1]', 'NVARCHAR(MAX)') AS BlockingSQLText
    FROM (
        SELECT CAST(event_data AS XML) AS event_data
        FROM sys.fn_xe_file_target_read_file('C:\UPDATE\blckSessions*.xel', NULL, NULL, NULL)
    ) AS Data
    CROSS APPLY event_data.nodes('//event[@name="blocked_process_report"]/data[@name="blocked_process"]/value/blocked-process-report') AS XEventData (blocked_report)
    CROSS APPLY XEventData.blocked_report.nodes('blocked-process/process') AS BlockedProcessNode (blocked_process)
    CROSS APPLY XEventData.blocked_report.nodes('blocking-process/process') AS BlockingProcessNode (blocking_process)
)
,blckData2 AS (SELECT
	CONVERT(VARCHAR(19), MIN(EventTime), 120) AS Eventime_start,
	CONVERT(VARCHAR(19), MAX(EventTime), 120) AS Eventime_last,
	DATEDIFF(SECOND, MAX(EventTime), MIN(EventTime)) as Lasted,
	BlockingSPID,
    BlockingHostname,
    BlockingLoginName,
    BlockingSQLText,
    BlockedSPID,
    BlockedHostname,
    BlockedLoginName,
    BlockedSQLText
FROM blckData
GROUP BY BlockedSPID, BlockedHostname, BlockedLoginName, BlockedSQLText, BlockingSPID, BlockingHostname, BlockingLoginName, BlockingSQLText
)
SELECT
	Eventime_start
	,Eventime_last
	,ABS(Lasted) AS Lasted
	,BlockingSPID
	,BlockedSPID
    ,BlockingHostname
    ,BlockingLoginName
    ,BlockingSQLText
    ,BlockedHostname
    ,BlockedLoginName
    ,BlockedSQLText
from blckData2 ORDER BY Eventime_last DESC;


-- prvo je potrebno aktivirati 
exec sp_configure 'show advanced options',1;
GO
RECONFIGURE;
GO

exec sp_configure 'blocked process threshold (s)',5;
GO
RECONFIGURE;
GO

select * from sys.configurations where name = N'blocked process threshold (s)';























-- provjera da li pada job trenutno
WITH LastExecution AS (
    SELECT TOP 1 CD.job_id, CD.start_execution_date
    FROM msdb.dbo.sysjobactivity AS CD
    INNER JOIN msdb.dbo.sysjobs AS j ON j.job_id = CD.job_id
    WHERE j.name = 'RESTORE LOGS'
    ORDER BY CD.start_execution_date DESC
)
SELECT *
FROM msdb.dbo.sysjobs AS j
INNER JOIN msdb.dbo.sysjobsteps AS js ON js.job_id = j.job_id
INNER JOIN msdb.dbo.sysjobhistory AS jh ON jh.job_id = j.job_id AND jh.step_id = js.step_id
WHERE jh.run_status = 0
AND j.name = 'RESTORE LOGS'
AND EXISTS (
    SELECT 1
    FROM LastExecution LE
    WHERE j.job_id = LE.job_id
    AND jh.run_date = CONVERT(varchar, LE.start_execution_date, 112)
    AND jh.run_time >= CONVERT(int, FORMAT(LE.start_execution_date, 'HHmmss'))
)
ORDER BY jh.run_date DESC, jh.run_time DESC;

  
  
  
  
  
  
  
  
  
    ----------------------------------------------------------------------------------------------------------------
  
  -- ZA Krema PROVJERE
  DECLARE @count INT, @countTrenutni INT, @BODY VARCHAR(250);
SET @count = (SELECT COUNT(*) FROM [dbo].[F_EB_EOD_ERROR_DETAIL] with(nolock));

WHILE 1 = 1
BEGIN
    SET @countTrenutni = (SELECT COUNT(*) FROM [dbo].[F_EB_EOD_ERROR_DETAIL] with(nolock));
    IF @countTrenutni > @count
		 BEGIN
				SET @count = @countTrenutni;  

				SET @BODY = N'<H2 style="background-color:#FAA0A0;">' + ' F_EB_EOD_ERROR' + '</H2>' +
				            ' Poštovani,' + '<br/>' +  '<br/>' + 'Upisan je novi row u tabeli F_EB_EOD_ERROR.'  + '<br/>' + 
				            'Lijep pozdrav.';	
				EXEC msdb.dbo.sp_send_dbmail
				    @profile_name = 'MAIL',
				    @recipients = 'cob.operators@TAMBURAgroup.ba;amar.abaz@TAMBURAgroup.ba',
				    @body = @BODY,
				    @subject = 'F_EB_EOD_ERROR - New Insert',
				    @body_format = 'HTML';
		END
		ELSE PRINT 'Nema novih unosa.'  
    WAITFOR DELAY '00:00:10';  
END












</code></pre>
