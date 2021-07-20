# AWS Lambda with Spring Cloud Function and Spring Native

## Background

Java cold start is not great in AWS Lambda comparing to languages like python and NodeJS.
Additionally, if we decide to use frameworks like Spring, the warming time will be even longer. I'm
pretty used to use Spring and wanted to exploit this experience while developing functions on AWS
using Spring Cloud Function. So, how can we justify the cost of using Java (and spring)?

Well, I stumbled on a video
describing [Spring Native](https://www.youtube.com/watch?v=JsUAGJqdvaA&t=3687s) and I thought this
might be a solution.

## Project Structure

```
spring-native-aws-lambda
  .mvn/wrapper           # for building without installing maven    
  mvnw                   # build on unix like OS
  mvnw.cmd               # build on windows 
  Dockerfile             # Building for aws environment using amazonlinux image
  pom.xml                # project dependencies
  src
    assembly
        native.xml       # Used by maven-assembly-plugin to zip the native image and shell/native/bootstrap
    shell
      native
        bootstrap        # bootstraps the function in AWS Lambda env
    main
      java
        com.coffeebeans.springnativeawslambda
```
Let's navigate th code in these files.

## Code

### pom.xml

#### Spring Cloud Function Dependencies

**1. Dependency Management**

Spring cloud function (SCF) provides a pom dependency that we can import into `dependencyManagement`
element

```xml

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>2020.0.2</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```
This is helps us to import compatible version for SCF

**2. Dependencies**

```xml

<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-function-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-function-adapter-aws</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>
  <dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-lambda-java-events</artifactId>
    <version>2.0.2</version>
  </dependency>
  <dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-lambda-java-core</artifactId>
    <version>1.1.0</version>
    <scope>provided</scope>
  </dependency>
  <dependency>
    <groupId>org.reactivestreams</groupId>
    <artifactId>reactive-streams</artifactId>
    <version>1.0.0</version>
  </dependency>
</dependencies>
```
**NOTE:** that we will be using reactive mode,
hence `org.springframework.boot:spring-boot-starter-webflux`
and `org.reactivestreams:reactive-streams`
---
#### Spring Native Dependencies

**1. Dependency Management**

```xml

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.experimental</groupId>
      <artifactId>spring-native</artifactId>
      <version>0.10.0</version>
    </dependency>
  </dependencies>
</dependencyManagement>
```

**2. Dependency**

```xml

<dependencies>
  <dependency>
    <groupId>org.springframework.experimental</groupId>
    <artifactId>spring-native</artifactId>
  </dependency>
</dependencies>
```

**3. Plugins**

```xml

<build>
  <plugins>
    <plugin>
      <groupId>org.springframework.experimental</groupId>
      <artifactId>spring-aot-maven-plugin</artifactId>
      <version>${spring-native.version}</version>
      <executions>
        <execution>
          <id>test-generate</id>
          <goals>
            <goal>test-generate</goal>
          </goals>
        </execution>
        <execution>
          <id>generate</id>
          <goals>
            <goal>generate</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```
**4. Profile**

```xml

<profiles>
  <!-- Enable building a native image using a local installation of native-image with GraalVM native-image-maven-plugin -->
  <profile>
    <id>native-image</id>
    <properties>
      <!-- Avoid a clash between Spring Boot repackaging and native-image-maven-plugin -->
      <classifier>exec</classifier>
    </properties>
    <build>
      <plugins>
        <plugin>
          <groupId>org.graalvm.nativeimage</groupId>
          <artifactId>native-image-maven-plugin</artifactId>
          <version>21.1.0</version>
          <configuration>
            <imageName>${project.artifactId}</imageName>
            <buildArgs>${native.build.args}</buildArgs>
          </configuration>
          <executions>
            <execution>
              <goals>
                <goal>native-image</goal>
              </goals>
              <phase>package</phase>
            </execution>
          </executions>
        </plugin>

        <plugin>
          <!-- packages native image and bootstrap file for uploading to AWS -->
          <artifactId>maven-assembly-plugin</artifactId>
          <executions>
            <execution>
              <id>native-zip</id>
              <phase>package</phase>
              <goals>
                <goal>single</goal>
              </goals>
              <inherited>false</inherited>
            </execution>
          </executions>
          <configuration>
            <descriptors>
              <descriptor>src/assembly/native.xml</descriptor>
            </descriptors>
          </configuration>
        </plugin>
      </plugins>
    </build>
  </profile>
</profiles>
```
---
One last step is, make sure that we can pull dependencies and plugins
from `org.springframework.experimental`. For this, we need to add `https://repo.spring.io/release`
as a repository and a `pluginrepository` in `settings.xml`
#### Settings.xml

```xml

<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <profiles>
    <profile>
      <id>spring-native-demo</id>
      <repositories>
        <repository>
          <id>spring-releases</id>
          <name>Spring Releases</name>
          <url>https://repo.spring.io/release</url>
        </repository>
      </repositories>
      <pluginRepositories>
        <pluginRepository>
          <id>spring-releases</id>
          <name>Spring Releases</name>
          <url>https://repo.spring.io/release</url>
        </pluginRepository>
      </pluginRepositories>
    </profile>
  </profiles>
  <activeProfiles>
    <activeProfile>spring-native-demo</activeProfile>
  </activeProfiles>
</settings>
```
---
Now, we are ready to write our java function
### Application Configuration
The application is configured like any other spring boot application with very few differences
- It uses `@SpringBootConfiguration` instead of `@SpringBootApplication`
- `FunctionalSpringApplication#run(Class, String[])` instead of `SpringApplication#run(Class, String[])`
- We made a native hint to our application to serialise our models properly
```java
@SerializationHint(types = {Request.class, Response.class})
@SpringBootConfiguration
public class SpringNativeAwsLambdaApplication implements ApplicationContextInitializer<GenericApplicationContext> {

  public static void main(final String[] args) {
    FunctionalSpringApplication.run(SpringNativeAwsLambdaApplication.class, args);
  }

  @Override
  public void initialize(final GenericApplicationContext context) {
    context.registerBean("exampleFunction",
        FunctionRegistration.class,
        () -> new FunctionRegistration<>(new ExampleFunction()).type(ExampleFunction.class));
  }
}
```

```properties
#this is required to be true for custom runtime where we will run our native-image, override it to false if you deploying spring cloud function to aws without spring-native
spring.cloud.function.web.export.enabled=true
```
**NOTE:** Spring Cloud Function supports a "functional" style of bean declarations for small apps where you need fast startup So, we used it in our application (this is the `initialize` method as well implementing `ApplicationContextInitializer`)

---
### The Function
Spring Cloud Functions allows us to create functions by extending java8 3 core functional interfaces
- `java.util.function.Function`
- `java.util.function.Consumer`
- `java.util.function.Supplier`

More about each use case can be found [here](https://cloud.spring.io/spring-cloud-function/reference/html/spring-cloud-function.html#_programming_model)

For our example we extended `java.util.function.Function` 

**NOTE:** We placed our class in a package called `functions`. This way, it will be [scanned automatically](https://cloud.spring.io/spring-cloud-function/reference/html/spring-cloud-function.html#_function_component_scan) even without `@Component` annotation. 
```java
@Slf4j
public class ExampleFunction implements Function<Request, Response> {

  @Override
  public Response apply(final Request request) {
    log.info("Converting request into a response...");

    final Response response = Response.builder()
        .name(request.getName())
        .saved(true)
        .build();

    log.info("Converted request into a response.");
    return response;
  }
}
```
---
#### Models 
Our models are implementing `Serializable`

---

## Building
### Building for Local Environment 
```shell
./mvnw -ntp clean package -Pnative-image
target/spring-native-aws-lambda
```
#### Building for AWS Environment
- Run the following commands (in local terminal or CI/CD pipeline)

```shell
mkdir target

docker build --no-cache \
--tag spring-native-aws-lambda:0.0.1-SNAPSHOT \
--file Dockerfile .

docker ps -a  
docker rm spring-native-aws-lambda                                                                                                                                
docker run --name spring-native-aws-lambda spring-native-aws-lambda:0.0.1-SNAPSHOT                                                            
docker cp spring-native-aws-lambda:app/target/ .
```
- Upload the `spring-native-aws-lambda-0.0.1-SNAPSHOT-native-zip.zip` file to aws lambda,
- Set the handler to `exampleFunction`
- Test, and
- Et voila! It runs with 500 ms for cold start (Not bad for spring boot application, I guess)

### Testing locally
You can access the application on whatever port you started it on using http request
```shell
curl --location --request POST 'http://localhost:8080/exampleFunction' \
--header 'Content-Type: application/json' \
--data-raw '{
    "name": "CoffeeBeans"
  }'
```

