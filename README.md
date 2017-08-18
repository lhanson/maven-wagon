# maven-wagon implementation with in-line host key in settings.xml

Mirror of Apache Maven wagon, with extensions for in-line host key configuration in settings.xml

## The problem

Many people want to use the `SingleKnownHostsProvider` wagon provider to manually provide a host key when deploying with Maven to a remote SCP server specified in their `settings.xml`. This is to avoid relying on `$HOME/.ssh/known_hosts` and thus be independant from the user context.
So, they often try to overload the `knownHostsProvider` implementation class the following way, to get a `SingleKnownHostProvider` instead of the default `FileKnownHostProvider`:

    <server>
    [...]
      <configuration>
        <knownHostsProvider implementation="org.apache.maven.wagon.providers.ssh.knownhost.SingleKnownHostProvider">
        [...]
        </knownHostsProvider>
      </configuration>
    </server>

**This does not work**.

Because when the repository URL starts with `scp:`, Plexus, the component manager used by Maven, searches a component with role `org.apache.maven.wagon.Wagon` and hint `scp`, which, in the current Wagon implementation (up to 3.0.1 at least), is of class `org.apache.maven.wagon.providers.ssh.jsch`. This class extends the class `AbstractJschWagon` in the same package, and this later class statically defines a `file` hint to get a `knownHostProvider`. Here is an excerpt from the sources:

    public abstract class AbstractJschWagon
    [...]
    /**
     * @plexus.requirement role-hint="file"
     */
    private volatile KnownHostsProvider knownHostsProvider;

Therefore, this `file` hint makes Plexus use the class `FileKnownHostsProvider` to instanciate the `knownHostsProvider` object. This is because the class `FileKnownHostsProvider` is defined the following way at the beginning of its source file:

    public class FileKnownHostsProvider
    [...]
    * @plexus.component role="org.apache.maven.wagon.providers.ssh.knownhost.KnownHostsProvider"
    *    role-hint="file"

On the contrary, the class `SingleKnownHostProvider` is not defined with role `file` but with role `single`:

    public class SingleKnownHostProvider
    [...]
    * @plexus.component role="org.apache.maven.wagon.providers.ssh.knownhost.KnownHostsProvider"
    *    role-hint="single"

## The solution

To solve this difficulty, this package introduces a patch to the `AbstractJschWagon` class, that adds a new `hostKey` configuration parameter in `settings.xml` and avoids using the `FileKnownHostProvider` when this parameter is defined. The host name and key set in the `hostKey` parameter are published as the unique host key entry used when invoking Jsch to connect to the remote server.

## Using this package

To use this package, just do the following:

    % git clone https://github.com/AlexandreFenyo/maven-wagon.git
    % cd maven-wagon
    % mvn install -DskipTests

Note 1: this will not override your Wagon installation, since this patched distribution is specifically renamed 3.0.1-SINGLE.

Note 2: if you want to run unit testing, just remote `-DskipTests`. This may take a while.

In your `pom.xml`, use the following extension:

    <build>
      <extensions>
        <extension>
          <groupId>org.apache.maven.wagon</groupId>
            <artifactId>wagon-ssh</artifactId>
            <version>3.0.1-SINGLE</version>
        </extension>
      </extensions>
    </build>

In this example, we suppose that you have used the following server definition in your `pom.xml`:

    <distributionManagement>
      <repository>
        <id>myServer</id>
        <url>scp://my.remote.server.com/tmp/repo</url>
      </repository>
    </distributionManagement>

Now, in

  <servers>
    <server>
      <id>myServer</id>
        <username>fenyo</username>
        <privateKey>d:/cygwin64/home/fenyo/.ssh/id_dsa</privateKey>
        <configuration>
          <hostKey>my.remote.server.com ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQD2sVyRPm1OnoudM0Ekm9rp5H/Bwgd9TOaYzmcGcqimm137U0bnvwFA0EnyyjMdzGvgUBIrSTssZRM97p1/0O63gD4cpKvXf6ZYzoHSeX4Zmg2MptD9scqzMF4HewSSMvZIvgNn8h9QmL8dIy2ynudVuE03P+bPCb7Y1eEG5V3JqL++j+HAvbsAwRVaAf1U3EQxgzMpnwwFF2bdUuuqvGJPqfs6S1Vg4ATdGUr8lrmUFemo/lT0+nB5OBYQFyfJRd6fAv8vYkvrANjNBlg7L8m3MUwgl3Jt4xzPjbIlEwI4L9sKQ7P3nVUw55f9zjX8eIjgJSosr1uswJN1LiJjD11F</hostKey>
        </configuration>
      </server>
    </servers>
