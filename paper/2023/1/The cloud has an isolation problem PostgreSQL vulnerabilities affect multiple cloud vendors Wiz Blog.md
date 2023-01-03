# The cloud has an isolation problem: PostgreSQL vulnerabilities affect multiple cloud vendors | Wiz Blog
As part of our ongoing research into the cloud, Wiz Research found vulnerabilities in popular PostgreSQL-as-a-Service offerings of multiple cloud vendors. These vulnerabilities were not the result of a bug in the PostgreSQL codebase, but rather in the code that cloud vendors introduced to the project to make it fit their needs. These vulnerabilities could be used as the initial foothold required for more complex attacks, such as in cross-tenant vulnerabilities like [ExtraReplica](https://www.wiz.io/blog/wiz-research-discovers-extrareplica-cross-account-database-vulnerability-in-azure-postgresql/). Disclosed in April 2022, ExtraReplica could allow attackers to access the databases of other customers in Azure Database for PostgreSQL (Flexible Server).

This post focuses on the previously undisclosed technical details of our research and reveals for the first time our exploration of the infrastructure of another major cloud provider, Google Cloud Platform (GCP), exploiting the same type of vulnerability to gain initial access to the environment. Given the success of our research, we delve into the importance of enforcing and maintaining tenant isolation to prevent cross-tenant vulnerabilities and discuss the complexity of conducting a broad, coordinated disclosure across impacted parties. Finally, we outline our ongoing efforts to help overcome these challenges by creating a common forum in which all cloud vendors can discuss remediation.

The cloud has an isolation problem
----------------------------------

In the past year, Wiz Research conducted extensive research on a fundamental premise of the cloud – **tenant isolation**. We examined how effectively the environments of customers are isolated from each other across different cloud service providers (CSPs). Specifically, we focused on how cloud isolation is enforced in managed services. Many managed services offered by CSPs are based on popular open-source software that runs on top of vendor-managed compute instances. In many cases, this is legacy software that was not designed with cloud multi-tenancy needs in mind. This raises a question: How are CSPs adapting legacy open-source software to fit the needs of the cloud?

Focus on PostgreSQL
-------------------

After investigating the PostgreSQL offerings of multiple major cloud providers, we discovered multiple vulnerabilities in the modifications that these cloud providers introduced to their respective PostgreSQL projects to make it a multi-tenant managed service. These vulnerabilities allowed us to execute arbitrary commands on the vendor-managed compute instances of multiple PostgreSQL-as-a-Service offerings in order to gain footholds within cloud provider internal networks. As a result, we gained an inside look into the operations of the internet’s most significant database services. In extreme cases, we were able to leverage the vulnerabilities to achieve unauthorized cross-tenant data access to other customers using the affected service.

### What is PostgreSQL?

PostgreSQL is one of the most popular database servers in the world. It is a powerful, open source and mature object-relational database used by thousands of organizations to store different types of data. It has earned a strong reputation for its proven architecture and reliability. Due to the popularity of the project, it is offered as a managed service by most CSPs.

### PostgreSQL in the cloud

PostgreSQL is a 25-year-old project. It was first designed and developed before cloud computing and managed services even existed, so its authors could not possibly have anticipated the future needs of the cloud. When CSPs started adapting PostgreSQL for managed database solutions, they discovered that it lacked a permissions model suitable for a managed service. Customers were accustomed to having full admin capabilities over their non-cloud managed databases, but CSPs could not allow customers to have capabilities that might endanger the security or stability of PostgreSQL-as-a-Service in the cloud. For example, all CSPs offer customers similar semi-admin capabilities such as creating [event triggers](https://www.postgresql.org/docs/9.3/event-triggers.html), [checkpoints](https://www.postgresql.org/docs/current/sql-checkpoint.html), loading [extensions](https://www.postgresql.org/docs/current/sql-createextension.html), managing replications and more. But at the same time, CSPs restrict customers from accessing the file system of the compute instance hosting the database, or executing code on the underlying server.

Every CSP tackled this problem a bit differently. Because PostgreSQL has a very limited permissions model, and basically does not allow only certain admin capabilities, CSPs had to introduce changes to PostgreSQL to allow users certain admin capabilities while still restricting them from performing unsafe operations. Some CSPs introduced these changes via extensions or custom configurations. Others went so far as modifying the source code of the PostgreSQL engine itself and maintaining their own fork. All of these approaches can lead to unexpected security issues.

To showcase our findings, we will examine two vulnerabilities we discovered in Google Cloud Platform and Azure.

Case study 1: Google Cloud Platform Cloud SQL
---------------------------------------------

Launched in October 2011, Cloud SQL is one of the flagship services offered by GCP, providing customers with a fully managed database service, automated backups, replication, encryption, and capacity increases while still using traditional database engines: MySQL, PostgreSQL and SQL Server.

Our research focuses solely on Cloud SQL’s PostgreSQL implementation. Our first step in the research was to try to execute code on the underlying virtual machine hosting our PostgreSQL instance.

### PostgreSQL COPY statement

PostgreSQL has a built-in feature for command execution, using the COPY statement. The COPY statement can be used to execute commands and read from or write to files.

For example, we can use the COPY statement to execute the id command and query the output of the command:

Figure 1: PostgreSQL command execution using COPY

Knowing this, we tried to execute the same query on Google Cloud SQL, but found that we lacked the specific privileges required for command execution:

Figure 2: Code execution in operating system level fails due to lack of privileges

### Managed PostgreSQL permissions

It is relatively easy to grasp the PostgreSQL permission model. It consists of a main component called **role**, which can thought of as equivalent to a user or a group. A role can also have [role attributes](https://www.postgresql.org/docs/current/role-attributes.html) that define its privileges. For example, there is a fundamental attribute called \`login privilege\` that allows roles to log in to the database (making them “users”). Another interesting attribute is the \`superuser status\`, which allows roles to be database administrators, bypassing all permission checks, except the right to log in. This is every attacker’s dream. In the figure above, when we tried to execute code with the COPY command, we received an error that we didn’t have the superuser privilege. So what privileges _do_ we have?

The following list describes the roles in our managed database:

We can make some interesting observations. The `cloudsqladmin` role is the database administrator that Google uses to manage the database. We can also see that our user (`postgres`) is part of a role called `cloudsqlsuperuser` which doesn’t have high permissions. Or does it?

### We are semi-superusers

Apparently, we are not superusers but we do have some superuser capabilities. According to GCP’s [official documentation](https://cloud.google.com/sql/docs/postgres/users#superuser_restrictions), the `cloudsqlsuperuser` grants our user with the following capabilities:

As this is not the default PostgreSQL behavior, we can safely assume that GCP introduced modifications to the PostgreSQL engine (perhaps using the `cloudsqlsuperuser` role behind the scenes) in order to grant some high-privileged capabilities to low-privileged users. Could we abuse these changes to elevate our privileges and gain more superuser capabilities like the COPY command?

### Privilege escalation via ALTER TABLE

One of the modifications that GCP introduced to the engine allows the `cloudsqlsuperuser` role to arbitrarily change the ownership of a table to any user or role in the database. At first, it may appear that this small change should not have any significant security consequences. However, the developers of PostgreSQL probably had a good reason to limit this operation strictly to superusers. At the time, we did not have the modified Google Cloud SQL Postgres binary, so we discovered this capability by experimenting with our databases, and figuring out which non-default capabilities we possessed. This abnormal behavior sparked our interest when we tried to alter a table’s owner and set it to the high-privileged user `cloudsqladmin`, which is the database’s instance superuser.

By attempting to set the `cloudsqladmin` user as the owner of `test_table` we expected to get the following error:

Figure 3: Cloud SQL ALTER TABLE expectation

So imagine our surprise when we actually got:

Figure 4: Cloud SQL ALTER TABLE reality

To understand the potential security impact of this capability, we had to dive deep into the PostgreSQL engine’s internals.

### Combining ALTER TABLE with index functions

PostgreSQL’s index feature allows for building complex and improved indexes, defined based on a function applied to one or more columns of a single table ([PostgreSQL documentation](https://www.postgresql.org/docs/current/indexes-expressional.html)).

The syntax for defining an index that is based on an index function is:

When the PostgreSQL INSERT/UPDATE/[ANALYZE](https://www.postgresql.org/docs/13/sql-analyze.html) commands are executed on a table with an index function, the function is called as part of the command.

Figure 5: index function invocation

During our research, we found that performing any of the above commands on a table implicitly invokes the index functions with the table owner’s permissions. This behavior is not detailed in the official PostgreSQL documentation, but references to it can be seen in the PostgreSQL source code:

### Exploitation

Combining these two primitives, we came up with the following attack flow:

1.  Create a new table.
    
2.  Insert some dummy content to the table, so the index function has something to work with.
    
3.  Create a malicious index function (with our code execution payload) on the table.
    
4.  ALTER the table owner to cloudsqladmin , GCP’s superuser role, used only by Cloud SQL to maintain and manage the database.
    
5.  ANALYZE the table, forcing the PostgreSQL engine to switch user-context to the table's owner ( cloudsqladmin ) and call the malicious index function with the cloudsqladmin permissions, resulting in executing our shell command, which we did not have permission to execute before.
    

In PostgreSQL, this flow looks something like this:

After executing the exploit SQL query, the `shell_commands_results` table contains the output of the executed code:

### Analyzing the modified PostgreSQL engine

After gaining code execution on the Cloud SQL managed PostgreSQL instance, we retrieved the modified PostgreSQL binary used by Cloud SQL and reverse engineered it. Reverse engineering the binary allowed us to gain a better understanding of the bug we discovered. Apparently, GCP modified the `ATExecChangeOwner` function to add a condition that checks whether the invoker user is part of the `cloudsqlsuperuser` role, the limited role that GCP provides customers. If so, the function treats it as a superuser and allows it to change the table’s owner to another user.

Perform ACL Checks only when invoker is not superuser or cloudsqlsuperuser

We can see the call to the `cloudsqlsuperuser()` function that was added by GCP to the PostgreSQL engine. If we are part of the `cloudsqlsuperuser` role and the owner of the table we want to change is not superuser, then we bypass the condition. This grants the ability to change table's owner to other users.

### Privilege escalation

At this point, we were executing code on the underlying compute instance running our managed PostgreSQL database. We were thrilled to map the internal environment, to identify new attack surfaces and find more vulnerabilities within the internal components of the service.

Figure 6: Executing the id command on the docker container hosting our instance

The Cloud SQL PostgreSQL database runs within a dedicated docker container as the `postgres` user. One thing that we immediately noticed is that the container shares the host virtual machine network namespace. This means that the container does not have its own dedicated IP address. Instead, the container binds directly to the host’s network interfaces. Having a shared network namespace with our host virtual machine allowed us to perform even more extensive network reconnaissance of the service’s internal network environment.

### Local privilege escalation to root

To gain a better understanding of the environment we were running in, we attempted to elevate our privileges to root. We searched the container for any writable files and directories, hoping to find a way to abuse writable files for privilege escalation.

This search showed us that our machine had a persistent directory, owned by our low-privileged user `postgres`.

Within that directory, we found the `iptables-save` file, a file owned by `root`. The file stores the iptables rules (local firewall rules) for our database instance.

Figure 7: A file owned by root in a directory under our control

Having a file owned by root in a directory under our control is something that immediately grabbed our attention. We imagined that if we managed to cause this file to be rewritten, we might be able to overwrite arbitrary files using a symlink attack. During previous research efforts, we enjoyed much success with these kinds of attacks, which leverage [symlinks](https://en.wikipedia.org/wiki/Symbolic_link) for an arbitrary file write. When the file is located under a directory owned by our user, we can delete the file and replace it with a symbolic link to the file we want to overwrite.

With this in mind, we next wondered how we could cause this file to be written to. Apparently, modifying our database instance network access rules in the Google Cloud SQL portal would result in new entries being appended to the `iptables-rules` file.

Figure 8: Adding new iptables entries

When we add new IP addresses in the service’s allow-list, the following line is added to the iptables-save file:

To recap: we had identified a file owned by root in a directory under our control, and we could make new entries appear in the file at will. It looked like we had everything needed to execute a symlinks attack and overwrite arbitrary files as root.

Say we wanted to overwrite our machine’s `/etc/passwd` file. What we would need to do is:

1\. Delete the `/psql/iptables-save` file.

2\. Create a symbolic link from `/psql/iptables-save` to `/etc/passwd:`

3\. Create a new crafted network entry for the database instance in Google Cloud SQL portal network.

The reason we needed to specify the `/host/var/lib/docker/overlay2/CONTAINER_HASH` is that we found that the write operation actually happens in a different docker container whose `/host` mount points to the host, so to overwrite our files, we had to specify the full path to our container.

There was just one last challenge we needed to overcome in order to leverage this file overwrite primitive for privilege escalation: finding the right file to overwrite. It was not easy to find a file to overwrite as we had only very limited control over what was being written to the file. Even if we managed to overwrite files like `/etc/passwd` and `/etc/shadow`, it would result in a parsing error and no real security consequences. After many attempts, we found some interesting behavior while overwriting the `ld.so.preload` file. For those who are not familiar with the file, it contains a whitespace-separated list of ELF shared objects to be loaded before every program. After overwriting this file with the content of our iptables-save (containing our line 1.1.1.1/32) and tracing binary executions, we observed that the binaries were attempting to load a shared library from the 1.1.1.1/32 path within our current directory! This gave us an idea: What if we place a shared library in that path? Then we would could execute a `suid` program like the mount binary, which comes as a `suid` binary in almost every Linux distribution. The library should run as root! This is exactly what we did.

We placed a symlink to `/etc/ld.so.preload` in our container:

Then we added the 1.1.1.1/32 IP to the allow list, which triggered the overwrite. Finally, we executed the mount program, which elevated our privileges to root. Here is a log from our execution:

### Container escape

Having gained CAP\_NET\_ADMIN and CAP\_NET\_RAW capabilities, we could escape the container. We leveraged the shared network namespace between the container and the host and used the same TCP injection technique described in [this article](https://www.ezequiel.tech/2020/08/dropping-shell-in.html), we automated the process by using the scapy Python library packaged with pyinstaller. The technique essentially exploits a feature in the GCP [guest agent](https://github.com/GoogleCloudPlatform/guest-agent) that runs on the host. Every so often, it queries the IMDS for a new configuration update. The guest agent sends certain parameters that cause the IMDS to respond after 60 seconds if there was no configuration change in that interval, allowing a window for injecting a fake configuration response to the guest agent from the IMDS. The configuration can contain an SSH key that should be added to the instance. The agent even creates the user supplied by the configuration if it is not found. With that all said, let’s see the script in action:

Figure 9: Escaping the container via TCP injection

After the script completed, we tried to log in with our newly created user:

Figure 10: Executing commands on the host virtual machine

As a bonus our user was a sudoer, granting us root privileges on the host.

Shortly thereafter, we received the following email:

It appears that our research triggered some alerts that caught the eyes of GCP’s security team. For the last year, we have been conducting similar research across most major cloud providers, but this is first time we were “caught.” Kudos to GCP’s security response team!

We promptly shared our findings in a detailed report, and began collaborating on a mitigation for these kinds of attacks.

Case study 2: Azure Database for PostgreSQL - Privilege Escalation and Code Execution
-------------------------------------------------------------------------------------

Like GCP, Microsoft Azure has introduced some modifications to the PostgreSQL engine in order to offer PostgreSQL-as-a-Service in the cloud. As with Cloud SQL, these modifications allow users certain superuser capabilities such as creating event triggers, creating checkpoints and loading extensions, all of which are usually reserved for superusers.

Apart from these superuser-like capabilities, Azure PostgreSQL grants users the `CREATEROLE` permission. This permission is not unique to Azure; other PostgreSQL-as-a-Service offerings (including Cloud SQL) also provide users with `CREATEROLE` permissions. But other offerings limit the role in one way or another to prevent its misuse. In Azure, there was no such limitation.

`CREATEROLE` is a very powerful permission. A user granted the `CREATEROLE` permission can create new users and associate them with specific roles. PostgreSQL comes with other powerful built-in roles such as: `pg_read_server_files`, `pg_write_server_files`, and `pg_execute_server_program`. Being a member of those roles could allow users to read/write files on the file system and even execute arbitrary commands on the underlying operating system.

The official [PostgreSQL documentation](https://www.postgresql.org/docs/current/sql-createrole.html) warns that users who are granted permission to arbitrarily create roles should be considered as “almost-superuser”:

> Be careful with the CREATEROLE privilege… \[I\]f a role does not have a certain privilege but is allowed to create other roles, it can easily create another role with different privileges than its own... Therefore, regard roles that have the CREATEROLE privilege as almost-superuser-roles.
> 
> PostgreSQL: Documentation: 14: CREATEROLE

The `CREATEROLE` permission is a fundamental and commonly used permission. And a managed PostgreSQL service is incomplete without this permission. Because of how powerful this permission is, almost all CSPs we assessed during our research introduced modifications to harden their PostgreSQL engine and disabled the option to grant powerful roles to regular users. However, Azure did not modify their `CREATEROLE` permission, allowing for the creation of powerful users with ability to execute arbitrary commands on the underlying virtual machine.

To exploit this unsafe behavior, we created a new user called “james” and associated it with all the powerful roles we could think of:

*   pg\_read\_server_files —gives the user the capability to arbitrarily read file from the file system
    
*   pg\_write\_server_files —gives the user the ability to arbitrarily write files to the file system
    
*   pg\_execute\_server_program —the most powerful role, giving a user a ability execute arbitrary command at the operating system level
    

After executing the query, we logged in as our newly created user “james” and used its permissions to execute arbitrary commands. For example, we executed the following command to invoke a reverse shell from the underlying virtual machine hosting our instance:

Obtaining a reverse shell from the underlying virtual machine gave us a foothold within the Azure internal network:

Figure 11: A reverse shell from Azure PostgreSQL Flexible instance

### Azure isolation

After gaining code execution on the managed PostgreSQL instance, we leveraged this initial foothold to prove unauthorized cross-tenant access to the databases belonging to other Azure PostgreSQL flexible customers. If you want to read more you can check out our dedicated [ExtraReplica blog post](https://www.wiz.io/blog/wiz-research-discovers-extrareplica-cross-account-database-vulnerability-in-azure-postgresql/).

Root cause analysis
-------------------

When they set out to offer PostgreSQL-as-a-Service, all CSPs had the same fundamental need: overcoming PostgreSQL’s inability to allow some superuser privileges to users without allowing everything else. Thus, all CSPs were forced to introduce modifications to the PostgreSQL project. The core problem is that introducing admin capabilities to low-privileged users is risky. A seemingly minor modification can allow users to bypass the security model and elevate their privileges within the local database. Because CSPs tend to introduce very similar capabilities, we were able to prove the same exploit code working on multiple vendors, even though the implementation code was different across different CSPs.

Disclosure process
------------------

After reporting the issues described above to the respective CSPs and realizing that these kinds of vulnerabilities are not unique to Azure and GCP, we initiated a broad responsible disclosure process with more than a dozen vendors offering managed PostgreSQL solutions. During that process, we learned that while all managed PostgreSQL vendors provide similar features and capabilities, each vendor has chosen a different way of granting customers those capabilities. Some utilize the PostgreSQL extensions mechanism. Others use special, complex configurations. Still others modify the PostgreSQL source code. Since the vulnerabilities vary between the unique modifications each vendor has implemented, we could not issue a CVE for these issues, which made the disclosure process more complex.

In addition, we initiated a private group with major CSPs to discuss the problematic pattern and possible courses of action to solve it for the entire industry. As part of the discussion in the group, GCP offered to contribute its PostgreSQL hardenings to the PostgreSQL project to allow the industry to stop implementing these changes individually and to allow everyone to consume this functionality from the original project that the community will maintain. That way, if a new vulnerability is discovered, it will be assigned a CVE and will be fixed in one central place from where everyone can update their software. Unfortunately, the PostgreSQL community was not enthusiastic about the idea, and it does not seem we will have such a solution soon. You are welcome to follow the conversation on the [PostgreSQL mailing list](https://www.postgresql.org/message-id/flat/CAMT0RQRX9j69GnrjJ2tkdy7zN_yofXUvU7iCOGdZE=jYiqX3wg@mail.gmail.com).

The managed databases company Aiven, with whom we worked closely on these issues, decided to release its hardenings as an open-source project. This is a good and effective approach since any new vulnerability affecting this project will receive a CVE and be fixed in one central place where users can be updated.

Summary
-------

During this journey of researching managed PostgreSQL across clouds and vendors, we learned that CSPs often provide popular and beloved open-source solutions as multi-tenant managed services. This is a significant power of the cloud - to offer anything as a scalable, managed service. However, these projects were not built with the needs of managed services in mind, so their adoption relies on multiple modifications and adjustments by each provider. We learned that while all CSPs want to offer their customers similar capabilities, they find different ways to achieve the same goal. As CSPS adapt non-cloud technologies to the cloud, new security issues are introduced.

### Isolation is king

We view this research as a great case study for the importance of cloud isolation. In the case of Cloud SQL, where tenant-isolation measures are strictly enforced, the impact of the vulnerabilities we found was relatively small. On the other hand, in the case of Azure PostgreSQL, where the isolation of the service was less strict, these kinds of vulnerabilities could have been leveraged to gain unauthorized cross-tenant access to the database instances belonging to other customers.

This case, where the same kind of vulnerability had completely different outcomes, illustrates how a service with strictly enforced isolation is more resilient to malicious actors, even if they are equipped with zero-day vulnerabilities.

Stay in touch!
--------------

Hi there! We are Ronen Shustin ([@ronenshh](https://twitter.com/ronenshh)) Shir Tamari ([@shirtamari](https://twitter.com/shirtamari)), Nir Ohfeld ([@nirohfeld](https://twitter.com/nirohfeld)) and Sagi Tzadik ([@sagitz_](https://twitter.com/sagitz_)) from the Wiz Research Team ([@wiz_io](https://twitter.com/wiz_io)). We are a group of veteran white hat hackers with a single goal:  make the cloud a safer place for everyone. We primarily focus on finding new attack vectors in the cloud and uncovering isolation issues in cloud vendors. We would love to hear from you! Feel free to contact us on Twitter or via email: [research@wiz.io](mailto:research@wiz.io).