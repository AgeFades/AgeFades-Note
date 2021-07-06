### 示例

- 一次性部署多个Jar，要拆分的话自己按需拆分

```shell
#!/bin/bash
# 拉代码、Maven编译打包
cd /home/developer/jdy/JDY-Backend/jdy-parent
git pull
mvn clean install -DskipTests

# 杀进程
ps -ef | grep jdy-agent-service | grep -v grep |  awk '{print $2}' | xargs kill -9
ps -ef | grep jdy-system-service | grep -v grep |  awk '{print $2}' | xargs kill -9
ps -ef | grep jdy-insurance-service | grep -v grep |  awk '{print $2}' | xargs kill -9
ps -ef | grep jdy-mechanism-service | grep -v grep |  awk '{print $2}' | xargs kill -9
ps -ef | grep jdy-gateway | grep -v grep |  awk '{print $2}' | xargs kill -9

# 启动项目
nohup java -jar /home/developer/jdy/JDY-Backend/jdy-parent/jdy-agent/jdy-agent-service/target/jdy-agent-service.jar --spring.profiles.active=dev > /home/developer/jdy/logs/agent.log 2>&1 &
nohup java -jar /home/developer/jdy/JDY-Backend/jdy-parent/jdy-system/jdy-system-service/target/jdy-system-service.jar --spring.profiles.active=dev > /home/developer/jdy/logs/system.log 2>&1 &
nohup java -jar /home/developer/jdy/JDY-Backend/jdy-parent/jdy-insurance/jdy-insurance-service/target/jdy-insurance-service.jar --spring.profiles.active=dev > /home/developer/jdy/logs/insurance.log 2>&1 &
nohup java -jar /home/developer/jdy/JDY-Backend/jdy-parent/jdy-mechanism/jdy-mechanism-service/target/jdy-mechanism-service.jar --spring.profiles.active=dev > /home/developer/jdy/logs/mechanism.log 2>&1 &
nohup java -jar /home/developer/jdy/JDY-Backend/jdy-parent/jdy-gateway/target/jdy-gateway.jar --spring.profiles.active=dev > /home/developer/jdy/logs/gateway.log 2>&1 &
```

### 单个jar示例

```shell
#!/bin/bash
ps -ef | grep jdy-agent-service | grep -v grep |  awk '{print $2}' | xargs kill -9
cd /home/developer/jdy/JDY-Backend/jdy-parent
git pull
mvn clean install -DskipTests
nohup java -jar /home/developer/jdy/JDY-Backend/jdy-parent/jdy-agent/jdy-agent-service/target/jdy-agent-service.jar --spring.profiles.active=dev > /home/developer/jdy/logs/agent.log 2>&1 &
```

