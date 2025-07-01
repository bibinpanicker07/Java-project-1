install java, docker, trivy, kubectl on jenkins
docker run -d -p 8081:8081 sonatype/nexus3
docker run -d -p 9000:9000 sonarqube:lts-community 
enabel rbac - create sa, role, role bindind, create token and add in jenkins
