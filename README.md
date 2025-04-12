# LanternPowerMonitor
This is a setup guide of the LanternPowerMonitor self hosted service. (https://github.com/MarkBryanMilligan/LanternPowerMonitor)

## Starting off
I used debian 11 and virtualbox because it is just what I had avialable while I was trying to figure out docker.
1. Download and install VirtualBox (https://www.virtualbox.org)
2. Download Debian 11 (https://cdimage.debian.org/mirror/cdimage/archive/11.11.0/amd64/iso-dvd/debian-11.11.0-amd64-DVD-1.iso)
3. Install VirtualBox and create Debian VM. Add shared folder on host machine to VM location /mnt/host/
4. Once Debian 11 is installed, open a terminal

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

**MOVE BACK TO TERMINAL WINDOW 1**
This actively compiles the source code into usable binaries.
```
sudo ./gradlew clean build publishToMavenLocal
```

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

**Diverging**
This is where everyone will go separate ways. You can have the Tomcat/MongoDB hosted on a raspiberry pi, within a server's docking container, or in some other hosting fashion. The rest of this guide was on my setup. I already had a home server running Unraid so I chose to run Tomcat/MongoDB within my docker environment.

I started by creating a MongoDB container using the following options.
![image](https://github.com/user-attachments/assets/ef42e267-f198-4489-b259-2d2b91ec4e33)

I chose to have docker create an initial database/User/Password because I was also setting up a MongoDB_Express container to monitor/test everything. Also you might be able to use mongo:latest for the repository but I had to downgrade and lock mine to 4.4.18 because my server's processor does not support AVP. Once it was done installing I went into the mongo container and ran the following commands to setup the admin user for tomcat.
```
mongosh -u root -p example
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

