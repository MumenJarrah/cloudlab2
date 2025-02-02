# Lab 5: Public-Key Infrastructure (PKI) Lab

```
Copyright © 2018 by Wenliang Du.
This work is licensed under a Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International
License. If you remix, transform, or build upon the material, this copyright notice must be left intact, or
reproduced in a way that is reasonable to the medium in which the work is being re-published.
```
## 1 Overview

Public key cryptography is the foundation of today’s secure communication, but it is subject to man-in-the-
middle attacks when one side of communication sends its public key to the other side. The fundamental
problem is that there is no easy way to verify the ownership of a public key, i.e., given a public key and its
claimed owner information, how do we ensure that the public key is indeed owned by the claimed owner?
The Public Key Infrastructure (PKI) is a practical solution to this problem.
The learning objective of this lab is for students to gain the first-hand experience on PKI. By doing the
tasks in this lab, students should be able to gain a better understanding of how PKI works, how PKI is used
to protect the Web, and how Man-in-the-middle attacks can be defeated by PKI. Moreover, students will be
able to understand the root of the trust in the public-key infrastructure, and what problems will arise if the
root trust is broken. This lab covers the following topics:

- Public-key encryption, Public-Key Infrastructure (PKI)
- Certificate Authority (CA), X.509 certificate, and root CA
- Apache, HTTP, and HTTPS
- Man-in-the-middle attacks


## 2 Lab Environment

### Step 1: Download and Extract the Lab Files
Download Labsetup.zip from the lab’s website and unzip it. This will provide you with the necessary files, including the docker-compose.yml for setting up the lab environment.

```
# Download the lab setup files
$ sudo wget https://seedsecuritylabs.org/Labs_20.04/Files/Crypto_PKI/Labsetup.zip

# Unzip the lab setup files
$ sudo unzip Labsetup.zip
```

### Step 2: Navigate to the Lab Setup Directory
Move into the extracted Labsetup folder, where you will find the docker-compose.yml file and other necessary files.

```
# Enter the Labsetup folder
$ cd Labsetup
```

### Step 3: Build the Docker Container
Use Docker Compose to build the container image. This step prepares the environment for running your web server with the required configurations.

```
# Build the Docker container
$ docker-compose build

# OR use the alias
$ dcbuild
```

### Step 4: Start the Docker Container
This command initializes and runs the container based on the configurations specified in the docker-compose.yml file.

```
# Start the Docker container
$ docker-compose up

# OR use the alias
$ dcup
```

### Step 5: Stop and Shut Down the Docker Container
When you’re finished or need to reset the environment, shut down the running container to release resources.

```
# Stop and shut down the Docker container
$ docker-compose down

# OR use the alias
$ dcdown
```
 
All the containers will be running in the background. To run commands on a container, we often need
to get a shell on that container. We first need to use the `"docker ps"` command to find out the ID of
the container, and then use `"docker exec"` to start a shell on that container. We have created aliases for
them in the `.bashrc` file.

#### 1. List Running Docker Containers
Use the alias dockps to view a list of running containers, displaying each container's ID and name in a simplified format.

```
$ dockps
```

#### Output
The output will list all running Docker containers, each with its unique ID and assigned name. An example output might look like this:

```
b1004832e275 hostA-10.9.0.5
0af4ea7a3e2e hostB-10.9.0.6
9652715c8e0a hostC-10.9.0.7
```
Each line includes the container ID and its corresponding name (in this case, hostA, hostB, and hostC), along with their assigned IP addresses.

#### 2. Access a Specific Container’s Shell
To open a shell inside a specific container, use the alias docksh followed by the first few characters of the container's ID. For example, to access hostC-10.9.0.7, use the ID prefix 96 (from the third line in the previous output).

```
$ docksh 96
```

#### Output
You will be logged into a shell session inside the specified container. The prompt will change to show the container ID, indicating that you are now inside the container:

```
root@9652715c8e0a:/#
```

#### Note: If a Docker command requires the container ID, you only need to type the first few characters, as long as they are unique among all running containers.


### DNS setup.
In this document, we will use bank32.com as the server name for the HTTPS web server throughout the lab.
The bank32.com server name will be recognized within the container setup as specified in the docker-compose.yml and related configuration files.

```
10.9.0.80 [http://www.bank32.com](http://www.bank32.com)
```

## 3 Lab Tasks

### 3.1 Task 1: Becoming a Certificate Authority (CA)

A Certificate Authority (CA) is a trusted entity that issues digital certificates. The digital certificate certifies the ownership of a public key by the named subject of the certificate. A number of commercial CAs
are treated as root CAs; VeriSign is the largest CA at the time of writing. Users who want to get digital
certificates issued by the commercial CAs need to pay those CAs.
In this lab, we need to create digital certificates, but we are not going to pay any commercial CA. We
will become a root CA ourselves, and then use this CA to issue certificate for others (e.g. servers). In this
task, we will make ourselves a root CA, and generate a certificate for this CA. Unlike other certificates,
which are usually signed by another CA, the root CA’s certificates are self-signed. Root CA’s certificates are
usually pre-loaded into most operating systems, web browsers, and other software that rely on PKI. Root
CA’s certificates are unconditionally trusted.

The Configuration File `openssl.conf`. In order to use `OpenSSL` to create certificates, you have to
have a configuration file. The configuration file usually has an extension .cnf. It is used by three `OpenSSL`
commands: `ca,req` and `x509`. The manual page of openssl.conf can be found from online resources.
By default, OpenSSL use the configuration file from `/usr/lib/ssl/openssl.cnf`. You can access the `openssl.cnf` file as shown in the figure below. Since we need
to make changes to this file, we will copy it into our current directory, and instruct `OpenSSL` to use this
copy instead.

![](images/lab5-00-u.png)

- Copy the default OpenSSL configuration file using the following command:
```
cp /usr/lib/ssl/openssl.cnf ./myCA_openssl.cnf
```

The `[CAdefault]` section of the configuration file shows the default setting that we need to prepare.
We need to create several sub-directories. Please uncomment the `uniquesubject` line in `myCA_openssl.cnf` file (by removing the sign (#) in the CA_default settings) to allow creation of certifications with the same subject, because it is very likely that we will do that in the lab.

Listing 1: Default CA setting

```
[ CA_default ]
dir = ./demoCA # Where everything is kept
certs = $dir/certs # Where the issued certs are kept
crl_dir = $dir/crl # Where the issued crl are kept
database = $dir/index.txt # database index file.
# unique_subject = no # Set to ’no’ to allow creation of
# several certs with same subject.
new_certs_dir = $dir/newcerts # default place for new certs.
serial = $dir/serial # The current serial number
```

For the `index.txt file`, simply create an empty file. For the `serial` file, put a single number in
string format (e.g. 1000) in the file. Once you have set up the configuration file `openssl.cnf`, you can
create and issue certificates.

**Certificate Authority (CA).** As we described before, we need to generate a self-signed certificate for our
CA. This means that this CA is totally trusted, and its certificate will serve as the root certificate. You can
run the following command to generate the self-signed certificate for the CA:

```
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 \
-keyout ca.key -out ca.crt
```

You will be prompted for a password. Do not lose this password, because you will have to type the
passphrase each time you want to use this CA to sign certificates for others. You will also be asked to fill in
the subject information, such as the Country Name, Common Name, etc. The output of the command are stored in two files: `ca.key` and `ca.crt`. The file `ca.key` contains the CA’s private key, while `ca.crt`
contains the public-key certificate.
You can also specify the subject information and password in the command line, so you will not be
prompted for any additional information. 

![](images/lab5-11-u.png)

In the following command, we use  `-subj` to set the subject
information and we use `-passout pass:dees` to set the password to `dees`.

```
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 \
-keyout ca.key -out ca.crt \
-subj "/CN=www.modelCA.com/O=Model CA LTD./C=US" \
-passout pass:dees
```

We can use the following commands to look at the decoded content of the X509 certificate and the
RSA key (`-text` means decoding the content into plain text; `-noout` means not printing out the encoded
version):

```
openssl x509 -in ca.crt -text -noout
openssl rsa -in ca.key -text -noout
```

Please run the above commands. From the output, please identify the followings:

- What part of the certificate indicates this is a CA’s certificate?
  By runnung this command `openssl x509 -in ca.crt -text -noout`, we can examine the certificate content. 

  - The following figure indicates that the certificate is issued for a Certificate Authority (CA) `CA:TRUE`. By locating the Basic Constraints section in the certificate output, it should contain:
  `X509v3 Basic Constraints:
    CA:TRUE`

  ![](images/lab5-22-u.png)
  
- What part of the certificate indicates this is a self-signed certificate?
  - This figure indicates that the certificate is self-signed. By looking at the Issuer and Subject fields. If they are identical, it means the certificate is self-signed.

  ![](images/lab5-33-u.png)
  
- In the RSA algorithm, we have a public exponent (e), a private exponent (d), a modulus (n), and two secret
    numbers (p) and (q) , such that n = p*q. Please identify the values for these elements in your certificate
    and key files.
  By running this command `openssl rsa -in ca.key -text -noout`, we can examine the private key content and look at the following elements:
  
    - Public Exponent (e): This is usually a small value like `65537`. Look for the line starting with `publicExponent` in        the key output
      ![](images/lab5-pub.png)
      
    - Private Exponent (d): This is the value of the private key's exponent, found in the line labeled `privateExponent`.
      ![](images/lab5-priv.png)
      
    - Modulus (n): This is the large number calculated as `n = p * q`, found in the line labeled modulus.
      ![](images/lab5-mod.png)
      
    - Prime Numbers (p and q): These are the two large prime numbers used to calculate `n`. Look for the lines labeled prime1       and prime2.
      ![](images/lab5-pre.png)
      

### 3.2 Task 2: Generating a Certificate Request for Your Web Server

A company called `bank32.com` wants to get a public-
key certificate from our CA. First it needs to generate a Certificate Signing Request (CSR), which basically
includes the company’s public key and identity information. The CSR will be sent to the CA, who will
verify the identity information in the request, and then generate a certificate.
The command to generate a CSR is quite similar to the one we used in creating the self-signed cer-
tificate for the CA. The only difference is the `-x509` option. Without it, the command generates a re-
quest; with it, the command generates a self-signed certificate. The following command generate a CSR for `www.bank32.com` you should use your own server name):

```
openssl req -newkey rsa:2048 -sha256 \
-keyout server.key -out server.csr \
-subj "/CN=www.bank32.com/O=Bank32 Inc./C=US" \
-passout pass:dees
```

The command will generate a pair of public/private key, and then create a certificate signing request
from the public key. We can use the following command to look at the decoded content of the CSR and
private key files:

```
openssl req -in server.csr -text -noout
openssl rsa -in server.key -text -noout
```

Adding Alternative names. Many websites have different URLs. For example, `www.example.com`, `example.com`, `example.net`, and `example.org` are all pointing to the same web server. Due to the hostname matching policy enforced by browsers, the common name in a certificate must match with the
server’s hostname, or browsers will refuse to communicate with the server.
To allow a certificate to have multiple names, the X.509 specification defines extensions to be attached
to a certificate. This extension is called Subject Alternative Name (SAN). Using the SAN extension, it’s
possible to specify several hostnames in the `subjectAltName` field of a certificate.
To generate a certificate signing request with such a field, we can put all the necessary information
in a configuration file or at the command line. We will use the command-line approach in this task. We can add the following option to
the `"openssl req"` command. It should be noted that the `subjectAltName` extension field must also
include the one from the common name field; otherwise, the common name will not be accepted as a valid
name.

```
-addext "subjectAltName = DNS:www.bank32.com, \
DNS:www.bank32A.com, \
DNS:www.bank32B.com"
```

Please add two alternative names to your certificate signing request. They will be needed in the tasks
later.

### 3.3 Task 3: Generating a Certificate for your server

The CSR file needs to have the CA’s signature to form a certificate. In the real world, the CSR files are
usually sent to a trusted CA for their signature. In this lab, we will use our own trusted CA to generate
certificates. The following command turns the certificate signing request `(server.csr)` into an X509 certificate `(server.crt)`, using the CA’s `ca.crt` and `ca.key`:

```
openssl ca -config myCA_openssl.cnf -policy policy_anything \
-md sha256 -days 3650 \
-in server.csr -out server.crt -batch \
-cert ca.crt -keyfile ca.key
```

In the above command, `myCAopenssl.cnf` is the configuration file we copied from `/usr/lib/ssl/openssl.cnf` (we also made changes to this file in Task 1). We use the `policy_anything` policy defined in the configuration file. This is not the default policy; the default policy has more restriction,
requiring some of the subject information in the request to match those in the CA’s certificate. The policy
used in the command, as indicated by its name, does not enforce any matching rule.

Copy the extension field. For security reasons, the default setting in `openssl.cnf` does not allow the `"openssl ca"` command to copy the extension field from the request to the final certificate. To enable
that, we can go to our copy of the configuration file, uncomment the following line:

```
# Extension copying option: use with caution.
copy_extensions = copy
```

After signing the certificate, please use the following command to print out the decoded content of the
certificate, and check whether the alternative names are included.

```
openssl x509 -in server.crt -text -noout
```

### 3.4 Task 4: Deploying Certificate in an Apache-Based HTTPS Website

In this task, we will see how public-key certificates are used by websites to secure web browsing. We will
set up an HTTPS website based Apache. The Apache server, which is already installed in our container,
supports the HTTPS protocol. To create an HTTPS website, we just need to configure the Apache server,
so it knows where to get the private key and certificates. Inside our container, we have already set up an
HTTPS site for `bank32.com`. Students can follow this example to set up their own HTTPS site.
An Apache server can simultaneously host multiple websites. It needs to know the directory where
a website’s files are stored. This is done via its `VirtualHost` file, located in the `/etc/apache2/sites-available` directory. In our container, we have a file called `bank32apachessl.conf`,
which contains the following entry:

```
<VirtualHost *:443>
DocumentRoot /var/www/bank
ServerName [http://www.bank32.com](http://www.bank32.com)
ServerAlias [http://www.bank32A.com](http://www.bank32A.com)
ServerAlias [http://www.bank32B.com](http://www.bank32B.com)
DirectoryIndex index.html
SSLEngine On
SSLCertificateFile /certs/bank32.crt ¿
SSLCertificateKeyFile /certs/bank32.key ¡
</VirtualHost>
```

The above example sets up the HTTPS site `https://www.bank32.com`(port `443` is the default
HTTPS port). The `ServerName` entry specifies the name of the website, while the `DocumentRoot`
entry specifies where the files for the website are stored. Using the `ServerAlias` entries, we allow the
website to have different names. You should also provide two alias entries.
We also need to tell Apache where the server certificate (Line1) and private key (Line2) are stored. In
the `Dockerfile`, we have already included the commands to copy the certificate and key to the `/certs` folder of the container.
In order to make the website work, we need to enable Apache’s `ssl` module and then enable this site.
They can be done using the following commands, which are already executed when the container is built.

```
$ a2enmod ssl // Enable the SSL module
$ a2ensite bank32_apache_ssl // Enable the sites described in this file
```

Starting the Apache server. The Apache server is not automatically started in the container, because of
the need to type the password to unlock the private key. Let’s go to the container and run the following
command to start the server (we also list some related commands):

```
// Start the server
$ service apache2 start

// Stop the server
$ service apache2 stop

// Restart a server
$ service apache2 restart
```

When Apache starts, it needs to load the private key for each HTTPS site. Our private key is encrypted,
so Apache will ask us to type the password for decryption. Inside the container, the password used for `bank32` is `dees`. Once everything is set up properly, we can browse the web site, and all the traffic
between the browser and the server will be encrypted.

Shared folder between the VM and container. In this task, we need to copy files from the VM to the
container. To avoid repeatedly recreating containers, we have created a shared folder between the VM and
container. When you use the Compose file inside the `Labsetup` folder to create containers, the `volumes`
sub-folder will be mounted to the container. Anything you put inside this folder will be accessible from
inside of the running container.

### Browsing the website. 
Now, point the browser to your web server (note: you should put `https` at the
beginning of your URL, instead of using `http`). Most likely, you will see a warning or error message as shown in the figure below. This is because the browser does not recognize your self-signed certificate as a trusted certificate because it is not issued by a well-known Certificate Authority (CA). Browsers only trust certificates from established, pre-configured CAs.

  ![](images/lab5-88-u.png)

To resolve the issue, you need to manually add your self-signed certificate as a trusted authority in the browser. You have to ensure that you have the `ca.crt` file generated earlier. Then, in Firefox, type `about:preferences#privacy` in the address bar and scroll down to the `Certificates` section and click `View Certificates` as shown in the figure below.
```
about:preferences#privacy
```
 ![](images/lab5-77-u.png)

In the `Authorities` tab, you will see a list of certificates that are already accepted by Firefox. From here, we can import our own certificates. 

 ![](images/lab5-99-u.png)

After choosing the certificate file, please select the following option: “Trust this CA to identify web sites”.

  ![](images/lab5-10-u.png)

You will see that our certificate is now in Firefox’s list of accepted certificates.

### 3.5 Task 5: Launching a Man-In-The-Middle Attack

In this task, we will show how PKI can defeat Man-In-The-Middle (MITM) attacks. Figure 1 depicts how
MITM attacks work. Assume Alice wants to visit `example.com` via the HTTPS protocol. She needs to get
the public key from the `example.com` server; Alice will generate a secret, and encrypt the secret using the
server’s public key, and send it to the server. If an attacker can intercept the communication between Alice
and the server, the attacker can replace the server’s public key with its own public key. Therefore, Alice’s
secret is actually encrypted with the attacker’s public key, so the attacker will be able to read the secret. The
attacker can forward the secret to the server using the server’s public key. The secret is used to encrypt the
communication between Alice and server, so the attacker can decrypt the encrypted communication.
The goal of this task is to help students understand how PKI can defeat such MITM attacks. In the task,
we will emulate an MITM attack, and see how exactly PKI can defeat it. We will select a target website
first. In this document, we use `www.example.com` as the target website, but in the task, to make it more
meaningful, students should pick a popular website, such as a banking site and social network site.

#### Step 1: Setting up the malicious website. 
In Task 4, we have already set up an HTTPS website. We
will use the same Apache server to impersonate `www.example.com`.
To achieve that, we will follow the instruction in Task 4 to add a `VirtualHost` entry to Apache’s SSL
configuration file: the `ServerName` should be `www.example.com`, but the rest of the configuration can

![](images/i2.png)

Figure 1: A Man-In-The-Middle (MITM) attack

be the same as that used in Task 4. Obviously, in the real world, you won’t be able to get a valid certificate
for `www.example.com`, so we will use the same certificate that we used for our own server.
Our goal is the following: when a user tries to visit `www.example.com`, we are going to get the user
to land in our server, which hosts a fake website for `www.example.com`. If this were a social network
website, The fake site can display a login page similar to the one in the target website. If users cannot
tell the difference, they may type their account credentials in the fake webpage, essentially disclosing the
credentials to the attacker.

#### Step 2: Becoming the man in the middle.
There are several ways to get the user’s HTTPS request to land
in our web server. One way is to attack the routing, so the user’s HTTPS request is routed to our web server.
Another way is to attack DNS, so when the victim’s machine tries to find out the IP address of the target web
server, it gets the IP address of our web server. In this task, we simulate the attack-DNS approach. Instead of
launching an actual DNS cache poisoning attack, we simply modify the victim’s machine’s `/etc/hosts`
file to emulate the result of a DNS cache positing attack by mapping the hostname `www.example.com` to
our malicious web server.

```
10.9.0.80 [http://www.example.com](http://www.example.com)
```

#### Step 3: Browse the target website. 
With everything set up, now visit the target real website, and see what
your browser would say. As shown in the screen shots below, the browser will display a warning.

Figure 2 shows the Apache2 Ubuntu Default Page displayed when accessing http://www.example.com. This indicates that the browser request is being directed to our server, confirming the DNS redirection is working.

![](images/lab5_2.png)

Figure 2: Apache2 Default Page showing successful DNS redirection to the attacker’s server for http://www.example.com.

Figure 3 shows the browser displaying a "Warning: Potential Security Risk Ahead" message when accessing https://www.example.com. The warning occurs because the certificate presented by our server does not match the www.example.com, that prevent MITM attacks.

![](images/lab5_3.png)

Figure 3: Security warning in the browser when accessing https://www.example.com

### You have successfully completed the lab
