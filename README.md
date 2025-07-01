install java, docker, trivy, kubectl on jenkins
docker run -d -p 8081:8081 sonatype/nexus3
docker run -d -p 9000:9000 sonarqube:lts-community 
enabel rbac - create sa, role, role bindind, create token and add in jenkins
add token from sonarqube on jenkins(security->users->token on sonarqube dashboard)
configure sonarqube server using this credentials
configure manage nodes with nexus username and pass
add the nexus url in pom.xml


Issue faced
docker build failed as the docker grp access for jenkins was given but jenkins was not restarted
