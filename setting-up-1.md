# Setting Up

Kilua RPC supports different server-side frameworks - Ktor, Jooby, Spring Boot, Javalin, Vert.x and Micronaut - so you have to choose one of them for your needs. It's worth to mention, that common and js/wasmJs code of your application are exactly the same for all servers, as well as the greater part of the actual service implementation for the jvm target. The differences are tied to initialization code and additional server side functionalities (e.g. authentication).

## Requirements

You need JDK 21 to build Kilua RPC application. Your project should use standard Kotlin multiplatform layout, with separate sources sets for `common`, `jvm` and `js` and/or `wasmJs` code, located in separate directories: `src/commonMain` , `src/jvmMain`  and `src/jsMain` and/or `src/wasmJsMain`. You can also declare a shared `src/webMain` sources set if you want do develop for both `js` and `wasmJs` targets:

```kotlin
val webMain by creating {
    dependsOn(commonMain)
    dependencies {
    }
}
val jsMain by getting {
    dependsOn(webMain)
}
val wasmJsMain by getting {
    dependsOn(webMain)
}
```

## Development

During the development phase you compile and run js and jvm targets separately.

#### Frontend

To run the frontend application with Gradle continuous build enter:

```
./gradlew -t jsRun                                (on Linux)
gradlew.bat -t jsRun                              (on Windows)
```

#### Backend

To run the backend application enter:

```
./gradlew jvmRun                                    (on Linux)
gradlew.bat jvmRun                                  (on Windows)
```

There are different levels of support when it comes to auto-reload. Javalin doesn't support auto-reload at all. Jooby, Vert.x and Micronaut have built-in auto-reload based on sources monitoring, so it works out of the box. In case of Ktor and Spring Boot auto-reload is based on the classpath monitoring, so you have to run another Gradle process for continuous build:

```
### Ktor or Spring Boot
./gradlew -t compileKotlinJvm                       (on Linux)
gradlew.bat -t compileKotlinJvm                     (on Windows)
```

After both parts of your application are running, you can open [http://localhost:3000/](http://localhost:3000/) in your favourite browser. Changes made to your sources (in any source set) should be automatically applied to your running application.&#x20;

## Production

To build complete application optimized for production run:

```
./gradlew clean jar                   (on Linux)
gradlew.bat clean jar                 (on Windows)
```

The application jar will be saved in `build/libs` directory (`projectname-1.0.0-SNAPSHOT.jar`). You can run your application with  the`java -jar` command.

```
java -jar build/libs/projectname-1.0.0-SNAPSHOT.jar
```
