# MultipleProjectSonarQube
Running SonarQube on Multiple Project (Not Multiple Module)

## Experiment plan
1) run SonarQube analysis on source from big project like https://github.com/wildfly/wildfly
2) run SonarQube analysis on jbossws-* dependencies of jbossws-cxf-client
   - as separate SonarQube projects
   - as one big SonarQube project
3) run SonarQube analysis (one big SonarQube project) on all the dependencies of jbossws-cxf-client
   - using Sonar Maven Plugin
   - using Sonar Runner
4) run SonarQube analysis (one big SonarQube project) on all the dependencies of WildFly

### Step 1 - Analysis on WildFly sources
```bash
rm -rf workspace && mkdir workspace

git clone https://github.com/wildfly/wildfly.git workspace/wildfly
mvn -f workspace/wildfly/pom.xml clean install -DskipTests -Denforcer.skip=true -Dcheckstyle.skip=true


wget -O workspace/sonarqube-6.3.1.zip https://sonarsource.bintray.com/Distribution/sonarqube/sonarqube-6.3.1.zip
unzip -q -d workspace/ workspace/sonarqube-6.3.1.zip

workspace/sonarqube-6.3.1/bin/linux-x86-64/sonar.sh start
sleep 10

mvn -f workspace/wildfly/pom.xml org.sonarsource.scanner.maven:sonar-maven-plugin:3.2:sonar -Dsonar.host.url=http://localhost:9000/

firefox http://localhost:9000/

```

### Step 2a) - Analysis on jbossws-* dependencies of jbossws-cxf-client in separate projects
```bash
rm -rf workspace && mkdir workspace

wget -O workspace/sonarqube-6.3.1.zip https://sonarsource.bintray.com/Distribution/sonarqube/sonarqube-6.3.1.zip
unzip -q -d workspace/ workspace/sonarqube-6.3.1.zip

workspace/sonarqube-6.3.1/bin/linux-x86-64/sonar.sh start
sleep 10

mkdir workspace/jbossws-cxf-client
cat <<EOF > workspace/jbossws-cxf-client/cxf-client-pom.xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <name>JBoss Web Services - Client test</name>

  <groupId>org.jboss.ws.cxf</groupId>
  <artifactId>jbossws-cxf-client-test</artifactId>
  <version>1.0.0.Final</version>

  <properties>
      <jbossws-cxf-client-version>5.1.8.Final</jbossws-cxf-client-version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.jboss.ws.cxf</groupId>
      <artifactId>jbossws-cxf-client</artifactId>
      <version>\${jbossws-cxf-client-version}</version>
    </dependency>
  </dependencies>

</project>
EOF
mvn -f workspace/jbossws-cxf-client/cxf-client-pom.xml dependency:sources > workspace/jbossws-cxf-client/dependencies.txt

wsprojects=(jbossws-api jbossws-spi jbossws-common jbossws-common-tools jbossws-cxf-client)
rm -rf workspace/jbossws-cxf-client/jbossws-projects && mkdir workspace/jbossws-cxf-client/jbossws-projects
for PP in ${wsprojects[@]}; do
  VERSION=`grep ":jbossws" workspace/jbossws-cxf-client/dependencies.txt| sed "s/\[INFO\]    //g" | grep "$PP:" | cut -d: -f 5 `
  PROJECT=`echo "$PP" | sed "s/jbossws-cxf-client/jbossws-cxf/g"`
  echo "https://github.com/jbossws/$PROJECT/tree/$PROJECT-$VERSION" | sed "s/jbossws-cxf-client/jbossws-cxf/g"
  git clone --branch $PROJECT-$VERSION https://github.com/jbossws/$PROJECT.git workspace/jbossws-cxf-client/jbossws-projects/$PROJECT
  mvn -f workspace/jbossws-cxf-client/jbossws-projects/$PROJECT/pom.xml org.sonarsource.scanner.maven:sonar-maven-plugin:3.2:sonar -Dsonar.host.url=http://localhost:9000/
done


firefox http://localhost:9000/
```

### Step 2b) - Analysis of all dependencies for jbossws-cxf-client in one big SonarQube project
```bash
rm -rf workspace && mkdir workspace

wget -O workspace/sonarqube-6.3.1.zip https://sonarsource.bintray.com/Distribution/sonarqube/sonarqube-6.3.1.zip
unzip -q -d workspace/ workspace/sonarqube-6.3.1.zip

workspace/sonarqube-6.3.1/bin/linux-x86-64/sonar.sh start
sleep 10

mkdir workspace/jbossws-cxf-client
cat <<EOF > workspace/jbossws-cxf-client/cxf-client-pom.xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <name>JBoss Web Services - Client test</name>

  <groupId>org.jboss.ws.cxf</groupId>
  <artifactId>jbossws-cxf-client-test</artifactId>
  <version>1.0.0.Final</version>

  <properties>
      <jbossws-cxf-client-version>5.1.8.Final</jbossws-cxf-client-version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.jboss.ws.cxf</groupId>
      <artifactId>jbossws-cxf-client</artifactId>
      <version>\${jbossws-cxf-client-version}</version>
    </dependency>
  </dependencies>

</project>
EOF
mvn -f workspace/jbossws-cxf-client/cxf-client-pom.xml dependency:sources > workspace/jbossws-cxf-client/dependencies.txt

rm -rf workspace/jbossws-cxf-client/jbossws-all-in-one && mkdir workspace/jbossws-cxf-client/jbossws-all-in-one

cat <<EOF > workspace/jbossws-cxf-client/jbossws-all-in-one/pom.xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
        <modelVersion>4.0.0</modelVersion>

        <name>JBoss Web Services CXF Client - All-In-One</name>
        <groupId>org.experiments.rsvoboda</groupId>
        <artifactId>jbossws-all-in-one</artifactId>
        <version>1.0.0</version>
        <packaging>pom</packaging>

        <modules>
EOF

for GAV in `grep ".*:.*:.*:.*:.*:.*" workspace/jbossws-cxf-client/dependencies.txt| sed "s/\[INFO\]    //g" | grep -v system | cut -d: -f1-2,5`; do
  MODULE=`echo $GAV | tr ":" "-"`
  mvn org.apache.maven.plugins:maven-dependency-plugin:2.8:unpack -Dartifact=$GAV:jar:sources -DoutputDirectory=workspace/jbossws-cxf-client/jbossws-all-in-one/$MODULE/src/main/java || true
  cat <<EOF >> workspace/jbossws-cxf-client/jbossws-all-in-one/pom.xml
                <module>$MODULE</module>
EOF

  cat <<EOF > workspace/jbossws-cxf-client/jbossws-all-in-one/$MODULE/pom.xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
        <modelVersion>4.0.0</modelVersion>

        <parent>
                <groupId>org.experiments.rsvoboda</groupId>
                <artifactId>jbossws-all-in-one</artifactId>
                <version>1.0.0</version>
        </parent>

        <artifactId>$MODULE</artifactId>
</project>
EOF

done

cat <<EOF >> workspace/jbossws-cxf-client/jbossws-all-in-one/pom.xml
        </modules>
</project>
EOF

mvn -f workspace/jbossws-cxf-client/jbossws-all-in-one/pom.xml  org.sonarsource.scanner.maven:sonar-maven-plugin:3.2:sonar -Dsonar.host.url=http://localhost:9000/ -Dsonar.exclusions=**/com/google/common/util/concurrent/Monitor.java


firefox http://localhost:9000/
```
Note: exclude because of http://stackoverflow.com/questions/43962471/sonarqube-analysis-of-guava-v18-internalprefixunaryexpression-cannot-be-cast-t