
# Block and Deadlock monitor <img src="https://cdn-dynmedia-1.microsoft.com/is/image/microsoftcorp/UHFbanner-MSlogo?fmt=png-alpha&bfc=off&qlt=100,1" align="right" width="150">
Showcase of my way to track blocking and deadlock sessions occurrence history with all the information needed to be analysed, using views and extended events. Created for MS SQL.  
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
    <br>
     <section id="about">
        <h2>1. About</h2>
        <p>This is the section for "About".</p>
    </section>
        <br>
            <hr>
    <section id="blocking-event">
        <h2>2. Blocking Event</h2>
        <p>This is the section for "Blocking Event".</p>
  <div class="copy-bar">
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
        <p>This is the section for "Deadlock Event".</p>

<div class="copy-bar">
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
        <p>This is the section for "Create Views".</p>
    </section>
        <br>
            <hr>
    <section id="conclusion">
        <h2>5. Conclusion</h2>
        <p>This is the section for "Conclusion".</p>
    </section>
        <br>
