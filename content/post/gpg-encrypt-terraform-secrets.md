+++
banner = "images/code.webp"
date = "2020-03-14T13:47:08+02:00"
tags = ["Encryption", "GPG", "PGP"]
title = "How to Use PGP to Encrypt Your Terraform Secrets"
short_description = "Protect your infrastructure secrets using strong encryption."
+++

If like me, you also use the terraform binary from your computer to describe and deploy the infrastructure of your projects, you might want to add an extra level of security. This extra level of security that I am referring to is encrypting your terraform secrets, both on-screen and in the terraform state files.

Of course, the method I am going to describe in this article can be used for production-grade environments. However, you might want to tweak it a little to suit your needs - like using PGP Keys protected by virtual or physical MFA devices, or using PGP Keys belonging to pipeline services as opposed to individual users.

First of all, the requirements. I am going to use GPG (the GNU implementation of PGP) and terraform. The details of the versions I am using are featured below.

My GPG version is:

```bash
$ gpg --version
gpg (GnuPG) 2.1.18
libgcrypt 1.7.6-beta
(...)
```

And my terraform version is:

```bash
$ terraform --version
Terraform v0.12.24
```

## Creating Your Encryption Key 

If you have not got a key, you need one. If you are not sure whether you have one or not, you can list your existing keys by running the following command:

```bash
$ gpg -k
```

However, if you were not sure in the first place, chances are you do not remember the passphrase to unlock them; so you might as well create a new one.

Creating a key using GPG is very, very easy. The tool has evolved to the point that as of October 2014, the [default is a 2048 bit RSA keypair](https://www.gnupg.org/faq/gnupg-faq.html#default_rsa2048).

To get a key, you only need to run the `gpg --gen-key` command and follow the prompts that the tool gives you. Go ahead and experiment with it. Since I like to automate things and turn them into code, I am going to use the unattended key generation option.

To use the unattended key generation option that GPG provides, we first need to create a file that describes what options we want our key to feature.

For my key, I created the following file I called `key-gen-template`:

```
%echo Generating a default key
Key-Type: default
Subkey-Type: default
Name-Real: Micky Menendez
Name-Comment: How to Use PGP to Encrypt Your Terraform Secrets
Name-Email: article@menendezjaume.com
Expire-Date: 2
# Passphrase: ABC
%commit
%echo done
```

Note that I have not used the set passphrase option. Instead, I want GPG to prompt me so I can add it manually. I left the argument in the demo file though, in case you wanted to use it and were wondering if there was a way to make the key generation fully unattended. Perhaps, in your pipeline, you have a way to inject secrets in-flight, and therefore you want to reference one of those secrets as your passphrase.

Also, note that I have set my expiry date to two days from now - in case I forgot to delete it! But you could set it to anything that aligned with your security appetite to even 0, which means that it does not expire.

Once I run the following command, I am prompted to introduce the passphrase to unlock the key (as mentioned, only because it was not included in the template file), and get the following result:

```bash
$ gpg --batch --gen-key key-gen-template 
gpg: Generating a default key
gpg: key 89592232B763C48A marked as ultimately trusted
gpg: revocation certificate stored as '/home/$(whoami)/.gnupg/openpgp-revocs.d/07D9DF35A7DDE76C5212415289592232B763C48A.rev'
gpg: done
```

Success! Now, we are in a position to use our newly created key to sign, encrypt, and decrypt messages. 

## Using Your PGP Key in Terraform

Let's assume that, using terraform, we want to create a user and output securely its Access Key ID and its Secret Access Key. Here is an example terraform script that we could use for that:

```terraform
resource "aws_iam_user" "user" {
  name = "demo_user"
  path = "/"
}

resource "aws_iam_access_key" "user_access_key" {
  user = aws_iam_user.user.name
  pgp_key = "your-key"
}

output "access_key_id" {
  value = aws_iam_access_key.user_access_key.id
}

output "secret_access_key" {
  value = aws_iam_access_key.user_access_key.encrypted_secret
}
```

Note the `pgp_key` field in the  `aws_iam_access_key` resource. What is interesting about this field and, in my opinion, not very well documented, is that instead of using what I always thought was the traditional format of a public key, it expects only the base64-encoded body of the key. 

 To get the traditional and therefore armour-surrounded key, we need to run something like `$ gpg --armor --output public-key.gpg --export article@menendezjaume.com`; using, of course, the identity you used to create your key.

```
-----BEGIN PGP PUBLIC KEY BLOCK-----
base64-encoded-body-of-the-key
-----END PGP PUBLIC KEY BLOCK-----
```

Next, we have three options: one manual and two automated ones.

First, we can manually edit and delete the header and footer and use the body of the key as input for our `pgp_key` argument.

Second, we can output the key in its binary format by running something like `$ gpg --output public-key-binary.gpg --export article@menendezjaume.com` and use the `local` terraform provider to read the base64-encoded content of the file.

Third, we can output the key in its binary format, pipe it to base64 encode it and save it by running something like `$ gpg --export article@menendezjaume.com | base64 > public-key-base64-encoded.gpg`, also using the `local` terraform provider, this time to read the raw content of the file.

In this case, I am using the second option, and have to update the terraform script with the following:

```terraform
data "local_file" "pgp_key" {
  filename = "/path/to/your/public-key-binary.gpg"
}

(...)

resource "aws_iam_access_key" "user_access_key" {
  user = aws_iam_user.user.name
  pgp_key = data.local_file.pgp_key.content_base64
}

(...)
```

Note the reference to `content_base64`. If we wanted to use the third option, we would instead use `content`.

Since we are using the provider `local` that was not previously installed in this example, we need to run `$ terraform init` before being able to use it.

At this point, we are in a position where we can `plan` and subsequently `apply` our infrastructure.

```
$ terraform apply -auto-approve
data.local_file.pgp_key: Refreshing state...
aws_iam_user.user: Creating...
aws_iam_user.user: Creation complete after 1s [id=demo_user]
aws_iam_access_key.user_access_key: Creating...
aws_iam_access_key.user_access_key: Creation complete after 1s [id=AKIAWIAZC4TF6BOAAX7B]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

Outputs:

access_key_id = AKIAWIAZC4TF6BOAAX7B
secret_access_key = wcDMA4SJlErDzE23AQwAsv/BznfdZFvFt6cPxo3I4Wc+fZhN+3BXiLtQSmfi6Wt8X10zYQIs52OFKLmX7fuUPy0BPhl+Wl7QU8T9k8wUNaqlctmJDklacQnHKIXvsZIzDSHNPBrNlo6mZp221ZlL1qEXfcRdpOcuRZJ0diJ59+sycizEUC7ROlxfQlFdoBncXPD88yINRkN52MgxNYVIFxDJyPnlJAi3BWVro+/AyhBNxi11XdVTjkNJOw8/zJBn7AyxBB8GnzRMNjQQC7vV37ic8gTx8z4dZfGuWLl6pP7vEf+RUNRnLnwoKPAYSmomI6MaOKU056cazA4S9jw2n9uaw7QVNt8uJ+PSc9AdcKLlm97cS8sQV26D2ONIKk+XRlVnTTky4R0T3PtqpmBmBBvnFCRilct6OTOFfqyD+Pdm9zHUkfM1nkqxz3cYa1UsAnZOWA2ok2U+hM5anYfib5eN3zlWeBVblJdH9z0s1HDAHtD/+HgBDocPbeuLfXI85PYu2T9DQ+8MzCCn/XMz0uAB5HCo8qbv3+n7k/nWhHKFcFThzBHgreCX4Q3z4EXiffHUfeCw5bsoJR4i0/f9OjQ/78zAhiXw00RwSJSxLT4KL9o2hf7x4HfjXSVo2e7eTsLgkOSjwOXFl/pMvXOZq8OEWL7N4lTZay/hrMAA
```

Now, we can note down our `access_key_id`, or query the terraform state in the future by using `$ terraform output access_key_id` to get it. In terms of the `secret_access_key`, before being able to use it, we need to decrypt it.

The long way to decrypt the `secret_access_key` would be to copy the contents into a file and to read the file with GPG to decrypt it. We are not going to do the long way. Instead, we want a one-liner that can be used by ourselves or by a pipeline service to get the decrypted secret and potentially inject it in a Secrets Management tool.

One of the problems that we face now is that GPG gets "confused" when we pipe things into it, so before doing so, we need to tell it that we are going to pipe an encrypted message into it:

```
$ export GPG_TTY=$(tty)
```

By correctly setting the environment variable GPG_TTY as per above, GPG clearly understands that we are going to pipe a secret into it.

Now, if we query terraform for the `secret_access_key`, base64 decode it and decrypt it using GPG, we get the following:

```bash
$ terraform output secret_access_key | base64 --decode | gpg --decrypt
gpg: encrypted with 3072-bit RSA key, ID 8489944AC3CC4DB7, created 2020-04-05
      "Micky Menendez (How to Use PGP to Encrypt Your Terraform Secrets) <article@menendezjaume.com>"
arI8ugTYGImzlojr022gc55L8cPqAdweUNiiswz3
```

Which gives us the raw `secret_access_key`! And of course, GPG prompted me for the passphrase of the key to use it.

By encrypting out terraform secrets, we are not only protecting them when they are output by terraform on screen, but also when they are written to our terraform state files. This is crucial because should our state file go missing, an attacker would not be able to use any of the credentials to compromise our infrastructure.

Do you encrypt your terraform secrets? Or do you simply enforce rotation?