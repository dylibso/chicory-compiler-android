# Experimental Chicory Android Compiler

## Installing

The experimental Chicory Android compiler is currently available through the [GitHub Package Registry](https://docs.github.com/en/enterprise-server@3.15/packages/working-with-a-github-packages-registry/working-with-the-apache-maven-registry).

### Maven

Configure Maven. Create a `settings.xml` file alongside your `pom.xml` or just add to your `~/.m2/settings.xml`:

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                      http://maven.apache.org/xsd/settings-1.0.0.xsd">

  <activeProfiles>
    <activeProfile>github</activeProfile>
  </activeProfiles>

  <profiles>
    <profile>
      <id>github</id>
      <repositories>
        <repository>
          <id>central</id>
          <url>https://repo.maven.apache.org/maven2</url>
        </repository>
        <repository>
          <id>github</id>
          <url>https://maven.HOSTNAME/OWNER/REPOSITORY</url>
          <snapshots>
            <enabled>true</enabled>
          </snapshots>
        </repository>
      </repositories>
    </profile>
  </profiles>

  <servers>
    <server>
      <id>github</id>
      <username>USERNAME</username>
      <password>TOKEN</password>
    </server>
  </servers>
</settings>
```

where USERNAME is your GitHub username, and TOKEN is a [Classic Token](https://github.com/settings/tokens).

Now you can depend on the package in your `pom.xml`:

```
<dependencies>
 <dependency>
    <groupId>com.dylibso.chicory</groupId>
    <artifactId>android-aot</artifactId>
    <version>999-SNAPSHOT</version>
  </dependency>
</dependencies>
```

### Gradle

https://docs.github.com/en/enterprise-server@3.15/packages/working-with-a-github-packages-registry/working-with-the-gradle-registry

In your `settings.gradle.kts`:

```kotlin
repositories {
    maven {
        url = uri("https://REGISTRY_URL/OWNER/REPOSITORY")
        credentials {
            username = project.findProperty("gpr.user") as String? ?: System.getenv("USERNAME")
            password = project.findProperty("gpr.key") as String? ?: System.getenv("TOKEN")
        }
    }
}
```

where USERNAME is your GitHub username, and TOKEN is a [Classic Token](https://github.com/settings/tokens). You can configure them in your `gradle.properties` or set the `USERNAME` and `TOKEN` environment variables.

Then add the dependency to your `build.gradle.kts`:

```
dependencies {
    implementation("com.dylibso.chicory:android-aot:999-SNAPSHOT")
}
```

## Using the Android Chicory Compiler

Once you have the `com.dylibso.chicory:android-aot` dependency in your classpath, [you only need to plug the `MachineFactory`.](https://chicory.dev/docs/usage/runtime-compiler)

```java
import com.dylibso.chicory.compiler.MachineFactoryCompiler;
import com.dylibso.chicory.wasm.Parser;
import com.dylibso.chicory.wasm.WasmModule;
import com.dylibso.chicory.experimental.android.aot.AotAndroidMachine;

var module = Parser.parse(new File("your.wasm"));
var instance = Instance.builder(module).
        withMachineFactory(AotAndroidMachine::new).
        build();
```

Because of the memory and stack limitations on the main Android thread, we strongly advise to initialize and invoke the instance on its own thread, with a generous amount of stack memory:

```
Runnable r = () -> instance.call(...);
Thread t = new Thread( new ThreadGroup("chicory"), r, "chicory-thread", 8 * 1024 * 1024) )
```

## Using the Android Chicory Compiler with Extism

The `Extism` Chicory SDK supports the Android compiler extension since version `0.2.0`. In this case, you will write:

```java
        var path = Path.of("https://github.com/extism/plugins/releases/download/v1.1.1/count_vowels.wasm");
        var wasm = ManifestWasm.fromFilePath(path).build();
        var manifest = Manifest.ofWasms(wasm)
                .withOptions(new Manifest.Options()
                        .withMachineFactory(AotAndroidMachine::new)).build();
        var plugin = Plugin.ofManifest(manifest).build();
```

Because of the memory and stack limitations on the main Android thread, we strongly advise to initialize and invoke the plugin on its own thread, with a generous amount of stack memory:

```
Runnable r = () -> plugin.call(...);
Thread t = new Thread( new ThreadGroup("chicory"), r, "chicory-thread", 8 * 1024 * 1024) )
```


## Using the Android Chicory Compiler with `mcpx4j`

The `mcpx4j` library to run [mcp.run](mcp.run) tools on-device, is built on Extism and it also supports the Android AOT compiler.
In this case, you will write something like:

```kotlin
        val mcpx =
            // Configure the MCP.RUN Session
            Mcpx.forApiKey(BuildConfig.mcpRunKey)
                .withServletOptions(
                    McpxServletOptions.builder()
                        .withMachineFactory{ AotAndroidMachine(it) }
                        // Setup an HTTP client compatible with Android
                        // on the Chicory runtime
                        .withChicoryHttpConfig(AndroidHttpConfig.get())
                        // Configure an alternative, Android-specific logger
                        .withChicoryLogger(AndroidLogger("mcpx4j-runtime"))
                        .build())
                // Configure also the MCPX4J HTTP client to use
                // the Android-compatible implementation
                .withHttpClientAdapter(HttpUrlConnectionClientAdapter())
                .withProfile(BuildConfig.profile)
                .build()
```

Because of the memory and stack limitations on the main Android thread, we strongly advise to initialize and invoke the plugin on its own thread, with a generous amount of stack memory; for instance:

```kotlin
    val service = Executors.newSingleThreadExecutor {
        Thread(ThreadGroup("chicory"), it, "chicory-thread", 8 * 1024 * 1024) }

      val call: Callable<String> = Callable { tool.call(input) }
      val res: String = service.submit(call).get()
```


