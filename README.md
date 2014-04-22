# Red Hat/ARM IoT App on OpenShift
![Red Hat](https://raw.githubusercontent.com/starksm64/DevNation2014/master/images/redhat-logo.png) + ![](https://raw.githubusercontent.com/starksm64/DevNation2014/master/images/arm_mbed.jpg)

Summary: A collection of projects that interfaces with the
[Sensinode Developer](https://silver.arm.com/browse/SEN00) NanoService 1.11 Developer
Release to expose the sensors of [mbed LPC1768](https://mbed.org/platforms/mbed-LPC1768/) +
[mbed Application Board](https://mbed.org/components/mbed-Application-Board/) combination as POJOs in the JBoss Wildfly server running on OpenShift.

# Introduction
---------------------

This project includes EE7 component projects that illustrate connecting to the ARM NSP server to expose the sensor information as POJOs.

* iotbof-ear : The EE7 EAR containing the project EJB, WAR
* iotbof-ejb : An EE7 singleton EJB that connects to the NSP
* iotbof-rest : A wrapper api that communicates with the NSP endpoints using REST
* iotbof-test : Tests for the project elements
* iotbof-web : A web application that receives push notifications of sensor information
as well as simple web pages for viewing the sensor data.

## Requirements
You will need an [OpenShift Online](https://www.openshift.com/app/account/new) account and [Install the OpenShift RHC Client Tools](https://www.openshift.com/developers/rhc-client-tools-install).


# Create the Wildfly Application
-------------------
Login to your OpenShift Online account and access the [Applications](https://openshift.redhat.com/app/console/applications) console to create a new Wildfly application that uses the [DevNation2014](https://github.com/starksm64/DevNation2014) git project.

1. Click the Wildfly 8 button to create start the deployment process![](images/WildflyBtnOS.png)
2. Enter your Public URL to use for the application. Here I'm using myiot-jbossdev.rhcloud.com ![](images/WildflyConfigureOS.png)
3. Set the Source Code to the https://github.com/starksm64/DevNation2014OS Git repository.
4. Click ![](images/CreateBtnOS.png) to finish the application create process
5. The resulting Next steps page will show you the locations of the application git repository to clone![](images/NextStepsOS.png).
    * Copy the git clone command line shown and run that on your local computer to pull the application repository down from OpenShift.
6. Run `git clone https://github.com/starksm64/DevNation2014` to pull the application source onto your computer.
7. cd into the local git repository
8. git pull -s recursive -X theirs https://github.com/starksm64/DevNation2014OS.git
11. git remote add DevNation2014 ../DevNation2014
12. git merge -s ours --no-commit DevNation2014/master
13. git read-tree --prefix=DevNation2014 -u DevNation2014/master
12. rm -rf DevNation2014/Presentation

## Wildfly Configuration
There are a few @Resource injections from JNDI that need to be configured in the wildfly server configuration to specify the correct locations for the NSP server, it's domain, and the location of the iotbof-web project NspNotificationService. The configuration file that needs to be edited is iotbof/scripts/pom.xml.

Pull the .openshift/config/standalone.xml file into an editor, and edit the subsystem xmlns="urn:jboss:domain:naming:2.0" section to be as shown here:

        <subsystem xmlns="urn:jboss:domain:naming:2.0">
            <bindings>
                <simple name="java:global/NSPDomain" value="domain" type="java.lang.String"/>
                <simple name="java:global/NSPUsername" value="admin" type="java.lang.String"/>
                <simple name="java:global/NSPPassword" value="secret" type="java.lang.String"/>
                <simple name="java:global/NSPURL" value="http://red-hat-summit.cloudapp.net:8080/" type="java.lang.String"/>
                <simple name="java:global/NotificationCallbackURL" value="http://${env.OPENSHIFT_GEAR_DNS}:${env.OPENSHIFT_WILDFLY_HTTP_PORT}/iotbof-web/rest/events/send" type="java.net.URL"/>
            </bindings>
            <remote-naming/>
        </subsystem>

* The NSPDomain binding provides the domain name on the NSP server for the sensors.
* The NSPURL provides the base URL for the NSP REST interface
* The NSPUsername is the username used to login to the NSP server. You will be given one for your group.
* The NSPPassword is the password used to login to the NSP server. You will be given one for your group.
* The wildfly.host is the public NIC IP address your Wildfly server is bound to.

## Building the Project
To build the project and bring up the NSPViewer application running under Wildfly, perform the following steps:

1. wget http://download.jboss.org/wildfly/8.0.0.Final/wildfly-8.0.0.Final.zip
    1. `unzip wildfly-8.0.0.Final.zip`
    2. Note the path to the wildfly-8.0.0.Final directory as it will be used a wildfly.home in configurations
2. `git clone https://github.com/starksm64/DevNation2014.git` to create the DevNation2014 repository
3. `cd DevNation2014`
4. Set JAVA_HOME if needed
5. edit the project root pom.xml and
	1. set the wildfly.home property to the directory path noted above
	2. cd into the scripts directory and run `mvn wildfly:start wildfly:execute-commands`
6. cd back to DevNation2014 and build the project by running `mvn install`
7. `cd iotbof-ear`
8. Run the application by running `mvn wildfly:run`
9. Now you should be able to open http://${wildfly.host}/iotbof-web/index.xhtml in your brower to view the NSPViewer application.



# Trouble Shooting

## Compilation errors
If you see a compilation error like the following while building the project using step 6 above:

	[INFO] -------------------------------------------------------------
	[ERROR] COMPILATION ERROR : 
	[INFO] -------------------------------------------------------------
	[ERROR] Failure executing javac, but could not parse the error:
	javac: invalid target release: 1.7
	Usage: javac <options> <source files>

then the java being picked up by default is Java 6 or lower. Explicitly export a JAVA_HOME that points to a JDK install of Java 7.

## Duplicate resource error

If the server fails to startup in step 8, and you see an error like the following:

[ERROR] Failed to execute goal org.wildfly.plugins:wildfly-maven-plugin:1.0.1.Final:run (default-cli) on project iot-ear: The server failed to start: Deployment failed and was rolled back. "JBAS014803: Duplicate resource [(\"deployment\" => \"iot-ear.ear\")]" -> [Help 1]

Edit your ${wildfly.home}/standalone/configuration/standalone.xml file and remove the deployments section at the end of the file:

	    </socket-binding-group>
	
	    <deployments>
	        <deployment name="iot-ear.ear" runtime-name="iot-ear.ear">
	            <content sha1="c27977d4ddd3dcd08ac3849ad34f27198380a4d6"/>
	        </deployment>
	    </deployments>
	</server>

The server is getting confused about the iot-ear.ear deployment for some reason that I have not figured out as yet.

## Missing JNDI bindings
Check the JNDI namespace of the server by running
`${WILDFLY_HOME}/bin/jboss-cli.sh -c --command=/subsystem=naming:jndi-view`

Sometimes you need to control the interface or http port your local wildfly instance uses. These can be set by adding a system-properties section to your wildfly-8.0.0.Final/standalone/configuration/standalone.xml file like the following:

    <system-properties>
        <property name="jboss.bind.address" value="127.0.0.1"/>
        <property name="jboss.http.port" value="8088"/>
    </system-properties>

## Notifications could not be enabled
If you see a warning about Notifications could not be enabled, then the NotificationCallbackURL is not accessible by the NSP server. Either the URL has a typo, or the port is not getting through the firewall/router you are behind are the typical problems I have run into.

## USB Terminal

Mac: [CoolTerm](http://freeware.the-meiers.org/CoolTermMac.zip) is a simple serial port terminal application (no terminal emulation) that is geared towards hobbyists and professionals with a need to exchange data with hardware connected to serial ports such as servo controllers, robotic kits, GPS receivers, microcontrollers, etc.

minicom is what to use for linux.
yum install minicom

## SSH tunnel

on laptop
ssh -N -R 8083:localhost:8083 dmz.starkinternational.com
on dmz.starkinternational.com
ssh -N -L 192.168.1.107:8083:localhost:8083 locahost

Not sure why this won't work on laptop:
ssh -R 192.168.1.107:8083:127.0.0.1:8083 -N dmz.starkinternational.com
