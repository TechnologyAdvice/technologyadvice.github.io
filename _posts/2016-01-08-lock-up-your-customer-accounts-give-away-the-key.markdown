---
layout: post
title: "Lock Up Your Customer Accounts, Give Away the Key"
date: 2016-01-08 18:00:00
categories: security
author: tom.shawver
cover: /assets/images/covers/usb-cryptex.png
description: With so much focus on secure user passwords recently, there's surprisingly little about secure database and service credentials. Encrypting your credentials against everyone from your CI to your engineers can be simple and quick to implement.
comments: true
disqus: true
---

In your web application, how secure do you require your users' credentials to be? Do you require passwords to be a certain length, with enough kinds of different characters? Maybe you're more progressive and employ two-factor auth, or alternative auth like biometrics or single-use SMS keys. If you do use passwords, maybe you're using a compute or space-expensive hashing algorithm like bcrypt, scrypt, PBKDF2, or Argon2 to keep them safe. Maybe you're encrypting your entire database on disk to keep your users as safe as possible. Nothing short of an attacker compromising your live database could leak that data out.

Quick sanity check: right now, where can I find the secret used by your production application to authenticate to your database?

- Is it committed and pushed up to your remote repo, like Github?
- Is it stored on a file server, able to be accessed if I compromise some other password, or maybe an IP address?
- Has it been shared in plain text over email? Chat?
- Does your continuous integration and/or deployment service have it, maybe saved in an environment variable? Is getting your secrets as easy as compromising one of their engineers?
- Can any of _your_ engineers get at it for a well-crafted bribe?

If any of those things are true, you're boned. You've locked all the windows and left the front door wide open. If you've not been compromised yet, you simply haven't been a valuable enough [target] yet. When will that happen? Will you know?

## The expensive solution

If you have the budget for launching a server cluster to house and safeguard your secrets, the options there are becoming more plentiful. [Vault] by Hashicorp, Square's [KeyWhiz], and Lyft's [Confidant] have all gotten recent praise, atop an ocean of other capable but less press-generating options.

The security benefits here are fantastic: Not only are your credentials stored encrypted, the central location allows for passwords to be easily rotated. Once your service securely authenticates with one of these server solutions, it gains access to any credentials for which this particular service has been granted access.

The downsides, though, put this out of reach for smaller teams:

- Running an additional server cluster in addition to the rest of your app is a significant expense, from both the hardware and maintenance team standpoint
- It introduces a single point of failure to your application: should that cluster become inaccessible, your entire stack is offline
- Integrating this after an app's already been written can be difficult, as each password retrieval now requires a round trip TCP connection
- Authenticating with the server often requires another set of credentials. How do you store those safely? It's [turtles all the way down].
- The solution is big, requiring multiple people maybe from multiple teams, buy-in from all key players, meetings, time, and a big rollout that takes attention away from core competencies. Wouldn't it be nice to roll out a solution more simply, at a natural pace?

## Envelope encryption

Let's strike the idea of an independent server-based solution for now, which immediately clears up the single point of failure, additional infrastructure costs, and the heavy rollout. Instead, we'll entertain a simpler technique: envelope encryption.

The idea behind envelope encryption is easy: Each of our services will choose a single unique key for a secure and widely-trusted symmetric encryption algorithm. A symmetric encryption algorithm means that the same key can be used both to encrypt and decrypt, and one of the most well-trusted standards for this is AES. We'll use this AES key to encrypt all the passwords our service needs to know about, and save them to a file.  Now, using a _different_ key that we'll access elsewhere, we'll encrypt that AES key and save it to the same file.

Here's what we've just achieved:

- All the secrets this service requires and the key to access them are now safely encrypted and can be sent anywhere -- saved in github, shared with a third party CI/CD service, hosted on shared infrastructure, etc.
- Since all those secrets can be decrypted by one AES key that's held locally, there's only one point of I/O here after this file is read: doing whatever is necessary to decrypt that AES key.
- At no point are any of our secrets held in plain text anywhere other than RAM, and that RAM can be cleared once the secret is no longer needed. It can be decrypted again as-needed.

But we've made a few assumptions above that are hard to gloss over: Where is this mysterious key that we encrypt the AES key with? How do we authenticate to do that? Encryption is hard; how do I manage it without making mistakes?

## Enter Cryptex

[Cryptex] is a Node.js library and CLI tool to effortlessly manage secure envelope encryption in your services. Use it in any kind of project by simply firing off the executable and capturing the output. Node.js projects gain additional simplicity from the module integrating directly with the app and providing a clear and simple API.

Cryptex offers a huge range of configuration options (even multiple ways to _specify_ those configuration options) so that it can plug seamlessly into any app, but for the purposes of this walkthrough I'm going to make the following decisions:

- We'll specify all our config in a `cryptex.json` file in the root of our project rather than configure it using environment variables, so that we can specify the config for all our environments
- Of the multiple ways Cryptex can handle encryption of our AES key, we'll use Amazon's Key Management Service (KMS) so we don't have any burden of managing security ourselves. This is a hardware device that prevents access to the master encryption key by any means, behind the locked doors and high physical security of Amazon's data centers. It's free to use at most volumes and pennies (literally) for super-high volume, and you can use it even if you're not using AWS for any other part of your infrastructure.
- Our local key will be an AES-256 key
- We'll save our encrypted secrets in base64 format

With that nailed down, let's tackle this!

### Generating the AES key

First, we need to create the AES key that we'll be using to encrypt all of our secrets. To do that, start by logging into the AWS console, finding the IAM section, and choosing "Encryption Keys".

Create the master key, giving it an alias and defining who is and isn't allowed to encrypt and decrypt using it. Assuming you set up an IAM user for each of your engineers, this is the magic that makes it possible to bar all but a select few devops from altering production secrets. To that end, I recommend creating a minimum of two keys: One for production with strict permissions, and one or more for development/staging/testing environments as necessary. You can come back here and create more accounts/change permissions later.

Now to create the AES key, you'll want the [AWS CLI tool], so make sure that's installed and you've configured your AWS keys locally. Then just fire off this command (replace `YOUR_KEY_ALIAS`):

```
aws kms generate-data-key-without-plaintext \
  --key-id alias/YOUR_KEY_ALIAS \
  --key-spec AES_256 \
  --output text \
  --query CiphertextBlob
```

Now you have your brand new AES key, encrypted via KMS, safe to share over insecure media.

### Creating your cryptex.json

Save the following blob in a file named `cryptex.json`, inserting your encrypted key from the last step in place of `KEY_HERE`, and your AWS region in place of `us-east-1` if that's not what you're using:

```javascript
{
  "default": {
    "keySource": "kms",
    "keySourceOpts": {
      "region": "us-east-1",
      "dataKey": "KEY_HERE"
    },
    "secrets": {
    }
  }
}
```

Note that `default` is used if no other environment is specified. You can add other environments to this file (name them `production`, `prod`, `test`, whatever you like!) and give them similar config.

### Encrypting your secrets

If you haven't already, install [Node.js] and use its package manager to install the Cryptex CLI tool:

```
npm install -g cryptex
```

Now, from the same folder as your `cryptex.json` file, encrypt any of your secrets:
 
{% highlight bash %}
cryptex encrypt SomeSecretPassword
# Output: 14h/K43TNhsp/slpikmlGj3zMOnCdPJLUe5AphTA3k7PW169m05K1vt880IXd6bL
{% endhighlight %}

Any encrypted secrets you'd like to keep, just throw them in your `cryptex.json` with a friendly name:

```javascript
{
  "default": {
    "keySource": "kms",
    "keySourceOpts": {
      "region": "us-east-1",
      "dataKey": "KEY_HERE"
    },
    "secrets": {
      "mysqlpass": "14h/K43TNhsp/slpikmlGj3zMOnCdPJLUe5AphTA3k7PW169m05K1vt880IXd6bL"
    }
  }
}
```

Repeat for any secrets you need to store, for each environment you have set up. Just pass `-e environment-name-here` to the `cryptex` command to switch environments.

> Note: It's also possible to store secrets in plaintext for an environment to ease the local development flow, while keeping everything safely encrypted for other environments. See the [Cryptex Documentation] to set that up.

### Decrypting secrets on the command line

Once your `cryptex.json` has some named secrets in it, decrypting them is as simple as:

```
cryptex decrypt mysqlpass
# Output: SomeSecretPassword
```

As with encrypting, pass `-e environment-name-here` to switch out of the default environment.

This command is now all you need to integrate encrypted secrets into any app!

### Decrypting secrets in Node.js

For users of Node, the Cryptex module can be installed into a project with:

```
npm install --save cryptex
```

And you can get your secrets programmatically like this, using the environment in the `NODE_ENV` environment variable, or `default` if that's not found:

```javascript
var cryptex = require('cryptex');

cryptex.getSecret('mysqlpass').then(function(password) {
  // password contains the decrypted secret
})
```

Is your existing app not set up to handle an asynchronous call before the password is needed? No problem: Use a loader.  Add a JS file to your app that looks something like this:

```javascript
var cryptex = require('cryptex');

cryptex.getSecrets([
  'mysqlpass',
  'mongodbpass',
  'apitoken'
]).then(function(secrets) {
  process.env.MYSQLPASS = secrets.mysqlpass;
  process.env.MONOGDBPASS = secrets.mongodbpass;
  process.env.APITOKEN = secrets.apitoken;
  require('./my-app.js');
}).catch(function(err) {
  console.log('Failed getting secrets', err.stack);
});
```

Just change `my-app.js` to the original entry point to your application, and now you're guaranteed to have all your passwords saved in environment variables the moment your app starts up.
 
If you can help it, though, it's better to request your secrets as-needed. That way, the decrypted versions leave RAM as soon as they're no longer being used, providing that extra bit of protection against memory-dumping malware. Cryptex also respects this ethic, removing your decrypted AES key from RAM after a short window.

### Safe AWS authentication for KMS

Now you're all set, but you have that pesky AWS secret access key that you need to communicate with KMS. It's turtles all the way down again! How do we get past it? We have three great options:

#### IAM roles

If your app is hosted on AWS infrastructure, you're in the clear. EC2 nodes and Lambda functions can be associated with IAM roles -- a set of AWS permissions that API calls originating from that machine use by default. Set up an IAM role that can use KMS's decrypt function with your production master key, associate that role with your production resources, and boom -- now you don't have to store a secret key or authenticate at all. It all just works. Even if you're using Amazon ECS to manage Docker containers!

#### AWS Cognito

AWS has an authentication service geared toward mobile developers (but still quite useful for web applications) called Cognito. Cognito allows you to authenticate with any of the major authentication providers out of the box, or _even hook into your own, home-rolled authentication solution_. Once Cognito deems you authenticated, it can provide you with a set of temporary AWS access credentials with the permissions you need. So however you're handling service authentication now can be baked right in. If you're _not_ handling it in any way right now, there's one last option...

#### Add it during provisioning

This is the least secure-out-of-the-box solution, so don't set this up without a good security engineer on hand. However, when a new server is provisioned, you can have your configuration tool -- Ansible, Chef, Puppet, Salt, etc. -- copy the credential to the box, as long as that tool is running locally and you're not handing that credential off in plaintext to some third party CD solution. This is not recommended, because any attacker or malware scanning the box can find this unencrypted secret and use it against you. However, they cannot do that without generating an event in AWS's thorough CloudTrail audit logs, so frequent (and preferably automated) review of those logs combined with the understanding that a compromised machine assures compromised secrets is still a better solution than plaintext secrets committed to git.
  
## Wrapping up

Cryptex is capable of more than I touched on here, so reading through the [Cryptex Documentation] is encouraged. In particular, if the complexities of Amazon KMS are a turn-off, other key sources are available -- and the consistent API makes it easy to plug in other custom providers.

But if you've gotten this far and I still haven't convinced you how important it is to protect your customers' private data with at least the same level of security you require of their own accounts, let me pose a different question: What do you think your customers would prefer?

_(Cover image sourced from this [usb cryptex])_

[target]: http://www.cio.com/article/2600345/security0/11-steps-attackers-took-to-crack-target.html
[Vault]: https://vaultproject.io/
[KeyWhiz]: https://square.github.io/keywhiz/
[Confidant]: https://lyft.github.io/confidant/
[turtles all the way down]: https://en.wikipedia.org/wiki/Turtles_all_the_way_down
[Cryptex]: https://github.com/TechnologyAdvice/Cryptex
[Cryptex Documentation]: https://github.com/TechnologyAdvice/Cryptex/blob/master/README.md
[AWS CLI tool]: http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-set-up.html
[Node.js]: http://nodejs.org
[usb cryptex]: http://www.amazon.com/Special-Edition-Cryptex-Micro-Adapter/dp/B01133M3GQ
