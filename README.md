# TP n°4 — BPEL: orchestration and choreography of a student enrollment

Author: KANA TSOLEFACK Marie Agathe — ISI 4, Information Systems Architecture.

## Content

| Path | What it is |
|---|---|
| `tp-bpel-enrollment/services/` | the 4 JAX-WS partner services (WAR) |
| `tp-bpel-enrollment/bpel-orchestrator/` | Part A: BPEL process, WSDL, XSD, `deploy.xml` |
| `tp-bpel-enrollment/choreography/` | Part B: the 4 Spring Boot event adapters |
| `tp-bpel-enrollment/report/` | PDF report |
| `enrollment-tests.xml` | SoapUI project (4 test cases) |
| `screenshots/` | captures of the runs |

## Requirements

JDK 11, Maven 3.9, Tomcat 9 with `ode.war` (ODE 1.3.8) already in `webapps/`, Artemis 2.33, SoapUI 5.7.
Two variables are used below:

```bash
export TOMCAT_HOME=~/opt/tomcat9
export ODE_PROCESSES=$TOMCAT_HOME/webapps/ode/WEB-INF/processes
```

## 1. Build everything

```bash
cd tp-bpel-enrollment
mvn clean install
```

## 2. Part A

Tomcat must be stopped while copying.

```bash
# partner services
cp services/*/target/*-service.war $TOMCAT_HOME/webapps/

# BPEL process (the partner WSDLs are needed by ODE next to the process
# and at the relative path used by the <import> of the .bpel)
cp -r bpel-orchestrator $ODE_PROCESSES/enrollment
cp services/*/src/main/resources/*.wsdl $ODE_PROCESSES/enrollment/
cp -r --parents services/*/src/main/resources/*.wsdl $ODE_PROCESSES/

$TOMCAT_HOME/bin/catalina.sh run
```

WSDL of the process: http://localhost:8080/ode/processes/EnrollmentService?wsdl
WSDL of a partner, for example the bank: http://localhost:8080/bank-service/BankService?wsdl

Tests: in SoapUI, *File > Import Project* > `enrollment-tests.xml`, then run the suite `TP-BPEL tests`.

| Test case | Expected |
|---|---|
| TC1 enroll 350000 | `ENROLLED` + roomNumber, passId, receipt |
| TC2 enroll 900000 | `REFUSED`, and Tomcat prints `[University] registration STU-… cancelled` |
| TC3 bank pay 500000 | `approved = true` |
| TC4 bank pay 500001 | `approved = false`, receipt `N/A` |

## 3. Part B

```bash
# broker + topics
~/artemis-broker/bin/artemis-service start
for t in EnrollmentRequested StudentRegistered PaymentDecided EnrollmentCompleted; do
  ~/artemis-broker/bin/artemis address create --name $t --multicast
done

# one terminal per adapter
(cd choreography/university-event-adapter && mvn spring-boot:run)
(cd choreography/bank-event-adapter       && mvn spring-boot:run)
(cd choreography/residence-event-adapter  && mvn spring-boot:run)
(cd choreography/transport-event-adapter  && mvn spring-boot:run)
```

Publish the `EnrollmentRequested` event (successful flow, then replace `350000` by `900000` for the refused flow):

```bash
curl -u admin:admin -H "Content-Type: application/json" -H "Origin: http://localhost" \
  -XPOST http://localhost:8161/console/jolokia \
  -d '{"type":"exec","mbean":"org.apache.activemq.artemis:broker=\"0.0.0.0\",component=addresses,address=\"EnrollmentRequested\"","operation":"sendMessage(java.util.Map,int,java.lang.String,boolean,java.lang.String,java.lang.String)","arguments":[{},3,"{\"correlationId\":\"enr-101\",\"type\":\"EnrollmentRequested\",\"data\":{\"student\":{\"name\":\"Marie\",\"program\":\"ISI4\"},\"amount\":\"350000\"}}",false,"admin","admin"]}'
```

The cascade of events is visible in the Artemis console (http://localhost:8161, admin / admin), or with one consumer per topic:

```bash
~/artemis-broker/bin/artemis consumer --destination topic://EnrollmentCompleted --message-count 1 --verbose
```
