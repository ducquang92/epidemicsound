# Docker security
## Secure the container
The container is secured using the following feature:
- Transport security: force-enable TLS version 1.2 (disable SSL 3.0, TLS 1.0, TLS 1.1) with strong cipher suites.
- Permission: configure apache service running with user and group www-data. The root directory belongs to user www-data.
- Fingerprint: disable server fingerprints.
- HTTP security features: configure the HTTP security header (X-XSS-protection, X-Frame-Options, HttpOnly and Secure flag in cookie).
## Scalability
- The container is configued to run with docker-compose in Docker Swarm environment. This simple application will be ran as a service in Docker Swarm cluster. By default, it is initialized with 2 replicas, but we can scale the service by scaling the docker container (docker service scale SERVICE=REPLICAS).
- In order to run the container in the docker swarm mode, you can use the following commmands:
```
docker service create --name registry --publish published=5000,target=5000 registry # Create a local registry
docker-compose publish # Publish the docker image to the local registry, you can also store it in your private online registry
docker stack deploy --compose-file docker-compose.yml demo # Deploy the new stack in the docker swarm cluster
docker service scale demo_securityengineer=4 # Manually scale the service to 4 instances
```
## Integration with SAST/OSA solution
- I suggest to integrate static code scanning solution (e.g, Checkmarx) into CI pipeline of the project. After all the build steps, there will be one non-blocking step to trigger security scanning of the source code and record the result in Checkmarx dashboard. As the scan usually takes a long time, this step will be not in the waiting state (asynchronous), but the rest of the steps will be continued to execute. After the scan is finished, the result will be notified via email/or another chat channel.
- Besides SAST, it is advisable to integrate OSA scanning and DAST (Dynamic application security testing) into the CI as well.
## Code review guideline
A thorough source code review guideline is available at https://owasp.org/www-pdf-archive/OWASP_Code_Review_Guide_v2.pdf (from page 43). However, the OWASP guide is really long and contains different examples for different programming lanauges. Therefore, I suggest to build a custom code review based on OWASP secure coding practice categoris (https://owasp.org/www-pdf-archive/OWASP_SCP_Quick_Reference_Guide_v1.pdf) and TOP 10 OWASP, besides results from security tools.
## Access rights, logging, encryption, and updates to the codes
- The codebase will be stored on a source code management system (e.g, Github) and we should use access right of these systems to manage access to the source code.
- In order to update the code, we should have a Continous Delivery pipeline that will simply connect to our servers/nodes running docker containers and trigger a request to clone the latest code. The file system is mapped to docker container, so code update to local file system will also be visible in the docker containers (note: in the Dockerfile, I used COPY instead of volume mapping for the sake of simplicity when running in Docker Swarm environment).
- The application should be configured to send audit logs, especially events related to authentication/authorization process, to a SIEM solution (e.g, QRadar, SPlunk). Then we can configure rules on these solution to detect abnomaly behavior (e.g, number of log-on events is greater than a threshold during a specific time).
- The following encryption should be configured for the application: file-system encryption (if dockers are running on top of virtual machines), database encryption, communication encryption (TLS).
