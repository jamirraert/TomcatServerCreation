For security purposes, Tomcat should run under a separate, unprivileged user. Run the following command to create a user called ``tomcat``
```
 sudo useradd -m -d /opt/tomcat -U -s /bin/false tomcat
```

By supplying ``/bin/false`` as the user's default shell, you ensure that it’s not possible to log in as ``tomcat``

Check if there is already JDK downloaded in the server
```
java -version
```

if none:
You’ll now install the JDK. First, update the package manager cache by running:
```
 sudo apt update
```

Then, install the JDK by running the following command:
```
$ sudo apt install openjdk-17-jdk
```

Check java version again:
```
java -version
```

The output should be similar to this:

```
openjdk version "17.0.9" 2023-10-17
OpenJDK Runtime Environment (build 17.0.9+9-Ubuntu-120.04)
OpenJDK 64-Bit Server VM (build 17.0.9+9-Ubuntu-120.04, mixed mode, sharing)
```

To install Tomcat, you’ll need the latest Core Linux build for Tomcat 10, which you can get from the downloads page. Select the latest Core Linux build, ending in `.tar.gz` At the time of writing, the latest version was 10.0.20. 

First, navigate to the /tmp directory:
```
 cd /tmp
```

Download the archive using wget by running the following command: ecg
```
wget https://dlcdn.apache.org/tomcat/tomcat-10/v10.0.20/bin/apache-tomcat-10.0.20.tar.gz
```
The wget command downloads resources from the Internet.

Then, extract the archive you downloaded by running:
```
sudo tar xzvf apache-tomcat-10*tar.gz -C /opt/tomcat --strip-components=1
```

Since you have already created a user, you can now grant tomcat ownership over the extracted installation by running:
```
sudo chown -R tomcat:tomcat /opt/tomcat/
sudo chmod -R u+x /opt/tomcat/bin
```

Both commands update the settings of your tomcat installation.

In this step, you installed the JDK and Tomcat. You also created a separate user for it and set up permissions over Tomcat binaries. You will now configure credentials for accessing your Tomcat instance.

** Step 2 — Configuring Admin Users **

To gain access to the Manager and Host Manager pages, you’ll define privileged users in Tomcat’s configuration. You will need to remove the IP address restrictions, which disallows all external IP addresses from accessing those pages.

Tomcat users are defined in `/opt/tomcat/conf/tomcat-users.xml`. Open the file for editing with the following command:
```
sudo nano /opt/tomcat/conf/tomcat-users.xml
```

Add the following lines before the ending tag:
```
<role rolename="manager-gui" />
<user username="manager" password="manager_password" roles="manager-gui" />

<role rolename="admin-gui" />
<user username="admin" password="admin_password" roles="manager-gui,admin-gui" />
```

Replace highlighted passwords with your own. When you’re done, save and close the file.

Here you define two user roles, `manager-gui` and `admin-gui`, which allow access to Manager and Host Manager pages, respectively. You also define two users, `manager` and `admin`, with relevant roles.

By default, Tomcat is configured to restrict access to the admin pages, unless the connection comes from the server itself. To access those pages with the users you just defined, you will need to edit config files for those pages.

To remove the restriction for the Manager page, open its config file for editing:
```
sudo nano /opt/tomcat/webapps/manager/META-INF/context.xml
```

Comment out the `Valve` definition, as shown:

```
...
<Context antiResourceLocking="false" privileged="true" >
  <CookieProcessor className="org.apache.tomcat.util.http.Rfc6265CookieProcessor"
                   sameSiteCookies="strict" />
<!--  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" /> -->
  <Manager sessionAttributeValueClassNameFilter="java\.lang\.(?:Boolean|Integer|Long|Number|String)|org\.apache\.catalina\.filters\.Csr>
</Context>
```

Save and close the file, then repeat for ** Host Manager **
```
sudo nano /opt/tomcat/webapps/host-manager/META-INF/context.xml
```
You have now defined two users,`manager` and `admin`, which you will later use to access restricted parts of the management interface. You’ll now create a `systemd` service for Tomcat.

** Step 3 — Creating a systemd service **

The `systemd` service that you will now create will keep Tomcat quietly running in the background.
The `systemd` service will also restart Tomcat automatically in case of an error or failure.

Tomcat, being a Java application itself, requires the Java runtime to be present, which you installed with the JDK in step 1. Before you create the service, you need to know where Java is located. You can look that up by running the following command:

```
sudo update-java-alternatives -l
```

The output will be similar to this:
```
java-1.17.0-openjdk-amd64      1711       /usr/lib/jvm/java-1.17.0-openjdk-amd64
```

Note the path where Java resides, listed in the last column. You’ll need the path momentarily to define the service.

You’ll store the `tomcat` service in a file named `tomcat.service`, under `/etc/systemd/system`. Create the file for editing by running:
```
sudo nano /etc/systemd/system/tomcat.service
```
Add the following lines:
```
[Unit]
Description=Tomcat
After=network.target

[Service]
Type=forking

User=tomcat
Group=tomcat

Environment="JAVA_HOME=/usr/lib/jvm/java-1.11.0-openjdk-amd64"
Environment="JAVA_OPTS=-Djava.security.egd=file:///dev/urandom"
Environment="CATALINA_BASE=/opt/tomcat"
Environment="CATALINA_HOME=/opt/tomcat"
Environment="CATALINA_PID=/opt/tomcat/temp/tomcat.pid"
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh

RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
```

Modify the highlighted value of `JAVE_HOME` if it differs from the one you noted previously.

Here, you define a service that will run Tomcat by executing the startup and shutdown scripts it provides. You also set a few environment variables to define its home directory (which is `/opt/tomcat` as before) and limit the amount of memory that the Java VM can allocate (in `CATALINA_OPTS`). Upon failure, the Tomcat service will restart automatically.

When you’re done, save and close the file.

Reload the `systemd` daemon so that it becomes aware of the new service:
```
sudo systemctl daemon-reload
```
You can then start the Tomcat service by typing:
```
sudo systemctl start tomcat
```
Then, look at its status to confirm that it started successfully:
```
sudo systemctl status tomcat
```
The output will look like this:
```
tomcat.service - Tomcat
     Loaded: loaded (/etc/systemd/system/tomcat.service; disabled; vendor preset: enabled)
     Active: active (running) since Fri 2022-03-11 14:37:10 UTC; 2s ago
    Process: 4845 ExecStart=/opt/tomcat/bin/startup.sh (code=exited, status=0/SUCCESS)
   Main PID: 4860 (java)
      Tasks: 15 (limit: 1132)
     Memory: 90.1M
     CGroup: /system.slice/tomcat.service
             └─4860 /usr/lib/jvm/java-1.11.0-openjdk-amd64/bin/java -Djava.util.logging.config.file=/opt/tomcat/conf/logging.properties ...
```

Press `q` to exit the command.

To enable Tomcat starting up with the system, run the following command:
```
sudo systemctl enable tomcat
```