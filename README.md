# LanternPowerMonitor
This is a setup guide of the LanternPowerMonitor self hosted service. (https://github.com/MarkBryanMilligan/LanternPowerMonitor)


## Starting off
I used debian 11 and virtualbox because it is just what I had avialable while I was trying to figure out docker.
1. Download and install VirtualBox (https://www.virtualbox.org)
2. Download Debian 11 (https://cdimage.debian.org/mirror/cdimage/archive/11.11.0/amd64/iso-dvd/debian-11.11.0-amd64-DVD-1.iso)
3. Install VirtualBox and create Debian VM. Add shared folder on host machine to VM location /mnt/host/
4. Once Debian 11 is installed, open a terminal

<br/>

**NEW TERMINAL WINDOW 1**

This sets up the environment I used to compile the server's service binaries.
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install openjdk-17-jre openjdk-17-jdk
wget https://github.com/MarkBryanMilligan/LanternPowerMonitor/archive/refs/heads/main.zip
unzip main.zip
mv LanternPowerMonitor-main/ LanternPowerMonitor/
cd LanternPowerMonitor/java
chmod +x gradlew
```
What these commands did was to update and upgrade the operating system. Then download, unzip, and prep the LanternPowerMonitor service.

<br/>

**NEW TERMINAL WINDOW 2**

I may have beening doing this section wrong but this is how I got it to work. Normally after compiling you should have been able to run
```
sudo java -jar lantern-powermonitor-config-2.0.0-DEV-all.jar com.lanternsoftware.currentmonitor.CreateAuthKey
sudo java -jar lantern-powermonitor-config-2.0.0-DEV-all.jar com.lanternsoftware.currentmonitor.CreateMongoConfig MongoDB_IP MongoDB_User MongoDB_Password
```
and this would have created the authKey.dat and mongo.cfg files in /opt/tomcat/. However this would not work for me and I never figured out why. So I modified the source file to get the config I needed for my self-hosted service.

```
cd ~/LanternPowerMonitor/java/lantern-powermonitor-config/src/main/java/com/lanternsoftware/powermonitor
nano CreateMongoConfig.java
```
modify line
```
new MongoConfig("MONGOHOST", "MONGOUSER", "MONGOPASS", "CURRENT_MONITOR").saveToDisk(LanternFiles.CONFIG_PATH + "mongo.cfg");
```
to your IP, MongoDB user and password
```
new MongoConfig("MongoDB_IP", "MongoDB_User", "MongoDB_Password", "CURRENT_MONITOR").saveToDisk(LanternFiles.CONFIG_PATH + "mongo.cfg");
```
Ctrl+O to save file and Ctrl+X to exit

<br/>

** MOVE BACK TO TERMINAL WINDOW 1 **

This actively compiles the source code into usable binaries.
```
sudo ./gradlew clean build publishToMavenLocal
```

<br/>

**MOVE BACK TO TERMINAL WINDOW 2**

This section builds the authKey.dat and moongo.cfg files that are needed for the Tomcat server LanternPowerMonitor service. One thing I figured out during this portion was that the CreateAuthKey would generate a mongo.cfg file rather than the authKey.dat file. So you have to generate the file and then mv/renmae it before generating the config. The copy commands just copied the files to a shared folder so you can get them off your virtual machine, if you do not have drag and drop enabled.
```
cd ~/LanternPowerMonitor/java/lantern-powermonitor-config/build/libs
sudo java -jar lantern-powermonitor-config-2.0.0-DEV-all.jar com.lanternsoftware.currentmonitor.CreateAuthKey
sudo mv /opt/tomcat/mongo.cfg /opt/tomcat/authKey.dat
sudo java -jar lantern-powermonitor-config-2.0.0-DEV-all.jar com.lanternsoftware.currentmonitor.CreateMongoConfig MongoDB_IP MongoDB_User MongoDB_Password
sudo cp /opt/tomcat/authKey.dat /mnt/host/authKey.dat
sudo cp /opt/tomcat/mongo.cfg /mnt/host/mongo.cfg
sudo cp ~/LanternPowerMonitor/java/lantern-powermonitor-service/build/libs/lantern-powermonitor-service-2.0.0-DEV.war /mnt/host/currentmonitor.war
```

<br/>

## Diverging ##
This is where everyone will go separate ways. You can have the Tomcat/MongoDB hosted on a raspiberry pi, within a server's docking container, or in some other hosting fashion. The rest of this guide was on my setup. I already had a home server running Unraid so I chose to run Tomcat/MongoDB within my docker environment.

### MongoDB ###
I started by creating a MongoDB container using the following options.  
![image](https://github.com/user-attachments/assets/ef42e267-f198-4489-b259-2d2b91ec4e33)

I chose to have docker create an initial database/User/Password because I was also setting up a MongoDB_Express container to monitor/test everything. Also you might be able to use mongo:latest for the repository but I had to downgrade and lock mine to 4.4.18 because my server's processor does not support AVP. Once it was done installing I went into the mongo container console and ran the following commands to setup the admin user for tomcat.
```
mongo -u root -p example
use admin
db.createUser(
  {
    user: "powerlanternuser",
    pwd: "powerlanternpasssword", // or cleartext password
    roles: [
      { role: "userAdminAnyDatabase", db: "admin" },
      { role: "readWriteAnyDatabase", db: "admin" }
    ]
  }
)
```
You best bet with this is to run the first two commands and then copy and paste the entire db.createUser().

<br/>

### MongoDB_Express Container ###
I setup another container and installed the latest version of MongoDB_express with the following config.  
![image](https://github.com/user-attachments/assets/f64ba7ac-1233-4e3e-9d0f-4bcb39ba2c92)

After it is installed you should be able to goto the IP:Port and login. This will allow you to monitor and configure any MongoDB database.  
![image](https://github.com/user-attachments/assets/319508ee-29c8-4ff4-9de9-9a75ec8cc065)
Yours shouldn't have CURRENT_MONITOR yet, this is a screenshot from after everything is installed.  

<br/>

### Tomcat Container ###
This was a PITA. I started off by setting up the tomcat docker container with the following config. I set my tomcat container to host network instead of bridge. I am not sure why I had to do this instead of bridging. I think this was due to the self signed SSL certificates, but I am not sure. (This might also be a hold over from when I was testing the docker containers on my host system and not the server. However since I have this on host instead of bridge the ports are irrelevant to the setup of the docker container.
![image](https://github.com/user-attachments/assets/ee11affa-c838-4e30-9d0f-2e08b42bcdf2)

Once the container was up, I opened the docker console to the container and I modified the tomcat server in the following ways:

1) Uncommented following lines in /usr/local/tomcat/tomcat-users.xml
```
  <role rolename="manager-gui"/>
  <user username="admin" password="pass" roles="manager-gui"/>
  <user username="robot" password="pass" roles="manager-script"/>
```

2) I then copied all of the folders from /usr/local/tomcat/webapps.dist to /usr/local/tomcat/webapps. You could get away with only copying the manager folder, but I chose to bring everything over.

3) In order to access the manager, I had to modify /usr/local/tomcat/webapp/manager/META-INF/context.xml. I had to comment out this section to be able to access the tomcat manager.
```
<Context antiResourceLocking="false" privileged="true" >
  <CookieProcessor className="org.apache.tomcat.util.http.Rfc6265CookieProcessor"
                   sameSiteCookies="strict" />
<!--
  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />
  <Manager sessionAttributeValueClassNameFilter="java\.lang\.(?:Boolean|Integer|Long|Number|String)|org\.apache\.catalina\.filters\.CsrfPreventionFilter\$LruCache(?:\$1)?|java\.util\.(?:Linked)?HashMap"/>
-->
</Context>
```

4) I then ran the command
```
$JAVA_HOME/bin/keytool -genkey -alias tomcat -keyalg RSA -keystore /usr/local/tomcat/conf/localhost-rsa.jks
```
to generate the self-signed SSL certificates.

5) I uncomment the "SSL/TLS HTTP/1.1 Connector on port 8443" section in /usr/local/tomcat/conf/server.xml
```
    <!-- Define an SSL/TLS HTTP/1.1 Connector on port 8443
         This connector uses the NIO implementation. The default
         SSLImplementation will depend on the presence of the APR/native
         library and the useOpenSSL attribute of the AprLifecycleListener.
         Either JSSE or OpenSSL style configuration may be used regardless of
         the SSLImplementation selected. JSSE style configuration is used below.
    -->

    <Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
               maxThreads="150" SSLEnabled="true"
               maxParameterCount="1000"
               >
        <SSLHostConfig>
            <Certificate certificateKeystoreFile="conf/localhost-rsa.jks"
                         certificateKeystorePassword="changeit" type="RSA" />
        </SSLHostConfig>
    </Connector>
```
This will point the SSL connector to the SSL certificates that were generated in step 4 above.

6) I made a copy of the entire directory /usr/local/tomcat/ into another directory. (I think I used a folder in my home directory)

After restarting the container, I could then reach the tomcat manager (http://Unraid_IP:8080/manager/html)

Then because I was testing, I shut down the Tomcat container and modified it to:
![image](https://github.com/user-attachments/assets/e42b5699-c1b6-4c61-8e0c-64628c13b072)
I mounted the /opt/tomcat and /usr/local/tomcat directories so I could access them from the server system and not have to use the console again. I am not sure if it was my testing or something I did, but this broke my Tomcat container. The tomcat directory was empty when I did this, so I just copied the tomcat backup in the home directory back into /usr/local/tomcat and the container started working again.

<br/>

### Tomcat PowerLanternService setup ###
I copied the authKey.dat and mongo.cfg from the debian shared folder to the Tomcat container's /opt/tomcat folder.  

I went to http://Unraid_IP:8080/manager/html/ and logged in.  
![image](https://github.com/user-attachments/assets/e2559d8d-69eb-43b7-8c62-9154612744ef)

I chose the WAR file from the debian shared folder and clicked deploy. This takes a few so be patience. You can also go to the MongoDB_Express if you have that setup and check if you have a CURRENT_MONITOR database yet.

Eventually the Tomcat webpage will come back and hopefully have /currentmonitor installed.
![image](https://github.com/user-attachments/assets/65480d67-590d-4fc8-b5a9-c5f80b8dfb10)

You should now be able to get to https://Unraid_IP:8443/currentmonitor/login
![image](https://github.com/user-attachments/assets/107dda89-2f1f-4cd3-a9dc-9bc1a2e3bf6c)

Go to your Lantern Power Monitor mobile app and get to the login screen.

![image](https://github.com/user-attachments/assets/f4c9e971-3b5b-4152-9179-7628bcf22b68)

Click on the bottom right, "Host" and change it from "LanternPowermonitor.com" to "https://Unraid_IP:8443".  
You can then click on signup and create an account. If everything is setup correctly you should be able to login. You can also see your account get created in MongoDB_Express, CURRENT_MONITOR\account.  

<br/>

## ENERGY HUB SETUP ##
Not a lot has to happen here other than enabling "accept_self_signed_certificates".  
To do this SSH to the IP of the energy hubs. Default user is "pi" and password is "LanternPowerMonitor"  
Then go to /opt/currentmonitor and modify the config.json file. Change:  
```
  "accept_self_signed_certificates": false,
```
to true and restart the service with the command:
```
sudo systemctl restart currentmonitor.service
or
sudo reboot
```
You should be able to use the mobile app to adopt your energy hubs.

<br/>

## ENERGY HUB CALIBRATION ##
After adopting energy hubs and creating a panel on the mobile app, you then need to calibrate them.  
I had a multimeter and I bought a digital currrent clamp tester (www.amazon.com/dp/B0CVXCFMLY?ref_=ppx_hzsearch_conn_dt_b_fed_asin_title_2).  
Using the multimeter on the 12V AC/AC voltage transformer, I connected a gator clamp to one side and testing probe to the other.  
![image](https://github.com/user-attachments/assets/963c120b-f854-4a0b-9cc5-47a48c079522)
![image](https://github.com/user-attachments/assets/da7de150-30e7-4b3f-8b4e-f85c1a5f2971)

This gave me the Hub # V for the panel calibration.

![image](https://github.com/user-attachments/assets/da9f8c86-ef26-4d19-b059-b695a3d932d3)


I swapped it from watts to amps in the mobile app. I then took my digital current clamp tester and hooked it to a circuit on that hub that was a stable number and not jumping around.
![image](https://github.com/user-attachments/assets/544edcee-2cd5-48e2-8019-62a78e0da370)

I tweaked the Hub # CT number on the mobile app to match the digital current clamp reading, then saved the settings.

<br/>

## OTHER NOTES ##
Log Locations  
<ins>Tomcat Container</ins>  
/usr/local/tomcat/logs/  
catalina.DATE.log - Tomcat general logs. Useful for troubleshooting the currentmonitor.war deployment.  
localhost.DATE.log - Tomcat error logs. Useful for troubleshooting the currentmonitor.war deployment.  
localhost_access_log.DATE.txt - Access logs for what devices are talking to Tomcat. Good for checking if your Energy hubs are talking to tomcat.  
manager.DATE.log - General Context info  

/opt/tomcat/log/  
log.txt - CurrentMonitor.war logging. This will show what the tomcat power monitor service is doing or any errors that are happening. This will also show energy hubs checking in and updating power readings.  

<ins>Energy Hub</ins>  
/opt/currentmonitor/log/  
log.txt - Basic Energy hub logs. If you are having issues with the energy hub and tomcat server talking check here for any errors.  

<br/>

### TroubleShooting ###
1) This error was in the tomcat logging and was caused by compiling the source with the wrong version of Java.
```
-----------------------Tomcat Catalina Log-----------------------
2025-03-23 14:21:03 23-Mar-2025 19:21:03.399 SEVERE [http-nio-8080-exec-5] org.apache.catalina.core.StandardContext.startInternal One or more listeners failed to start. Full details will be found in the appropriate container log file
2025-03-23 14:21:03 23-Mar-2025 19:21:03.401 SEVERE [http-nio-8080-exec-5] org.apache.catalina.core.StandardContext.startInternal Context [/lantern-powermonitor-service-2.0.0-DEV] startup failed due to previous errors

-----------------------Tomcat Localhost log-----------------------
23-Mar-2025 19:21:03.398 SEVERE [http-nio-8080-exec-5] org.apache.catalina.core.StandardContext.listenerStart Error configuring application listener of class [com.lanternsoftware.powermonitor.context.Globals]
	java.lang.UnsupportedClassVersionError: com/lanternsoftware/powermonitor/context/Globals has been compiled by a more recent version of the Java Runtime (class file version 55.0), this version of the Java Runtime only recognizes class file versions up to 52.0 (unable to load class [com.lanternsoftware.powermonitor.context.Globals])
		at org.apache.catalina.loader.WebappClassLoaderBase.findClassInternal(WebappClassLoaderBase.java:2347)
		at org.apache.catalina.loader.WebappClassLoaderBase.findClassInternal(WebappClassLoaderBase.java:2212)
		at org.apache.catalina.loader.WebappClassLoaderBase.findClass(WebappClassLoaderBase.java:823)
		at org.apache.catalina.loader.WebappClassLoaderBase.loadClass(WebappClassLoaderBase.java:1314)
		at org.apache.catalina.loader.WebappClassLoaderBase.loadClass(WebappClassLoaderBase.java:1162)
		at org.apache.catalina.core.DefaultInstanceManager.loadClass(DefaultInstanceManager.java:488)
		at org.apache.catalina.core.DefaultInstanceManager.loadClassMaybePrivileged(DefaultInstanceManager.java:470)
		at org.apache.catalina.core.DefaultInstanceManager.newInstance(DefaultInstanceManager.java:142)
		at org.apache.catalina.core.StandardContext.listenerStart(StandardContext.java:3986)
		at org.apache.catalina.core.StandardContext.startInternal(StandardContext.java:4501)
		at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:164)
		at org.apache.catalina.manager.ManagerServlet.start(ManagerServlet.java:1303)
		at org.apache.catalina.manager.HTMLManagerServlet.start(HTMLManagerServlet.java:642)
		at org.apache.catalina.manager.HTMLManagerServlet.doPost(HTMLManagerServlet.java:188)
		at javax.servlet.http.HttpServlet.service(HttpServlet.java:555)
		at javax.servlet.http.HttpServlet.service(HttpServlet.java:623)
		at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:199)
		at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:144)
		at org.apache.catalina.filters.CsrfPreventionFilter.doFilter(CsrfPreventionFilter.java:430)
		at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:168)
		at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:144)
		at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:51)
		at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:168)
		at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:144)
		at org.apache.catalina.filters.HttpHeaderSecurityFilter.doFilter(HttpHeaderSecurityFilter.java:129)
		at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:168)
		at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:144)
		at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:168)
		at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:90)
		at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:597)
		at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:130)
		at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:93)
		at org.apache.catalina.valves.AbstractAccessLogValve.invoke(AbstractAccessLogValve.java:660)
		at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:74)
		at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:346)
		at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:396)
		at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:63)
		at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:937)
		at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1793)
		at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:52)
		at org.apache.tomcat.util.threads.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1190)
		at org.apache.tomcat.util.threads.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:659)
		at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:63)
		at java.lang.Thread.run(Thread.java:750)
23-Mar-2025 19:21:03.398 SEVERE [http-nio-8080-exec-5] org.apache.catalina.core.StandardContext.listenerStart Skipped installing application listeners due to previous error(s)
-----------------------Tomcat Localhost log-----------------------
```

2) You forgot to rename lantern-powermonitor-service-2.0.0-DEV.war to currentmonitor.war.
```
-----------------------Tomcat Localhost log-----------------------
2025-03-24 15:36:06 24-Mar-2025 20:36:06.964 INFO [http-nio-8080-exec-58] org.apache.jasper.servlet.TldScanner.scanJars At least one JAR was scanned for TLDs yet contained no TLDs. Enable debug logging for this logger for a complete list of JARs that were scanned but no TLDs were found in them. Skipping unneeded JARs during scanning can improve startup time and JSP compilation time.
2025-03-24 15:36:37 24-Mar-2025 20:36:37.289 SEVERE [http-nio-8080-exec-58] org.apache.catalina.core.StandardContext.startInternal One or more listeners failed to start. Full details will be found in the appropriate container log file
2025-03-24 15:36:37 24-Mar-2025 20:36:37.290 SEVERE [http-nio-8080-exec-58] org.apache.catalina.core.StandardContext.startInternal Context [/lantern-powermonitor-service-2.0.0-DEV] startup failed due to previous errors
2025-03-24 15:36:37 24-Mar-2025 20:36:37.292 WARNING [http-nio-8080-exec-58] org.apache.catalina.loader.WebappClassLoaderBase.clearReferencesThreads The web application [lantern-powermonitor-service-2.0.0-DEV] appears to have started a thread named [Catalina-utility-2] but has failed to stop it. This is very likely to create a memory leak. Stack trace of thread:
2025-03-24 15:36:37  org.apache.catalina.core.StandardContext.backgroundProcess(StandardContext.java:4849)
2025-03-24 15:36:37  org.apache.catalina.core.ContainerBase$ContainerBackgroundProcessor.processChildren(ContainerBase.java:1172)
2025-03-24 15:36:37  org.apache.catalina.core.ContainerBase$ContainerBackgroundProcessor.processChildren(ContainerBase.java:1176)
2025-03-24 15:36:37  org.apache.catalina.core.ContainerBase$ContainerBackgroundProcessor.processChildren(ContainerBase.java:1176)
2025-03-24 15:36:37  org.apache.catalina.core.ContainerBase$ContainerBackgroundProcessor.run(ContainerBase.java:1154)
2025-03-24 15:36:37  java.base@17.0.14/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:539)
2025-03-24 15:36:37  java.base@17.0.14/java.util.concurrent.FutureTask.runAndReset(FutureTask.java:305)
2025-03-24 15:36:37  java.base@17.0.14/java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:305)
2025-03-24 15:36:37  java.base@17.0.14/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1136)
2025-03-24 15:36:37  java.base@17.0.14/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:635)
2025-03-24 15:36:37  org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:63)
2025-03-24 15:36:37  java.base@17.0.14/java.lang.Thread.run(Thread.java:840)
2025-03-24 15:36:37 24-Mar-2025 20:36:37.293 WARNING [http-nio-8080-exec-58] org.apache.catalina.loader.WebappClassLoaderBase.clearReferencesThreads The web application [lantern-powermonitor-service-2.0.0-DEV] appears to have started a thread named [Timer-1] but has failed to stop it. This is very likely to create a memory leak. Stack trace of thread:
2025-03-24 15:36:37  java.base@17.0.14/java.lang.Object.wait(Native Method)
2025-03-24 15:36:37  java.base@17.0.14/java.lang.Object.wait(Object.java:338)
2025-03-24 15:36:37  java.base@17.0.14/java.util.TimerThread.mainLoop(Timer.java:537)
2025-03-24 15:36:37  java.base@17.0.14/java.util.TimerThread.run(Timer.java:516)
2025-03-24 15:36:37 24-Mar-2025 20:36:37.293 WARNING [http-nio-8080-exec-58] org.apache.catalina.loader.WebappClassLoaderBase.clearReferencesThreads The web application [lantern-powermonitor-service-2.0.0-DEV] appears to have started a thread named [BufferPoolPruner-1-thread-1] but has failed to stop it. This is very likely to create a memory leak. Stack trace of thread:
2025-03-24 15:36:37  java.base@17.0.14/jdk.internal.misc.Unsafe.park(Native Method)
2025-03-24 15:36:37  java.base@17.0.14/java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:252)
2025-03-24 15:36:37  java.base@17.0.14/java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:1679)
2025-03-24 15:36:37  java.base@17.0.14/java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:1182)
2025-03-24 15:36:37  java.base@17.0.14/java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:899)
2025-03-24 15:36:37  java.base@17.0.14/java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1062)
2025-03-24 15:36:37  java.base@17.0.14/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1122)
2025-03-24 15:36:37  java.base@17.0.14/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:635)
2025-03-24 15:36:37  java.base@17.0.14/java.lang.Thread.run(Thread.java:840)
-----------------------Tomcat Localhost log-----------------------
```

3) The compiled mongo.cfg file is incorrect, it is not going to your server but to lanternsoftware.com.
```
-----------------------Tomcat log-----------------------
24-Mar-2025 20:36:37.289 SEVERE [http-nio-8080-exec-58] org.apache.catalina.core.StandardContext.listenerStart Exception sending context initialized event to listener instance of class [com.lanternsoftware.powermonitor.context.Globals]
	com.mongodb.MongoTimeoutException: Timed out after 30000 ms while waiting for a server that matches WritableServerSelector. Client view of cluster state is {type=UNKNOWN, servers=[{address=lanternsoftware.com:27017, type=UNKNOWN, state=CONNECTING, exception={com.mongodb.MongoSocketOpenException: Exception opening socket}, caused by {java.net.SocketTimeoutException: Connect timed out}}]
		at com.mongodb.internal.connection.BaseCluster.createTimeoutException(BaseCluster.java:380)
		at com.mongodb.internal.connection.BaseCluster.selectServer(BaseCluster.java:125)
		at com.mongodb.internal.connection.SingleServerCluster.selectServer(SingleServerCluster.java:46)
		at com.mongodb.internal.binding.ClusterBinding.getWriteConnectionSource(ClusterBinding.java:134)
		at com.mongodb.client.internal.ClientSessionBinding.getConnectionSource(ClientSessionBinding.java:128)
		at com.mongodb.client.internal.ClientSessionBinding.getWriteConnectionSource(ClientSessionBinding.java:102)
		at com.mongodb.internal.operation.SyncOperationHelper.withSuppliedResource(SyncOperationHelper.java:144)
		at com.mongodb.internal.operation.SyncOperationHelper.withSourceAndConnection(SyncOperationHelper.java:125)
		at com.mongodb.internal.operation.MixedBulkWriteOperation.lambda$execute$3(MixedBulkWriteOperation.java:188)
		at com.mongodb.internal.operation.MixedBulkWriteOperation.lambda$decorateWriteWithRetries$0(MixedBulkWriteOperation.java:146)
		at com.mongodb.internal.async.function.RetryingSyncSupplier.get(RetryingSyncSupplier.java:67)
		at com.mongodb.internal.operation.MixedBulkWriteOperation.execute(MixedBulkWriteOperation.java:207)
		at com.mongodb.internal.operation.MixedBulkWriteOperation.execute(MixedBulkWriteOperation.java:77)
		at com.mongodb.client.internal.MongoClientDelegate$DelegateOperationExecutor.execute(MongoClientDelegate.java:173)
		at com.mongodb.client.internal.MongoCollectionImpl.executeSingleWriteRequest(MongoCollectionImpl.java:1085)
		at com.mongodb.client.internal.MongoCollectionImpl.executeDelete(MongoCollectionImpl.java:1058)
		at com.mongodb.client.internal.MongoCollectionImpl.deleteMany(MongoCollectionImpl.java:537)
		at com.mongodb.client.internal.MongoCollectionImpl.deleteMany(MongoCollectionImpl.java:532)
		at com.lanternsoftware.util.dao.mongo.MongoProxy.delete(MongoProxy.java:383)
		at com.lanternsoftware.util.dao.mongo.MongoProxy.delete(MongoProxy.java:376)
		at com.lanternsoftware.powermonitor.dataaccess.MongoPowerMonitorDao.<init>(MongoPowerMonitorDao.java:95)
		at com.lanternsoftware.powermonitor.context.Globals.contextInitialized(Globals.java:31)
		at org.apache.catalina.core.StandardContext.listenerStart(StandardContext.java:4059)
		at org.apache.catalina.core.StandardContext.startInternal(StandardContext.java:4501)
		at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:164)
		at org.apache.catalina.manager.ManagerServlet.start(ManagerServlet.java:1303)
		at org.apache.catalina.manager.HTMLManagerServlet.start(HTMLManagerServlet.java:642)
		at org.apache.catalina.manager.HTMLManagerServlet.doPost(HTMLManagerServlet.java:188)
		at javax.servlet.http.HttpServlet.service(HttpServlet.java:555)
		at javax.servlet.http.HttpServlet.service(HttpServlet.java:623)
		at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:199)
		at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:144)
		at org.apache.catalina.filters.CsrfPreventionFilter.doFilter(CsrfPreventionFilter.java:430)
		at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:168)
		at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:144)
		at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:51)
		at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:168)
		at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:144)
		at org.apache.catalina.filters.HttpHeaderSecurityFilter.doFilter(HttpHeaderSecurityFilter.java:129)
		at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:168)
		at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:144)
		at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:168)
		at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:90)
		at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:597)
		at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:130)
		at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:93)
		at org.apache.catalina.valves.AbstractAccessLogValve.invoke(AbstractAccessLogValve.java:660)
		at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:74)
		at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:346)
		at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:396)
		at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:63)
		at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:937)
		at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1793)
		at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:52)
		at org.apache.tomcat.util.threads.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1190)
		at org.apache.tomcat.util.threads.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:659)
		at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:63)
		at java.base/java.lang.Thread.run(Thread.java:840)
-----------------------Tomcat log-----------------------

-----------------------Tomcat Localhost log-----------------------
2025-03-24 15:36:37 24-Mar-2025 20:36:37.294 WARNING [http-nio-8080-exec-58] org.apache.catalina.loader.WebappClassLoaderBase.clearReferencesThreads The web application [lantern-powermonitor-service-2.0.0-DEV] appears to have started a thread named [cluster-ClusterId{value='67e1c237c33bae0559d3ba1a', description='null'}-lanternsoftware.com:27017] but has failed to stop it. This is very likely to create a memory leak. Stack trace of thread:
2025-03-24 15:36:37  java.base@17.0.14/sun.nio.ch.Net.poll(Native Method)
2025-03-24 15:36:37  java.base@17.0.14/sun.nio.ch.NioSocketImpl.park(NioSocketImpl.java:186)
2025-03-24 15:36:37  java.base@17.0.14/sun.nio.ch.NioSocketImpl.timedFinishConnect(NioSocketImpl.java:553)
2025-03-24 15:36:37  java.base@17.0.14/sun.nio.ch.NioSocketImpl.connect(NioSocketImpl.java:602)
2025-03-24 15:36:37  java.base@17.0.14/java.net.SocksSocketImpl.connect(SocksSocketImpl.java:327)
2025-03-24 15:36:37  java.base@17.0.14/java.net.Socket.connect(Socket.java:633)
2025-03-24 15:36:37  com.mongodb.internal.connection.SocketStreamHelper.initialize(SocketStreamHelper.java:76)
2025-03-24 15:36:37  com.mongodb.internal.connection.SocketStream.initializeSocket(SocketStream.java:104)
2025-03-24 15:36:37  com.mongodb.internal.connection.SocketStream.open(SocketStream.java:78)
2025-03-24 15:36:37  com.mongodb.internal.connection.InternalStreamConnection.open(InternalStreamConnection.java:211)
2025-03-24 15:36:37  com.mongodb.internal.connection.DefaultServerMonitor$ServerMonitorRunnable.lookupServerDescription(DefaultServerMonitor.java:196)
2025-03-24 15:36:37  com.mongodb.internal.connection.DefaultServerMonitor$ServerMonitorRunnable.run(DefaultServerMonitor.java:156)
2025-03-24 15:36:37  java.base@17.0.14/java.lang.Thread.run(Thread.java:840)
-----------------------Tomcat Localhost log-----------------------
```

4) Something is wrong with authKey.dat. Either you pulled the older source before April 7th, 2025 commit or something happened during its generation. Recommend going back and regenerating authKey.dat and make sure to rename properly.
```
-----------------------Tomcat LanternPowerMonitor error log-----------------------
2025-04-07 14:42:06,246 ERROR AESTool - Failed to encrypt data
java.security.InvalidKeyException: Invalid AES key length: 288 bytes
        at java.base/com.sun.crypto.provider.AESCrypt.init(AESCrypt.java:90)
        at java.base/com.sun.crypto.provider.CipherBlockChaining.init(CipherBlockChaining.java:97)
        at java.base/com.sun.crypto.provider.CipherCore.init(CipherCore.java:482)
        at java.base/com.sun.crypto.provider.AESCipher.engineInit(AESCipher.java:338)
        at java.base/javax.crypto.Cipher.implInit(Cipher.java:871)
        at java.base/javax.crypto.Cipher.chooseProvider(Cipher.java:929)
        at java.base/javax.crypto.Cipher.init(Cipher.java:1444)
        at java.base/javax.crypto.Cipher.init(Cipher.java:1375)
        at com.lanternsoftware.util.cryptography.AESTool.encrypt(AESTool.java:179)
        at com.lanternsoftware.util.cryptography.AESTool.encryptToBase64(AESTool.java:140)
        at com.lanternsoftware.powermonitor.dataaccess.MongoPowerMonitorDao.toAuthCode(MongoPowerMonitorDao.java:722)
        at com.lanternsoftware.powermonitor.dataaccess.MongoPowerMonitorDao.authenticateAccount(MongoPowerMonitorDao.java:676)
        at com.lanternsoftware.powermonitor.servlet.SignupServlet.doGet(SignupServlet.java:49)
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:529)
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:623)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:199)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:144)
        at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:51)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:168)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:144)
        at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:168)
        at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:90)
        at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:482)
        at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:130)
        at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:93)
        at org.apache.catalina.valves.AbstractAccessLogValve.invoke(AbstractAccessLogValve.java:660)
        at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:74)
        at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:346)
        at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:396)
        at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:63)
        at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:937)
        at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1793)
        at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:52)
        at org.apache.tomcat.util.threads.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1190)
        at org.apache.tomcat.util.threads.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:659)
        at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:63)
        at java.base/java.lang.Thread.run(Thread.java:840)
-----------------------Tomcat LanternPowerMonitor error log-----------------------
```

5) Energy hub config doesn't have "accept_self_signed_certificates" in config.json set to true.
```
-----------------------LanternPowerMonitor Energy Hub error log-----------------------
2025-04-08 23:15:48,913 ERROR HttpPool - Failed to make http request to https://192.168.150.5:8443/currentmonitor/config
javax.net.ssl.SSLHandshakeException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
        at java.base/sun.security.ssl.Alert.createSSLException(Alert.java:131)
        at java.base/sun.security.ssl.TransportContext.fatal(TransportContext.java:366)
        at java.base/sun.security.ssl.TransportContext.fatal(TransportContext.java:309)
        at java.base/sun.security.ssl.TransportContext.fatal(TransportContext.java:304)
        at java.base/sun.security.ssl.CertificateMessage$T13CertificateConsumer.checkServerCerts(CertificateMessage.java:1357)
        at java.base/sun.security.ssl.CertificateMessage$T13CertificateConsumer.onConsumeCertificate(CertificateMessage.java:1232)
        at java.base/sun.security.ssl.CertificateMessage$T13CertificateConsumer.consume(CertificateMessage.java:1175)
        at java.base/sun.security.ssl.SSLHandshake.consume(SSLHandshake.java:392)
        at java.base/sun.security.ssl.HandshakeContext.dispatch(HandshakeContext.java:443)
        at java.base/sun.security.ssl.HandshakeContext.dispatch(HandshakeContext.java:421)
        at java.base/sun.security.ssl.TransportContext.dispatch(TransportContext.java:189)
        at java.base/sun.security.ssl.SSLTransport.decode(SSLTransport.java:172)
        at java.base/sun.security.ssl.SSLSocketImpl.decode(SSLSocketImpl.java:1511)
        at java.base/sun.security.ssl.SSLSocketImpl.readHandshakeRecord(SSLSocketImpl.java:1421)
        at java.base/sun.security.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:456)
        at java.base/sun.security.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:427)
        at org.apache.http.conn.ssl.SSLConnectionSocketFactory.createLayeredSocket(SSLConnectionSocketFactory.java:396)
        at org.apache.http.conn.ssl.SSLConnectionSocketFactory.connectSocket(SSLConnectionSocketFactory.java:355)
        at org.apache.http.impl.conn.DefaultHttpClientConnectionOperator.connect(DefaultHttpClientConnectionOperator.java:142)
        at org.apache.http.impl.conn.PoolingHttpClientConnectionManager.connect(PoolingHttpClientConnectionManager.java:359)
        at org.apache.http.impl.execchain.MainClientExec.establishRoute(MainClientExec.java:381)
        at org.apache.http.impl.execchain.MainClientExec.execute(MainClientExec.java:237)
        at org.apache.http.impl.execchain.ProtocolExec.execute(ProtocolExec.java:185)
        at org.apache.http.impl.execchain.RetryExec.execute(RetryExec.java:89)
        at org.apache.http.impl.execchain.RedirectExec.execute(RedirectExec.java:111)
        at org.apache.http.impl.client.InternalHttpClient.doExecute(InternalHttpClient.java:185)
        at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:83)
        at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:108)
        at com.lanternsoftware.util.http.HttpPool.execute(HttpPool.java:87)
        at com.lanternsoftware.util.http.HttpPool.executeToPayload(HttpPool.java:121)
        at com.lanternsoftware.util.http.HttpPool.executeToByteArray(HttpPool.java:114)
        at com.lanternsoftware.util.http.HttpPool.executeToString(HttpPool.java:110)
        at com.lanternsoftware.currentmonitor.MonitorApp.main(MonitorApp.java:243)
Caused by: sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
        at java.base/sun.security.validator.PKIXValidator.doBuild(PKIXValidator.java:439)
        at java.base/sun.security.validator.PKIXValidator.engineValidate(PKIXValidator.java:306)
        at java.base/sun.security.validator.Validator.validate(Validator.java:264)
        at java.base/sun.security.ssl.X509TrustManagerImpl.validate(X509TrustManagerImpl.java:313)
        at java.base/sun.security.ssl.X509TrustManagerImpl.checkTrusted(X509TrustManagerImpl.java:222)
        at java.base/sun.security.ssl.X509TrustManagerImpl.checkServerTrusted(X509TrustManagerImpl.java:129)
        at java.base/sun.security.ssl.CertificateMessage$T13CertificateConsumer.checkServerCerts(CertificateMessage.java:1341)
        ... 28 common frames omitted
Caused by: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
        at java.base/sun.security.provider.certpath.SunCertPathBuilder.build(SunCertPathBuilder.java:148)
        at java.base/sun.security.provider.certpath.SunCertPathBuilder.engineBuild(SunCertPathBuilder.java:129)
        at java.base/java.security.cert.CertPathBuilder.build(CertPathBuilder.java:297)
        at java.base/sun.security.validator.PKIXValidator.doBuild(PKIXValidator.java:434)
        ... 34 common frames omitted
2025-04-08 23:15:48,918 ERROR MonitorApp - Failed to load breaker config.  Retrying in 5 seconds...
-----------------------LanternPowerMonitor Energy Hub error log-----------------------
```


<br/>

### Misc/My Setup ###
Boards

![image](https://github.com/user-attachments/assets/7aacfec2-737f-4151-bf01-9717f521f55d)
![image](https://github.com/user-attachments/assets/909d6981-8b59-4f0e-9e34-74337128b79e)

Starting Solder

![image](https://github.com/user-attachments/assets/fa77976c-d2ea-4591-913d-29d83b377878)
![image](https://github.com/user-attachments/assets/eb39148f-4fb9-49f0-a4f7-a3ab4c542bb9)
![image](https://github.com/user-attachments/assets/f1cc4d82-322c-4285-8672-cb9c8b97bafc)
![image](https://github.com/user-attachments/assets/7a3e70d4-2761-416e-bb54-3f83105f4f67)

Hatted with case

![image](https://github.com/user-attachments/assets/0fc9401b-4ddc-4439-9eda-6940399b4535)
![image](https://github.com/user-attachments/assets/8a05a8ec-bdb2-411c-93d7-a585cf31ebef)

Current Clamps

![image](https://github.com/user-attachments/assets/d9696077-9ca2-4352-84b2-c7fc12cf8653)

Wire wrapping

![image](https://github.com/user-attachments/assets/52ef5f90-4650-41e3-888f-5c3f574f5281)

Energy Hub Mounted

![image](https://github.com/user-attachments/assets/2e056a7c-7717-42e3-8fcb-a865b344f2b2)

Energy Hubs Wired

![image](https://github.com/user-attachments/assets/9927fd13-7e83-4219-b573-738bdd0aed28)
![image](https://github.com/user-attachments/assets/9c3208af-153f-46d1-86dd-91b6b2011f20)
![image](https://github.com/user-attachments/assets/f8bd63e2-67a7-4e4b-b667-c9ae413a1b41)
![image](https://github.com/user-attachments/assets/9dbabed8-d2c2-4254-8453-027485c1069b)



