---
layout: post
title: How to setup the workstation for Build Tools for VMware Aria
date: "2024-03-24"
media_subpath: /assets/img/vrbt-how-to-setup-workstation/
categories: [VMware, Build Tools, Aria Orchestrator, How To, vRO]
tags: [vmware, building_tools]
---

The following applies on Mac, but should work on Windows as well with proper Windows command accordingly.
More documentation can be found [here](https://github.com/vmware/build-tools-for-vmware-aria).

## Prerequisites

### Install [Homebrew](https://brew.sh)

```shell
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### Install Maven

> For now, only Maven 3.8.x is supported. 3.9.x is working well. But not higher.
{: .prompt-info }

```shell
brew install maven
```

### Install and configure OpenJDK@17

- Install OpenJDK

```shell
brew install openjdk@17
```

- Add `JAVA_HOME` to `.zshrc`

```shell
echo 'export JAVA_HOME=$(/usr/libexec/java_home)' >> ~/.zshrc
```

- Apply changes

```shell
source .zshrc
```

### Download and install [NodeJS](https://nodejs.org/dist/v16.15.1/node-v16.15.1.pkg)

> For now, only NodeJS 14.x is supported. 16.x is working well. But not higher.
{: .prompt-info }

### Create a Keystore for vRO package signing

- Create a directory for Keystore

```shell
mkdir archetype.keystore-2.0.0
cd archetype.keystore-2.0.0
```

- Generate a new keystore. Replace with your values.

> Java keystore used for signing packages build time. All API calls from the toolchain (i.e. the client) verify the SSL certificate returned by vRO/vRA (i.e. the server). If you are using self-signed or third-party signed certificates, you may need to add those certificates or their CA certificates to the default JAVA keystore, i.e. `JAVA_HOME/lib/security/cacerts`. **This is the recommended approach.**
{: .prompt-info }

```shell
keytool -keystore archetype.keystore -genkey -alias dunesrsa_alias -storepass 'XXXXXX' -keyalg RSA
What is your first and last name?
  [Unknown]:  John Doe
What is the name of your organizational unit?
  [Unknown]:  XX
What is the name of your organization?
  [Unknown]:  XX
What is the name of your City or Locality?
  [Unknown]:  XX
What is the name of your State or Province?
  [Unknown]:  XX
What is the two-letter country code for this unit?
  [Unknown]:  XX
Is CN=John Doe, OU=XX, O=XX, L=XX, ST=XX, C=XX correct?
  [no]:  yes

Enter key password for <dunesrsa_alias>
 (RETURN if same as keystore password):

Warning:
The JKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard format using "keytool -importkeystore -srckeystore archetype.keystore -destkeystore archetype.keystore -deststoretype pkcs12".
```

- Generate a key to sign the toolchain. Replace the values with yours.

> It's essential to note that the `emailAddress` should NOT be empty. Otherwise, the vRO import will break with a '400 OK' error
{: .prompt-info }

```shell
keytool -genkey -keyalg RSA -keysize 2048 -alias dunesrsa_alias -keystore archetype.keystore -storepass 'XXXXX' -validity 3650 -dname "CN=Project,OU=Department,O=Company,L=City,ST=State,C=XX,emailAddress=XXX"

Enter key password for <dunesrsa_alias>
 (RETURN if same as keystore password):

Warning:
The JKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard format using "keytool -importkeystore -srckeystore archetype.keystore -destkeystore archetype.keystore -deststoretype pkcs12".
```

### Generate a Private Key and Certificate for vRO Package Signing

```shell
cd /path/archetype.keystore-2.0.0
```

Generate a private key and export it. Replace with your values.

```shell
openssl genpkey -out private_key.pem -algorithm RSA
```

```shell
openssl req -new -key private_key.pem -out csr.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) []:XX
State or Province Name (full name) []:XX
Locality Name (eg, city) []:XX
Organization Name (eg, company) []:XX
Organizational Unit Name (eg, section) []:XX
Common Name (eg, fully qualified host name) []:XX
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
```

```shell
openssl req -x509 -days 999 -key private_key.pem -in csr.csr -out cert.pem
```

> Make sure that `archetype.keystore-2.0.0` directory contains those three files
{: .prompt-info }

![img-description](dfbfdbaa-104c-4c70-9318-ba7c79516013.png){: .shadow }{: width="300" height="200" }{: .normal }

### Create an `archetype.keystore-2.0.0.zip`

```shell
cd ..
zip archetype.keystore-2.0.0.zip -r archetype.keystore-2.0.0
```

### Create `.m2` and `keystore` directory

```shell
mkdir ~/.m2
cd .m2
mkdir keystore
```

### Copy `archetype.keystore-2.0.0.zip` to .m2 directory

```shell
cp /path/archetype.keystore-2.0.0.zip ~/.m2/keystore/
```

Now, the `~/.m2/keystore/` should contain the following:

![img-description](a5d2bf00-7ec6-4eb9-8754-826a1b108ebc.png){: .shadow }{: width="300" height="200" }{: .normal }

### Create settings-security.xml

> All the encrypted passwords will be used later in the `settings.xml`. Maven password encryption details can be found [here](https://maven.apache.org/guides/mini/guide-encryption.html).
{: .prompt-info }

```shell
cd ~/.m2
touch settings-security.xml
```

### Generate Maven master password

```shell
mvn --encrypt-master-password
Master password: XXX
{5bTlAWaH...}
```

Save it into `settings-security.xml`.

```shell
vi settings-security.xml
```

Copy password into the `settings-security.xml` and save.

```xml
<settingsSecurity>
    <master>{5bTlAWaH...}</master>
</settingsSecurity>
```

### Encrypt credentials

Encrypt <u>user's</u> password and <u>keystore's</u> password.

> Using this method, Maven will handle the escape of all special characters. There is no longer a need to provide the password as part of the command.
{: .prompt-tip }
> Example:

```shell
mvn --encrypt-password
Password:
{5bTlAWaH...}
```

### Create `settings.xml`

```shell
vi settings.xml
```

Copy the XML body below into `settings.xml` ans save the file.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings
    xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.1.0 http://maven.apache.org/xsd/settings-1.1.0.xsd"
    xmlns="http://maven.apache.org/SETTINGS/1.1.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <servers>
        <server>
            <username>user@domain.local</username>
            <password>{5bTlAWaH...}</password>
            <id>vra01</id>
        </server>
        <server>
            <username>user@domain.local</username>
            <password>{5bTlAWaH...}</password>
            <id>vro01</id>
        </server>
    </servers>
    <profiles>
        <profile>
            <id>packaging</id>
            <properties>
                <keystorePassword>{ftykftyid}</keystorePassword>
                <keystoreLocation>/Users/user1/.m2/keystore/archetype.keystore-2.0.0.zip</keystoreLocation>
                <vroPrivateKeyPem>/Users/user1/.m2/keystore/private_key.pem</vroPrivateKeyPem>
                <vroCertificatePem>/Users/user1/.m2/keystore/cert.pem</vroCertificatePem>
                <vroKeyPass>PASSWORD</vroKeyPass>
            </properties>
        </profile>
        <profile>
            <id>bundle</id>
            <properties>
                <assembly.skipAssembly>false</assembly.skipAssembly>
            </properties>
        </profile>
        <profile>
            <id>artifactory</id>
            <repositories>
                <repository>
                    <snapshots>
                        <enabled>false</enabled>
                    </snapshots>
                    <id>central</id>
                    <name>central</name>
                    <url>https://repo1.maven.org/maven2/</url>
                </repository>
                <repository>
                    <snapshots>
                        <enabled>true</enabled>
                    </snapshots>
                    <id>central-snapshots</id>
                    <name>central-snapshots</name>
                    <url>https://repo1.maven.org/maven2/</url>
                </repository>
            </repositories>
            <pluginRepositories>
                <pluginRepository>
                    <snapshots>
                        <enabled>false</enabled>
                    </snapshots>
                    <id>central</id>
                    <name>central</name>
                    <url>https://repo1.maven.org/maven2/</url>
                </pluginRepository>
                <pluginRepository>
                    <snapshots>
                        <enabled>true</enabled>
                    </snapshots>
                    <id>central-snapshots</id>
                    <name>central-snapshots</name>
                    <url>https://repo1.maven.org/maven2/</url>
                </pluginRepository>
            </pluginRepositories>
            <properties>
                <releaseRepositoryUrl>https://repo1.maven.org/maven2/</releaseRepositoryUrl>
                <snapshotRepositoryUrl>https://repo1.maven.org/maven2/</snapshotRepositoryUrl>
            </properties>
        </profile>
        <profile>
            <!--Environment
            identifier. Multiple environments are allowed by configuring multiple profiles -->
            <id>vro01</id>
            <properties>
                <vrealize.ssl.ignore.hostname>false</vrealize.ssl.ignore.hostname>
                <vrealize.ssl.ignore.certificate>false</vrealize.ssl.ignore.certificate>
                <!--vRO Connection-->
                <vro.host>{vro_host}</vro.host>
                <vro.port>{vro_port}</vro.port>
                <vro.serverId>vro01</vro.serverId>
                <vro.auth>{basic}</vro.auth> <!-- If "basic" is selected here, ensure com.vmware.o11n.sso.basic-authentication.enabled=true System Property is set in vRO -->
                <vro.authHost>{auth_host}</vro.authHost> <!-- Required for external vRO instances when vra auth is used -->
                <vro.authPort>{auth_port}</vro.authPort> <!-- Required for external vRO instances when vra auth is used -->
                <vro.refresh.token>{refresh_token}</vro.refresh.token> <!-- login with token when vra auth is used -->
                <vro.proxy>http://proxy.host:80</vro.proxy>
                <vro.tenant>{vro_tenant}</vro.tenant>
            </properties>
        </profile>
        <profile>
            <!--Environment
            identifier. Multiple environments are allowed by configuring multiple profiles -->
            <id>vra01</id>
            <properties>
                <vrealize.ssl.ignore.hostname>false</vrealize.ssl.ignore.hostname>
                <vrealize.ssl.ignore.certificate>false</vrealize.ssl.ignore.certificate>
                <!--vRA Connection-->
                <vra.host>{vra_host}</vra.host>
                <vra.port>{vra_port}</vra.port>
                <vra.tenant>{vra_tenant}</vra.tenant>
                <vra.serverId>vra01</vra.serverId>
            </properties>
        </profile>
    </profiles>
    <activeProfiles>
        <activeProfile>artifactory</activeProfile>
        <activeProfile>packaging</activeProfile>
    </activeProfiles>
</settings>
```

> If I encrypt the `vroKeyPass` with `mvn --encrypt-password` the project building will fail in the later stage. I didn't find a way to make it work. The `vroKeyPass` should remain a clear-text password.
{: .prompt-warning }

- `vrealize.ssl.ignore` can be changed from `false` to `true` if needed. Of course, it is not recommended in the production environment
- `vroKeyPass` is a password used when we created a Private Key
- `keystorePassword` is a password used when we created a Keystore
- `keystoreLocation` change to your location
- `vroPrivateKeyPem` change to your location
- `vroCertificatePem` change to your location
- `<servers><server><username>` change to your username
- `<servers><server><password>` change to your password
- `<servers><server><id>` change to your id

The minimum required settings for vRO

```xml
<vro.host>VRO_FQDN</vro.host>
<vro.port>443</vro.port>
<vro.serverId>vro01</vro.serverId>
<vro.auth>basic</vro.auth>
```

The minimum required settings for vRA

```xml
<vro.host>VRA_FQDN</vro.host>
<vro.port>443</vro.port>
<vro.serverId>vra01</vro.serverId>
<vro.auth>basic</vro.auth>
```

Here's how the final setup will look if one (or both) of these minimum settings are applied:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings
    xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.1.0 http://maven.apache.org/xsd/settings-1.1.0.xsd"
    xmlns="http://maven.apache.org/SETTINGS/1.1.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <servers>
        <server>
            <username>user@domain.local</username>
            <password>{5bTlAWaH...}</password>
            <id>vra01</id>
        </server>
        <server>
            <username>user@domain.local</username>
            <password>{5bTlAWaH...}</password>
            <id>vro01</id>
        </server>
    </servers>
    <profiles>
        <profile>
            <id>packaging</id>
            <properties>
                <keystorePassword>{ftykftyid}</keystorePassword>
                <keystoreLocation>/Users/user1/.m2/keystore/archetype.keystore-2.0.0.zip</keystoreLocation>
                <vroPrivateKeyPem>/Users/user1/.m2/keystore/private_key.pem</vroPrivateKeyPem>
                <vroCertificatePem>/Users/user1/.m2/keystore/cert.pem</vroCertificatePem>
                <vroKeyPass>PASSWORD</vroKeyPass>
            </properties>
        </profile>
        <profile>
            <id>bundle</id>
            <properties>
                <assembly.skipAssembly>false</assembly.skipAssembly>
            </properties>
        </profile>
        <profile>
            <id>artifactory</id>
            <repositories>
                <repository>
                    <snapshots>
                        <enabled>false</enabled>
                    </snapshots>
                    <id>central</id>
                    <name>central</name>
                    <url>https://repo1.maven.org/maven2/</url>
                </repository>
                <repository>
                    <snapshots>
                        <enabled>true</enabled>
                    </snapshots>
                    <id>central-snapshots</id>
                    <name>central-snapshots</name>
                    <url>https://repo1.maven.org/maven2/</url>
                </repository>
            </repositories>
            <pluginRepositories>
                <pluginRepository>
                    <snapshots>
                        <enabled>false</enabled>
                    </snapshots>
                    <id>central</id>
                    <name>central</name>
                    <url>https://repo1.maven.org/maven2/</url>
                </pluginRepository>
                <pluginRepository>
                    <snapshots>
                        <enabled>true</enabled>
                    </snapshots>
                    <id>central-snapshots</id>
                    <name>central-snapshots</name>
                    <url>https://repo1.maven.org/maven2/</url>
                </pluginRepository>
            </pluginRepositories>
            <properties>
                <releaseRepositoryUrl>https://repo1.maven.org/maven2/</releaseRepositoryUrl>
                <snapshotRepositoryUrl>https://repo1.maven.org/maven2/</snapshotRepositoryUrl>
            </properties>
        </profile>
        <profile>
            <!--Environment
            identifier. Multiple environments are allowed by configuring multiple profiles -->
            <id>vro01</id>
            <properties>
                <vrealize.ssl.ignore.hostname>false</vrealize.ssl.ignore.hostname>
                <vrealize.ssl.ignore.certificate>false</vrealize.ssl.ignore.certificate>
                <!--vRO Connection-->
                <vro.host>VRO_FQDN</vro.host>
                <vro.port>443</vro.port>
                <vro.serverId>vro01</vro.serverId>
                <vro.auth>basic</vro.auth>
            </properties>
        </profile>
        <profile>
            <!--Environment
            identifier. Multiple environments are allowed by configuring multiple profiles -->
            <id>vra01</id>
            <properties>
                <vrealize.ssl.ignore.hostname>false</vrealize.ssl.ignore.hostname>
                <vrealize.ssl.ignore.certificate>false</vrealize.ssl.ignore.certificate>
                <!--vRA Connection-->
                <vro.host>VRA_FQDN</vro.host>
                <vro.port>443</vro.port>
                <vro.serverId>vra01</vro.serverId>
                <vro.auth>basic</vro.auth>
            </properties>
        </profile>
    </profiles>
    <activeProfiles>
        <activeProfile>artifactory</activeProfile>
        <activeProfile>packaging</activeProfile>
    </activeProfiles>
</settings>
```

Now, the `.m2` directory should include the following:

```shell
Permissions Size   User  Date Modified Name
drwxr-xr-x@    -   user1 27 Mar 20:07  keystore
.rw-r--r--@  105   user1 27 Mar 21:07  settings-security.xml
.rw-r--r--@  6.1k  user1 27 Mar 21:07  settings.xml
```

## Install Building Tools dependencies

Clone the [repo](https://github.com/vmware/build-tools-for-vmware-aria.git) and run the following:

```shell
cd /path/build-tools-for-vmware-aria-2.37.0

mvn clean install -f common/keystore-example/pom.xml
mvn clean install -f maven/npmlib/pom.xml
mvn clean install -f pom.xml
mvn clean install -f maven/base-package/pom.xml
mvn clean install -f packages/pom.xml
mvn clean install -f maven/typescript-project-all/pom.xml
mvn clean install -f maven/repository/pom.xml
```

All the mvn commands should be completed successfully. This will confirm that NodeJS is installed correctly. If one of the steps will fail, check that the installed version of Node is 16.x.

## Install vRealize Developer Tools

Follow the instructions provided in the [repository](https://github.com/vmware/vrealize-developer-tools?tab=readme-ov-file). The simplest way is to install the extension in the VSCode. Just go to the Extensions Marketplace and install it.
![img-description](0a9ffd23-2a86-4314-aa33-897f9eda7409.png){: .shadow }{: .normal }

## Create a new project in VSCode

Open VSCode, go to Command Palette (CMD + SHIFT + P) and start typing `vRealize`. Select `vRealize: New Project`.
![img-description](92d76fbb-7de1-4e4c-a49e-8a12110202c0.png){: .shadow }{: .normal }
Select the type of the project. Let's create a vRO TypeScript-based Project
![img-description](d1a91d8a-0bba-4001-b071-460083ad51a7.png){: .shadow }{: .normal }{: .normal }
Provide a Project ID
![img-description](e67015a4-fef5-43ab-87da-41cc7c6fca49.png){: .shadow }{: .normal }
Provide a Project Name
![img-description](17c843bc-f1b7-497a-bd98-44058877fb4a.png){: .shadow }{: .normal }
Save the project in some directory.

> If the error occurs, it may happen because of the default version `DarchetypeVersion=2.12.5`. The quick solution will be to change the default version in the vRealize Developer Tool setting in VSCode below to any relevant version that should be used.
{: .prompt-tip }
![img-description](Screenshot 2024-03-27 at 15.11.54.png){: .shadow }{: width="400" height="300" }{: .normal }

When everything was done properly, we should see the following in the VSCode
![img-description](ef7e0127-1200-4cc3-85d4-da84417bcd9e.png){: .shadow }{: width="500" height="400" }{: .normal }

### Optional: Configure vRO to support Basic Authentication

If `Basic` authentication is chosen, follow [this](https://docs.vmware.com/en/VMware-Aria-Automation/8.16/Installing-Configuring-Automation-Orchestrator/GUID-30026BCF-DC1F-471E-A63C-A29E85FBDD41.html) procedure. This is how it should look like at the end.
![img-description](1f69c989-5ce3-43ec-bbd2-22e070d78dae.png){: .shadow }{: width="500" height="400" }{: .normal }

## UPDATE 1: Windows installation

Thanks to [Mohammad Makeen AlDamouni](https://www.linkedin.com/in/mohammad-makeen/) for providing these tips.

Windows based installation requires a few additional adjustments:

> Make sure both Python and OpenSSL are added to the environment variables.
{: .prompt-tip }

1. Install OpenSSL (version: OpenSSL 3.1.3 19 Sep 2023 (Library: OpenSSL 3.1.3 19 Sep 2023) )
2. Install Python (version: Python 3.8.0) - pip: 19.2.3

    > The steps below are similar to those mentioned above for the Mac and can be referenced.
    {: .prompt-info }

3. Creation of the Keystore can be done using the [KeyStore Explorer](https://github.com/kaikramer/keystore-explorer).
4. Extract private key from the keystore.
5. Generate the certificate.
6. Confirm that both the generated previously certificate and private key are valid.

> Some of the `mvn clean install` commands can fail. The reason for that is the user who executed the command encountered a privilege restriction, which prevented the command from running successfully. To resolve this issue, one possible solution is to open the Command Prompt with administrative access by choosing the "Run as administrator" option.
{: .prompt-tip }
> It is possible to add `-X` to the `mvn` command to get the output at the `debug` level.
{: .prompt-tip }

## Next step

In the next post, we'll see how to push the code to the vRO with some cool tricks.
