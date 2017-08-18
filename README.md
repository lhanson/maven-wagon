# maven-wagon implementation with in-line host key in settings.xml

Mirror of Apache Maven wagon, with extensions for in-line host key in settings.xml

Many people want to use the wagon `SingleKnownHostsProvider` provider to manually provide a Host Key when deploying with Maven to a remote server specified in their `settings.xml`.
They often try to overload the knownHostsProvider implementation class this way, to get a SingleKnownHostProvider instead of the default FileKnownHostProvider:

    <server>
    [...]
      <configuration>
        <knownHostsProvider implementation="org.apache.maven.wagon.providers.ssh.knownhost.SingleKnownHostProvider">
        [...]
        </knownHostsProvider>
      </configuration>
    </server>

This **does not work**. Because when the repository URL starts with `scp:`, Plexus, the component manager used by Maven, searches a component with role `org.apache.maven.wagon.Wagon` and hint `scp`, which, in the current Wagon implementation, is of class `org.apache.maven.wagon.providers.ssh.jsch`. This class extends the class AbstractJschWagon in the same package, and this later class defines statically the `file` hint to get a knownHostProvider:

    /**
     * @plexus.requirement role-hint="file"
     */
    private volatile KnownHostsProvider knownHostsProvider;


<knownHostsProvider implementation="org.apache.maven.wagon.providers.ssh.knownhost.SingleKnownHostProvider">
                    <hostKeyChecking>yes</hostKeyChecking>
                    <key>81:66:27:b9:15:36:3a:91:ec:66:43:4f:69:a0:ef:c4:b9:15:36</key>
                </knownHostsProvider>
