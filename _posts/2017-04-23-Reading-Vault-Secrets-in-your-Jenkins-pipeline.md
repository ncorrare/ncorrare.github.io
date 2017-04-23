---
layout: post
title: Reading Vault Secrets in your Jenkins pipeline
categories: [general, vault, security, ci]
tags: [vault, jenkins]
fullview: true
---
The Jenkins credential store in most enterprises is becoming a potential attack vector. It's generally filled with long lived credentials, sometimes even to production systems.

In comes Hashicorp's Vault, a Secret Management solution that enables the secure store of secrets, and dynamic generation of credentials for your job. Looks like a great match right? Look at the demo, certainly looks promising (specially with Jenkins beautiful new BlueOcean UI):

![Jenkins Demo](/assets/media/0OCENniTf9.gif "Jenkins Pipeline consuming secrets from Vault")

Interested? Let's dive into it:

### What is Hashicorp Vault?
Quite simply, is a tool for managing secrets. What's really innovative about Vault is that it has methods for establishing both user and machine identity (through Auth Backends), so secrets can be consumed programatically. Identity is ultimately established by a (short lived) token. It also generates dynamic secrets on a number of backends, such as Cassandra, MySQL, PostgreSQL, SQL Server, MongoDB, etc. ... I'm not going to cover here how to configure Vault, feel free to head out to the [Vault documentation](https://www.vaultproject.io/docs/index.html). For all intent and purposes of this document, you can [download Vault](https://www.vaultproject.io/downloads.html) and run it in "dev mode" for a quick test:

{% highlight bash %}
$ vault server -dev
WARNING: Dev mode is enabled!

In this mode, Vault is completely in-memory and unsealed.
Vault is configured to only have a single unseal key. The root
token has already been authenticated with the CLI, so you can
immediately begin using the Vault CLI.

The only step you need to take is to set the following
environment variable since Vault will be talking without TLS:

    export VAULT_ADDR='http://127.0.0.1:8200'

The unseal key and root token are reproduced below in case you
want to seal/unseal the Vault or play with authentication.

Unseal Key: 2252546b1a8551e8411502501719c4b3
Root Token: 79bd8011-af5a-f147-557e-c58be4fedf6c

==> Vault server configuration:

         Log Level: info
           Backend: inmem
        Listener 1: tcp (addr: "127.0.0.1:8200", tls: "disabled")
{% endhighlight %}

Please don't run a Vault cluster in dev mode on production, feel free to reachout through the usual means if you need help.

#### AppRole
AppRole is a secure introduction method to establish machine identity. In AppRole, in order for the application to get a token, it would need to login using a Role ID (which is static, and associated with a policy), and a Secret ID (which is dynamic, one time use, and can only be requested by a previously authenticated user/system. In this case, we have two options:
- Store the Role IDs in Jenkins
- Store the Role ID in the Jenkinsfile of each project

So let's generate a policy for this role:
{% highlight bash %}
$ echo 'path "secret/hello" {
  capabilities = ["read", "list"]
}' | vault policy-write java-example -
Policy 'java-example' written.
{% endhighlight %}

In this case, tokens assigned to the java-example policy would have permission to read a secret on the secret/hello path.

Now we have to create a Role that will generate tokens associated with that policy, and retrieve the token:
{% highlight bash %}
$ vault write auth/approle/role/java-example \
> secret_id_ttl=60m \
> token_ttl=60m \
> token_max_tll=120m \
> policies="java-example"
Success! Data written to: auth/approle/role/java-example
$ vault read auth/approle/role/java-example
Key                 Value
---                 -----
bind_secret_id      true
bound_cidr_list
period              0
policies            [default java-example]
secret_id_num_uses  0
secret_id_ttl       3600
token_max_ttl       0
token_num_uses      0
token_ttl           3600
{% endhighlight %}
{% highlight bash %}
$ vault read auth/approle/role/java-example/role-id
Key     Value
---     -----
role_id 67bbcf2a-f7fb-3b41-f57e-88a34d9253e7
{% endhighlight %}
Note that in this case, the tokens generated through this policy have a time-to-live of 60 minutes. That means that after an hour, that token is expired and can't be used anymore. If you're Jenkins jobs are shorted, you can adjust that time to live now to increase security.

Let's write a secret that our application will consume:
{% highlight bash %}
$ vault write secret/hello value="You've Succesfully retrieved a secret from Hashicorp Vault"
Success! Data written to: secret/hello
{% endhighlight %}
Now Jenkins will need permissions to retrieve Secret IDs for our newly created role. Jenkins shouldn't be able to access the secret itself, list other Secret IDs, or even the Role ID.

{% highlight bash %}
$ echo 'path "auth/approle/role/java-example/secret-id" {
  capabilities = ["read","create","update"]
}' | vault policy-write jenkins -
{% endhighlight %}

And generate a token for Jenkins to login into Vault. This token should have a relatively large TTL, but will have to be rotated:
{% highlight bash %}
$ vault token-create -policy=jenkins
Key             Value
---             -----
token           de1fdee1-72c7-fdd0-aa48-a198eafeca10
token_accessor  8ccfb4bb-6d0a-d132-0f1d-5542139ec81c
token_duration  768h0m0s
token_renewable true
token_policies  [default jenkins]
{% endhighlight %}
In this way we're minimizing attack vectors:
- Jenkins only knows it's Vault Token (and potentially the Role ID) but doesn't know the Secret ID, which is generated at pipeline runtime and it's for one time use only.

- The Role ID can be stored in the Jenkinsfile. Without a token and a Secret ID has no use.

- The Secret ID is dynamic and one time use only, and only lives for a short period of time while it's requested and a login process is carried out to obtain a token for the role.

- The role token is short lived, and it will be useless once the pipeline finishes. It can even be revoked once you're finished with your pipeline.

### Jenkins pipeline and configuration

A full example for the project is available [https://github.com/ncorrare/vault-java-example](here). The Jenkinsfile will be using is this one:
{% highlight groovy %}
pipeline {
  agent any
  stages { 
    stage('Cleanup') {
      steps {
        withMaven(maven: 'maven-3.2.5') {
          sh 'mvn clean'
        }
        
      }
    }
    stage('Test') {
      steps {
        withMaven(maven: 'maven-3.2.5') {
          sh 'mvn test'
        }
        
      }
    }
    stage('Compile') {
      steps {
        withMaven(maven: 'maven-3.2.5') {
          sh 'mvn compile'
        }
        
      }
    }
    stage('Package') {
      steps {
        withMaven(maven: 'maven-3.2.5') {
          sh 'mvn package'
        }
        
      }
    }
    stage('Notify') {
      steps {
        echo 'Build Successful!'
      }
    }
    stage('Integration Tests') {
      steps {
      sh 'curl -o vault.zip https://releases.hashicorp.com/vault/0.7.0/vault_0.7.0_linux_arm.zip ; yes | unzip vault.zip'
        withCredentials([string(credentialsId: 'role', variable: 'ROLE_ID'),string(credentialsId: 'VAULTTOKEN', variable: 'VAULT_TOKEN')]) {
        sh '''
          set +x
          export VAULT_ADDR=https://$(hostname):8200
          export VAULT_SKIP_VERIFY=true
          export SECRET_ID=$(./vault write -field=secret_id -f auth/approle/role/java-example/secret-id)
          export VAULT_TOKEN=$(./vault write -field=token auth/approle/login role_id=${ROLE_ID} secret_id=${SECRET_ID})
          java -jar target/java-client-example-1.0-SNAPSHOT-jar-with-dependencies.jar 
        '''
        }
      }
    }
  }
  environment {
    mvnHome = 'maven-3.2.5'
  }
}
{% endhighlight %}

Lot's of stuff here, including certain Maven tasks, but we will be focusing on the Integration Tests stage (the last one), what we're doing here is:
- Downloading the Vault binary (in my case the ARM linux one, I'll brag about that in a different blog post :D)

- Reading credentials from the Jenkins credential store. In this case I'm storing both the ROLE_ID and the VAULT_TOKEN I've generated for Jenkins. As mentioned before if you want to split them for more security, you can just use the ROLE_ID as a variable in the Jenkinsfile.

- I'm doing a set +x to disable verbosity in the shell in order not to leak credentials, although even with -x, in this case I'm just showing the Secret ID (which is useless after I've already used it), and the VAULT_TOKEN that I'm going to use to consume credentials, which is short lived, and can be revoked at the end of this runtime just adding a `vault token-revoke` command.

- I'm retrieving a SECRET_ID using Jenkins administrative token (that I've manually generated before, that's the only one that would be relatively longed lived, but can only generate SECRET_IDs).

- I'm doing an AppRole login with the ROLE_ID and the SECRET_ID, and storing that token (short lived).

- My Java Process is reading VAULT_TOKEN and VAULT_ADDR to contact Vault and retrieve the secret we stored.

The VAULT_TOKEN (And optionally ROLE_ID) are stored in the Credential store in Jenkins:

![Jenkins Cred Store](/assets/media/jenkins-cred-store.png "Jenkins Credential Store")
