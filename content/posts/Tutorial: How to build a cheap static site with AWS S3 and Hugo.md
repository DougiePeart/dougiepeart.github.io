+++
title = "How to build a cheap static site with AWS S3 and Hugo"
author = ["Dougie Peart"]
date = 2022-08-19T19:56:00+01:00
tags = ["AWS", "linux", "hugo", "S3", "website"]
categories = ["tutorial"]
draft = false
+++

## Introduction {#introduction}

I've been learning AWS recently while working towards getting the [AWS Certified Solutions Architect - Associate](https://aws.amazon.com/certification/certified-solutions-architect-associate/) certification. Setting up a good personal website has been on my to-do list for a while so this seemed like a perfect opportunity to get some practical experience on the platform.

I looked at the pricing as I had first considered running an EC2 instance, but I then saw that Lightsail instances were a lot cheaper. They're essentially just like a VPS on any other cloud platform. I spun up an instance and selected the WordPress template maintained by Bitnami.

I had a website up and running really quickly, but on reflection, WordPress seemed a little bloated. I use Emacs as my editor of choice, so using [Hugo](https://gohugo.io/) with the [ox-hugo](https://ox-hugo.scripter.co/) package seemed a perfect fit.

I tried following a tutorial that involved pushing the code to CodeCommit, CodePipeline would pick up the change and then kick off a build with CodeBuild to run Hugo and sync the public folder to an S3 bucket. It was a cool [tutorial](https://conormclaughlin.net/2017/11/automating-deployment-of-your-hugo-site-to-s3-using-aws-codepipeline/) and quite a novel solution, but it failed a few times for me when running Hugo in the build job. I was looking through the Hugo documentation to see where I was going wrong when I noticed the "hugo deploy" command. This command just uploads the public directory to an S3 bucket for you!


## Setting up the S3 Bucket {#setting-up-the-s3-bucket}

Log into AWS and go into the S3 section then click "Create bucket"

{{< figure src="/ox-hugo/createbucket.png" >}}

Name the bucket after your website (it doesn't need to be the site name, it can say anything) and select an AWS region, I'm going for eu-west-2 because it's close to me.

{{< figure src="/ox-hugo/namebucket.png" >}}

Untick the "Block all public access" box and acknowledge the warning at the bottom of that section. See the image above for all the settings I've used. Once you're happy, scroll to the bottom and select "Create bucket"

{{< figure src="/ox-hugo/selectbucket.png" >}}

Once created, click into the bucket and then click into the properties tab.

{{< figure src="/ox-hugo/selectproperties.png" >}}

Scroll down to the bottom and select Edit on Static website hosting

{{< figure src="/ox-hugo/selectstatichosting.png" >}}

Enable static website hosting and enter the page file names index.html and error.html respectively. when you're done save the changes.

{{< figure src="/ox-hugo/editstatichosting.png" >}}


## Set up a user in IAM {#set-up-a-user-in-iam}

Open the IAM dashboard and select users from the left-hand side. Select Add Users and add a user called "hugo" with access key as the credential type.

{{< figure src="/ox-hugo/addhugouser.png" >}}

Go to the next page and select Create Group. Create a group called "hugo", search for s3 in the policies and check the box for AmazonS3FullAccess, then select Create Group. Next click into Tags, add any tags you like, click into Review and when you're happy, create the user.

{{< figure src="/ox-hugo/addhugogroup.png" >}}

On the final page, you'll be given the Access Key ID and the Secret Access Key

{{< figure src="/ox-hugo/hugoaccesskeys2.png" >}}

Take a note of these for later and don't share them with anyone:

```nil
Access Key ID:
AKIA4QJB6VGADXAFZ3RZ

Secret Access Key:
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
```


## Installing the AWS CLI {#installing-the-aws-cli}

Curl the installer file:

```nil
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"\
unzip awscliv2.zip\
```

Run the installer:

```nil
sudo ./aws/install
```

Then configure the CLI with your credentials from earlier

{{< figure src="/ox-hugo/awsconfigure2.png" >}}

You should now be able to interact with your AWS subscription. Try running "aws s3 ls" and you should see the bucket you've created.

```nil
[username@hostname]$ aws s3 ls
2022-08-19 21:36:22 dougiepeart.co.uk
```

You've now got the AWS CLI set up, now all that's left is to configure Hugo to push to your bucket (and configure your Hugo site, but that's beyond the scope of this tutorial).


## Configuring Hugo {#configuring-hugo}

Inside your config.toml file for your Hugo site append the following:

```nil
[deployment]
  [[deployment.targets]]
    name = "aws-s3"
    URL= "s3://dougiepeart.co.uk?region=eu-west-2"

  [[deployment.matchers]]
    # Cache static assets for 1 year.
    pattern = "^.+\\.(js|css|svg|ttf)$"
    cacheControl = "max-age=31536000, no-transform, public"
    gzip = true

  [[deployment.matchers]]
    pattern = "^.+\\.(png|jpg)$"
    cacheControl = "max-age=31536000, no-transform, public"
    gzip = false

  [[deployment.matchers]]
    # Set custom content type for /sitemap.xml
    pattern = "^sitemap\\.xml$"
    contentType = "application/xml"
    gzip = true

  [[deployment.matchers]]
    pattern = "^.+\\.(html|xml|json)$"
    gzip = true
```

Obviously, change the S3 URL to your bucket name and selected region though.

Congratulations you've got everything you need to push your site from your machine to S3!

cd into your websites directory and run:

```nil
~/docs/repos/dougiepeart.co.uk $ hugo && hugo deploy
Start building sites …
hugo v0.101.0+extended linux/amd64 BuildDate=unknown

                   | KO | EN
-------------------+----+-----
  Pages            | 13 | 49
  Paginator pages  |  0 |  0
  Non-page files   |  0 |  0
  Static files     | 80 | 80
  Processed images |  0 |  0
  Aliases          |  2 | 10
  Sitemaps         |  2 |  1
  Cleaned          |  0 |  0

Total in 5708 ms
Deploying to target "aws-s3" (s3://dougiepeart.co.uk?region=eu-west-2)
Identified 22 file(s) to upload, totaling 290 kB, and 0 file(s) to delete.
Success!
Success!
```

You've now got your site in a publicly accessible S3 bucket :)

You can find the URL in the bucket's properties under the Static website section

But you have to navigate to a stupid URL to see it :(


## Configuring your DNS record. {#configuring-your-dns-record-dot}

So, log into Cloudflare (assuming you've set [them up as your DNS provider](https://developers.cloudflare.com/dns/zone-setups/full-setup/setup/)) and go into the DNS section.

Ordinarily, you'd set an A record here pointing to an IP address, but as I'm sure you've noticed, AWS didn't allocate us one. Not to worry though, you don't need an A record, you can just use a CNAME.

{{< figure src="/ox-hugo/cname_000.png" >}}

You can just use @ for the name here, but I've set it as dougiepeart.co.uk for clarity.

Paste in your S3 URL and ensure it is set to proxied as you'll be using this for your SSL certificate.

You should now be able to browse to your website and see your Hugo site :)


## Adding a Certificate {#adding-a-certificate}

The final step. In Cloudflare, go into the SSL/TLS section on the left-hand side. In the overview, set it to Flexible.

{{< figure src="/ox-hugo/sslflexible.png" >}}

If you reload your website, you should now have a secure connection.

That's it. You now have super cheap and fast hosting for your static website.

If you have any questions, feel free to email [me](mailto:contact@dougiepeart.co.uk).
