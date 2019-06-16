# Under Construction
The tool consists of three parts:
- Parse logging-configuration files.
- Parse logger-configuration related code in source files.
- Detect smells based on predefined rules.

The tool will be released after some refactoring.

## Parse logging-configuration files
To get all the names of loggers used in configurations, we use heuristic method to parse the `XML` files and `properties` files of LOG4J, LOG4J2 and LOGBACK. 

We have tried to parse logging-configuration files in systematic methods, but fail to achieve it. We first tried to convert `properties` files into `XML` files, so that we can use DTD/XSD-based method to parse XML files. (DTD/XSD files are formal definition of schema of XML, which can be used to validate and parse XML files) However, we find that none of the logging libraries provides DTD/XSD files to parse XML files. Moreover, it is not trivial to convert `properties` files into `XML` files.


## Parse logger-configuration related code in source files
Generally, we have to built the projects to get all the class files and jar files needed to run the projects. 
- On one hand, some loggers are declared by annotations in Java and the definition code of loggers are injected into bytecode during building. For example, [Lombok](https://github.com/rzwitserloot/lombok) provide `@Logger` annotations to declare loggers, and it is commonly used in Alibaba's projects.
- On the other hand, we need to get the class files of the dependencies to get the logger defined in external libraries.

After that, we use IBM WALA to parse the class files and jar files. For each class, we get all [SSAInvokeInstruction](https://wala.github.io/javadoc/com/ibm/wala/ssa/SSAInvokeInstruction.html), which represent all method invocations in WALA's IR (Intermediate Representation). And we check if the target of SSAInvokeInstruction is logger definition method by comparing with predefined list. This list is retrieved manually. We get all method invocations, whose names start with "getLog" or "getLogger", and check the source code to determine if it is logger definition manually.
```
{
	"Lorg/apache/commons/logging/LogFactory.getLog": "CommonsLog",
	"Lorg/jboss/logging/Logger.getLogger": "JBossLog",
	"Ljava/util/logging/Logger.getLogger": "JUL",
	"Lorg/apache/log4j/Logger.getLogger": "Log4j",
	"Lorg/apache/logging/log4j/LogManager.getLogger": "Log4j2",
	"Lorg/slf4j/LoggerFactory.getLogger": "Slf4j",
	"Lcom/ibatis/common/logging/LogFactory.getLog": "Ibatis",
	"Lorg/eclipse/jetty/util/log/Log.getLogger": "Jetty",
	"Lorg/mortbay/log/Logger.getLogger": "Jetty",
	"Lorg/jboss/netty/logging/InternalLoggerFactory.getInstance": "Netty",
	"Lorg/datanucleus/util/NucleusLogger.getLoggerInstance": "Nucleus",
	"Lorg/jpox/util/JPOXLogger.getLoggerInstance": "JPOX"
}
```
The next step is to get the names of loggers, and we find that the loggers are usually defined in following ways:
```
	private final Log log = LogFactory.getLogger("StringLiteral");
	private final Log log = LogFactory.getLogger(getClass());
	private final Log log = LogFactory.getLogger(Main.class.getName());
	private final Log log = LogFactory.getLogger(this.getClass());
```
For each case, we implement different logic to retrieve the logger names. However, there are some rare cases where the logger names are  defined dynamically. We do not take them into consideration here.


## Detect smells based on predefined rules
Take invalid logger for example, they are defined as loggers that are used in configurations, but do not have any matched definition in source code. Let LC denote all the loggers used in configurations and LD denotes all the loggers defined in source code. The  pseudo-code as follows:
```
for l1 in LC:
  invalid = true
  for l2 in LD:
    if l1.equals(l2) or l1.startwith(l2):
      invalid = false
```