# maven-wagon implementation with in-line host key in settings.xml

Mirror of Apache Maven wagon, with extensions for in-line host key in settings.xml

Many people want to use the wagon `SingleKnownHostsProvider` provider to manually provide a host key when deploying with Maven to a remote SCP server specified in their `settings.xml`. This is to avoid relying on `$HOME/.ssh/known_hosts` and thus be user-context independant.
They often try to overload the `knownHostsProvider` implementation class the following way, to get a SingleKnownHostProvider instead of the default FileKnownHostProvider:

    <server>
    [...]
      <configuration>
        <knownHostsProvider implementation="org.apache.maven.wagon.providers.ssh.knownhost.SingleKnownHostProvider">
        [...]
        </knownHostsProvider>
      </configuration>
    </server>

This **does not work**.

Because when the repository URL starts with `scp:`, Plexus, the component manager used by Maven, searches a component with role `org.apache.maven.wagon.Wagon` and hint `scp`, which, in the current Wagon implementation (up to 3.0.1), is of class `org.apache.maven.wagon.providers.ssh.jsch`. This class extends the class `AbstractJschWagon` in the same package, and this later class statically defines a `file` hint to get a `knownHostProvider`. Here is an excerpt from the sources:

    /**
     * @plexus.requirement role-hint="file"
     */
    private volatile KnownHostsProvider knownHostsProvider;

Therefore, this `file` hint makes Plexus use the class `FileKnownHostsProvider` to instanciate the `knownHostsProvider` object. This is because this class is defined this way at the beginning of its source file:

    * @plexus.component role="org.apache.maven.wagon.providers.ssh.knownhost.KnownHostsProvider"
    *    role-hint="file"

On the contrary, the class SingleKnownHostProvider is not defined with role `file` but with role `single`:

    * @plexus.component role="org.apache.maven.wagon.providers.ssh.knownhost.KnownHostsProvider"
    *    role-hint="single"

So, to solve this difficulty, this package introduces a patch to the AbstractJschWagon class, that adds a new `hostKey` configuration parameter in `settings.xml` and avoids using the `FileKnownHostProvider` when this parameter is defined. The host name and key set in the `hostKey` parameter are published as the unique host key entry used when invoking Jsch to connect to the remote server.

