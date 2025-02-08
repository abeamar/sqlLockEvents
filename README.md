
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

