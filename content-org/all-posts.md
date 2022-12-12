
# Table of Contents

1.  [How To Build A Cheap Static Website With AWS S3](#orgf8b937f):AWS:linux:hugo:S3:website:@tutorial:
    1.  [Introduction](#org9428a4f)
    2.  [Setting up the S3 Bucket](#org4a8806c)
    3.  [Set up a user in IAM](#org7224940)
    4.  [Installing the AWS CLI](#org40d9ae0)
    5.  [Configuring Hugo](#org0341bc7)
    6.  [Configuring your DNS record.](#orgf7fa5c6)
    7.  [Adding a Certificate](#org2487c85)
2.  [Book Notes: The Beginner&rsquo;s guide to Stoicism](#orgb592573):book:notes:stoicism:philosophy:reading:@reading:
    1.  [Source](#orgaa02912)
    2.  [The Book in 3 Sentences](#org365b50e)
    3.  [Impressions](#org4a660e8)
    4.  [Who Should Read it?](#orgf72b5db)
    5.  [How the Book Changed Me](#orgd3ee9df)
    6.  [My Top 3 Quotes](#orgeca2cf6)
    7.  [Summary + Notes](#org76ff394)
        1.  [Myths and Misconceptions](#orgbe53ad5)
        2.  [Living In Accordance With Nature](#orgeac6be2)
        3.  [The Three Stoic Disciplines](#orgb185339)
        4.  [The Four Stoic Virtues](#org52a1958)
        5.  [Rules To Live By - Page 19](#org7cb0c85)
        6.  [A Pythagorean Practice - Page 32](#orgb4a0599)
        7.  [Socrates&rsquo;s Views Adopted Into Stoicism - Page 33](#orgc9dfb3e)
        8.  [Essential Tools](#org21be024)
        9.  [Seneca&rsquo;s Description of Stoicism](#org0fb98f3)
        10. [What DO You Control?](#org544e2be)
        11. [Before You Sleep](#orgb023770)
        12. [The Stoic Passions](#orgd1bcc33)
        13. [What Is So Unbearable About This Moment?](#orgbaf8d6d)
        14. [It Seemed So to Them](#orgd5608f6)
        15. [Creating Change](#orgbf1d881)
        16. [Establish A Daily Routine](#org6efa79c)



<a id="orgf8b937f"></a>

# DONE How To Build A Cheap Static Website With AWS S3     :AWS:linux:hugo:S3:website:@tutorial:


<a id="org9428a4f"></a>

## Introduction

I&rsquo;ve been learning AWS recently while working towards getting the [AWS Certified Solutions Architect - Associate](https://aws.amazon.com/certification/certified-solutions-architect-associate/) certification. Setting up a good personal website has been on my to-do list for a while so this seemed like a perfect opportunity to get some practical experience on the platform.

I looked at the pricing as I had first considered running an EC2 instance, but I then saw that Lightsail instances were a lot cheaper. They&rsquo;re essentially just like a VPS on any other cloud platform. I spun up an instance and selected the WordPress template maintained by Bitnami.

I had a website up and running really quickly, but on reflection, WordPress seemed a little bloated. I use Emacs as my editor of choice, so using [Hugo](https://gohugo.io/) with the [ox-hugo](https://ox-hugo.scripter.co/) package seemed a perfect fit.

I tried following a tutorial that involved pushing the code to CodeCommit, CodePipeline would pick up the change and then kick off a build with CodeBuild to run Hugo and sync the public folder to an S3 bucket. It was a cool [tutorial](https://conormclaughlin.net/2017/11/automating-deployment-of-your-hugo-site-to-s3-using-aws-codepipeline/) and quite a novel solution, but it failed a few times for me when running Hugo in the build job. I was looking through the Hugo documentation to see where I was going wrong when I noticed the &ldquo;hugo deploy&rdquo; command. This command just uploads the public directory to an S3 bucket for you!


<a id="org4a8806c"></a>

## Setting up the S3 Bucket

Log into AWS and go into the S3 section then click &ldquo;Create bucket&rdquo;

![img](../images/hugos3/createbucket.png)

Name the bucket after your website (it doesn&rsquo;t need to be the site name, it can say anything) and select an AWS region, I&rsquo;m going for eu-west-2 because it&rsquo;s close to me.

![img](../images/hugos3/namebucket.png)

Untick the &ldquo;Block all public access&rdquo; box and acknowledge the warning at the bottom of that section. See the image above for all the settings I&rsquo;ve used. Once you&rsquo;re happy, scroll to the bottom and select &ldquo;Create bucket&rdquo;

![img](../images/hugos3/selectbucket.png)  

Once created, click into the bucket and then click into the properties tab.

![img](../images/hugos3/selectproperties.png)

Scroll down to the bottom and select Edit on Static website hosting

![img](../images/hugos3/selectstatichosting.png)

Enable static website hosting and enter the page file names index.html and error.html respectively. when you&rsquo;re done save the changes.

![img](../images/hugos3/editstatichosting.png)


<a id="org7224940"></a>

## Set up a user in IAM

Open the IAM dashboard and select users from the left-hand side. Select Add Users and add a user called &ldquo;hugo&rdquo; with access key as the credential type.

![img](../images/hugos3/addhugouser.png)

Go to the next page and select Create Group. Create a group called &ldquo;hugo&rdquo;, search for s3 in the policies and check the box for AmazonS3FullAccess, then select Create Group. Next click into Tags, add any tags you like, click into Review and when you&rsquo;re happy, create the user.

![img](../images/hugos3/addhugogroup.png)

On the final page, you&rsquo;ll be given the Access Key ID and the Secret Access Key

![img](../images/hugos3/hugoaccesskeys2.png)

Take a note of these for later and don&rsquo;t share them with anyone:

    Access Key ID:
    AKIA4QJB6VGADXAFZ3RZ
    
    Secret Access Key:
    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


<a id="org40d9ae0"></a>

## Installing the AWS CLI

Curl the installer file:

    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"\
    unzip awscliv2.zip\

Run the installer:

    sudo ./aws/install

Then configure the CLI with your credentials from earlier

![img](../images/hugos3/awsconfigure2.png)

You should now be able to interact with your AWS subscription. Try running &ldquo;aws s3 ls&rdquo; and you should see the bucket you&rsquo;ve created.

    [username@hostname]$ aws s3 ls
    2022-08-19 21:36:22 dougiepeart.co.uk

You&rsquo;ve now got the AWS CLI set up, now all that&rsquo;s left is to configure Hugo to push to your bucket (and configure your Hugo site, but that&rsquo;s beyond the scope of this tutorial).


<a id="org0341bc7"></a>

## Configuring Hugo

Inside your config.toml file for your Hugo site append the following:

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

Obviously, change the S3 URL to your bucket name and selected region though.

Congratulations you&rsquo;ve got everything you need to push your site from your machine to S3!

cd into your websites directory and run:

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

You&rsquo;ve now got your site in a publicly accessible S3 bucket :)

You can find the URL in the bucket&rsquo;s properties under the Static website section

But you have to navigate to a stupid URL to see it :(


<a id="orgf7fa5c6"></a>

## Configuring your DNS record.

So, log into Cloudflare (assuming you&rsquo;ve set [them up as your DNS provider](https://developers.cloudflare.com/dns/zone-setups/full-setup/setup/)) and go into the DNS section.

Ordinarily, you&rsquo;d set an A record here pointing to an IP address, but as I&rsquo;m sure you&rsquo;ve noticed, AWS didn&rsquo;t allocate us one. Not to worry though, you don&rsquo;t need an A record, you can just use a CNAME.

![img](../images/hugos3/cname_000.png)

You can just use @ for the name here, but I&rsquo;ve set it as dougiepeart.co.uk for clarity.

Paste in your S3 URL and ensure it is set to proxied as you&rsquo;ll be using this for your SSL certificate.

You should now be able to browse to your website and see your Hugo site :)


<a id="org2487c85"></a>

## Adding a Certificate

The final step. In Cloudflare, go into the SSL/TLS section on the left-hand side. In the overview, set it to Flexible.

![img](../images/hugos3/sslflexible.png)

If you reload your website, you should now have a secure connection.

That&rsquo;s it. You now have super cheap and fast hosting for your static website.

If you have any questions, feel free to email [me](mailto:contact@dougiepeart.co.uk).


<a id="orgb592573"></a>

# DONE Book Notes: The Beginner&rsquo;s guide to Stoicism     :book:notes:stoicism:philosophy:reading:@reading:


<a id="orgaa02912"></a>

## Source

Author: Matthew Van Natta
Title: The Beginner&rsquo;s Guide to Stoicism
Year: 2019
Genre: Philosophy
Rating: 9/10


<a id="org365b50e"></a>

## The Book in 3 Sentences

This book introduces the tools one needs to deal with life&rsquo;s challenges in a calm and composed manner. You already possess the keys to living a fulfilling and joyous life. Despite having been created centuries ago, its teachings still apply to modern life.


<a id="org4a660e8"></a>

## Impressions

I found it immensely useful. It is packed with information that you can put to use straight away. It has taken me quite a while to compile my notes for it, but it was worth it.


<a id="orgf72b5db"></a>

## Who Should Read it?

I think it could benefit every person to read. However, I would highly recommend it to those like myself, who overthink most things in life and are looking for a way to overcome that habit.


<a id="orgd3ee9df"></a>

## How the Book Changed Me

I have been applying this in my life for a few weeks now and as overused as this phrase is, it has been life changing for me. I feel like I finally have control and that I can appreciate life for what it is. I am embracing my fate.

It is the first book I have taken detailed notes on since school, I think that says a lot about its effect on me.


<a id="orgeca2cf6"></a>

## My Top 3 Quotes

> &ldquo;Your mind is yours-and yours alone. If you focus on healthy thoughts and develop balance opinions about your situation, you will cultivate positive emotions and find lasting enthusiasm to live your best life. You will see negativity for what it is: a waste of your energy.&rdquo;
> 
> &#x2013; Matthew J. Van Natta, The Beginner&rsquo;s Guide to Stoicism, Page 19

> &ldquo;If you have received an impression of any pleasure, guard yourself and create a delay. Then think of the time you will enjoy the pleasure, and the time after, when you will repent and be disappointed with yourself. On the other side, imagine your happiness if you resist the temptation and get to commend yourself for the victory.&rdquo;
> 
> &#x2013; Epictetus, Enchiridion 34

> &ldquo;The only possession the wise person has is virtue, and of this they can never be robbed. Of all else they have merely the use on loan.&rdquo;
> 
> &#x2013; Seneca, On the Firmness of the Wise Man


<a id="org76ff394"></a>

## Summary + Notes


<a id="orgbe53ad5"></a>

### Myths and Misconceptions

The Stoics often dealt with Stoicism being misunderstood as being a cold philosophy. They were mistaken to be cold and unfeeling. However, Stoics taught that no one should strive to be an unfeeling statue.

Living a virtuous life leads to living a life rich with emotion. Embracing the positive, whilst quickly overcoming the negative.

Passivity is another common mischaracterisation. Stoics believe you can thrive in any situation, it teaches acceptance of the world as it is. A love of one&rsquo;s fate. Amor Fati. This can be confused with Apathy. &ldquo;Why change,&rdquo; they say, &ldquo;if one can be happy even in the worst of life&rsquo;s storms?&rdquo;. It may seem paradoxical, but Stoic acceptance gives one the strength to overcome life&rsquo;s challenges. When a rude person makes aggressive demands, the Stoic accepts that the person in front of them is being hostile, but they can choose how to respond. If the rude person&rsquo;s demands are unjust, the Stoic works for justice.

When you direct your attention toward that which you can control, your actions become well-aimed and effective.


<a id="orgeac6be2"></a>

### Living In Accordance With Nature

> &ldquo;When you are going to take in hand any act, remind yourself what kind of an act it is. If you are going to bathe, place before yourself what happens in the bath: some splashing the water, others pushing against one another, others abusing one another, and some stealing: and thus with more safety you will undertake the matter, if you say to yourself, I now intend to bathe, and to maintain my will in a manner conformable to nature. And so you will do in every act: for thus if any hindrance to bathing shall happen, let this thought be ready: it was not this only that I intended, but I intended also to maintain my will in a way conformable to nature; but I shall not maintain it so, if I am vexed at what happens.&rdquo;
> 
> &#x2013; Epictetus, Enchiridion 4


<a id="orgb185339"></a>

### The Three Stoic Disciplines

1.  The Discipline of Desire

    Aligning your values to only desire only that which is in your control.

2.  The Discipline of Action

    Aim to seek healthy and positive relationships with everyone you meet, even knowing that they may not reciprocate. 

3.  The Discipline of Assent

    Separating your initial reactions to the world from your final judgements about the world. Evaluate your thoughts to align with wisdom.


<a id="org52a1958"></a>

### The Four Stoic Virtues

1.  Wisdom

    Wisdom is most closely associated with the Discipline of Assent. It can be subdivided into the following:
    
    -   Good Sense
    -   Good Calculation
    -   Quick-wittedness
    -   Discretion
    -   Resourcefulness
    
    The primary role of Wisdom is to lead you to good; to focus your attention on healthy thoughts, opinions, desires, and aversions. It requires constant attention. Practising wisdom requires you to understand that you aren&rsquo;t required to accept the first reaction that comes to mind. You must instead build a bridge between your reaction and your final action.
    
    > &ldquo;The science of what ought or ought not to be done.&rdquo;
    > 
    > &#x2013; Pierre Hadot

2.  Courage

    Courage is mastery over your fears. It stands in opposition to cowardice. Courage is one of the two virtues that are linked to the Discipline of Desire. If you no longer focus your desires and aversions towards the external world but instead strive for virtue, you will unleash the power to act against the intolerable. The Stoics break Courage down into the following:
    
    -   Endurance
    -   Confidence
    -   High-mindedness
    -   Cheerfulness
    -   Industriousness
    
    > &ldquo;The science of what ought or ought not be tolerated.&rdquo;
    > 
    > &#x2013; Pierre Hadot

3.  Justice

    Justice relates to the Discipline of Action. Justice is not a direct translation of our definition of Justice (to abide by the law). It can be more easily understood as *morality*. It encompasses all of our interactions with others. Stoicism teaches that all people are valuable and that we are meant to work together. Stoic Justice helps you to work with others, even if they are opposed to it.
    
    > &ldquo;He who is running a race ought to endeavour and strive to the utmost of his ability to come off victor; but it is utterly wrong for him to trip up his competitor, or to push him aside. So in life it is not unfair for one to seek for himself what may accrue to his benefit; but it is not right to take it from another.&rdquo;
    > 
    > &#x2013;Chrysippus
    > As quoted in De Officiis by Cicero, iii. 10.
    
    As Marcus Aurelius puts it:
    
    > &ldquo;We were born to work together like feet, hands and eyes, like the two rows of teeth, upper and lower. To obstruct each other is unnatural. To feel anger at someone, to turn your back on him: these are unnatural.&rdquo;
    > 
    > &#x2013; Marcus Aurelius, Meditations, 2:1
    
    Justice can be broken down into:
    
    -   Honesty
    -   Equity
    -   Fairness
    -   Goodwill
    -   Benevolence
    -   Kindness
    
    > &ldquo;The science of what ought or ought not be disturbed.&rdquo;
    > 
    > &#x2013; Pierre Hadot

4.  Temperance (Moderation)

    Temperance is control over one&rsquo;s desires and, along with Courage, it is an expected outcome of practising the Discipline of Desire. Moderation stands in opposition to excess. If you desire only virtue, then you can be reasonable in what you want and generous with what you have been given. Moderation can be divided into:
    
    -   Appropriateness
    -   Modesty
    -   Self-control
    
    > &ldquo;The Stoics viewed life as a banquet. Picture yourself at a party where the host went all out with the food and drink. Imagine expensive wines, mouthwatering dishes, and decadent desserts. Everyone’s grabbing plates and glasses and heading over to get their fill. How are you going to act? Are you going to pile your plate high so you don’t miss out on anything? Will you fill up your glass knowing there’s not enough of that particular wine for everyone to try? If you miss out on the dessert you had your eye on, will it ruin your night? If you get to it in time, will you take so much that others don’t get to enjoy it? Epictetus says that a Stoic won’t desire that dessert before they have it and therefore won’t be disturbed if they never get it. If the dessert does arrive, they won’t take so much that others will be left without. Also, if they really grasp Stoicism, they might choose to let the dessert pass even though it arrives. That Stoic, Epictetus claims, is worthy of ruling with the gods! The banquet metaphor is meant to apply to all of your interactions. If you direct your desire toward being your best self, you won’t focus on getting things, but on using the things you already have.&rdquo;
    > 
    > &#x2013; Matthew J. Van Natta, The Beginner&rsquo;s Guide to Stoicism, Page 76
    
    > &ldquo;The science of what ought or ought not to be chosen.&rdquo;
    > 
    > &#x2013; Pierre Hadot


<a id="org7cb0c85"></a>

### Rules To Live By - Page 19

> &ldquo;Your mind is yours-and yours alone. If you focus on healthy thoughts and develop balance opinions about your situation, you will cultivate positive emotions and find lasting enthusiasm to live your best life. You will see negativity for what it is: a waste of your energy.&rdquo;
> 
> &#x2013; Matthew J. Van Natta, The Beginner&rsquo;s Guide to Stoicism, Page 19

Rarely do people notice that you&rsquo;re annoyed with them. So what is the use of being annoyed?


<a id="orgb4a0599"></a>

### A Pythagorean Practice - Page 32

The Stoics particularly enjoyed a Pythagorean practice that asks you to review your day by asking

-   What have I done wrong?
-   What have I done well?
-   What have I left that must be done tomorrow?


<a id="orgc9dfb3e"></a>

### Socrates&rsquo;s Views Adopted Into Stoicism - Page 33

-   No one desires to do evil
-   No one makes a mistake willingly
-   Virtue is sufficient for happiness


<a id="org21be024"></a>

### Essential Tools

1.  Circle The Present

    > &ldquo;Then remind yourself that past and future have no power over you. Only the present - and even that can be minimized. Just mark off its limits.&rdquo;
    > 
    > &#x2013; Marcus Aurelius, Meditations 8:36
    
    > &ldquo;Mark off its limits,&rdquo; references a practice called circling the present. It provides a way to relieve stress, catastrophic thinking, and other anxieties. To do this, you only allow yourself to dwell on the present, essentially fencing yourself off from the future and the past. Take a breath. Draw your attention to the present moment. The past is finished. The future is unknowable. Leave anxieties about the future alone; they solely exist in your imagination. You can only act in the present.&ldquo;
    > 
    > &#x2013; Matthew J. Van Natta, The Beginner&rsquo;s Guide to Stoicism, Page 45

2.  Infinite Opportunity

    > &ldquo;When your desires and aversions are things or situations, you will think of moments that don&rsquo;t provide what you want or moments that confront you with things you&rsquo;d avoid as &lsquo;bad.&rsquo; If you stop focusing on outcomes- but instead desire being your best at every moment- you&rsquo;ll understand that every situation provides an opportunity to practice virtue.
    > 
    > When confronted by a challenge, ask yourself:
    > 
    > -   How can I benefit from this?
    > -   What virtue can I draw on to meet this moment?&ldquo;
    > 
    > &#x2013; Matthew J. Van Natta, The Beginner&rsquo;s Guide to Stoicism, Page 46

3.  Pause and Compare

    > &ldquo;If you have received an impression of any pleasure, guard yourself and create a delay. Then think of the time you will enjoy the pleasure, and the time after, when you will repent and be disappointed with yourself. On the other side, imagine your happiness if you resist the temptation and get to commend yourself for the victory.&rdquo;
    > 
    > &#x2013; Epictetus, Enchiridion 34

4.  Reserve Clause

    How does one set out to perform an action and remain aligned to their virtues if it does not happen the way one hoped? The reserve clause.
    
    &ldquo;I will buy bread from the shop on the way home from work today, if nothing prevents it.&rdquo;
    
    You may get to the shop and find they don&rsquo;t have any bread, or that you&rsquo;ve forgotten your wallet. This is where the phrase &ldquo;if nothing prevents it,&rdquo; has power. I want to do *x*, but I also understand that I do not control the outcome.

5.  You Are Just An Appearance

    > &ldquo;You are just an appearance and not at all the thing you appear to be.&rdquo;
    > 
    > &#x2013; Epictetus
    
    Whenever an overwhelming value judgement forms in your mind, pause and repeat that line. Say no to the judgement until you&rsquo;ve examined it further.

6.  Bracketing

    When an event happens, take it at face value. After this, you can &ldquo;say something more.&rdquo; Ask a basic Stoic question. &ldquo;Is this under my control?&rdquo; This will allow you to think with a more clear mind and allow you to make a more reasoned judgement.

7.  Circle Yourself

    This is similar to *circle the present*. Think about what is most important to you, your ability to control your thoughts, actions, desires and aversions.
    
    Mentally separate this from the outside world. Remind yourself that your mind is fully under your control. Your will directs your thoughts and actions. Thinking of this will allow you to be free from outside influences, and you can choose the best action for you. The action which allows you to act according to the virtues.

8.  Physical Definition - Wisdom Exercise

    This exercise attempts to strip away your personal—and perhaps irrational—feelings concerning your desires. When you think about something you want, it helps to have a clear idea of it. Those expensive sneakers you want? They’re only shoes: leather meant to protect your feet. If you buy them they will wear out, get stained, and eventually become trash. Are shoes really worth stressing over?
    
    Epictetus asked his students to imagine they had a favourite cup. What is it on the most basic level? It’s ceramic. It holds drinks. It’s breakable. He told them to leave behind thoughts of “it’s painted so beautifully” and “it was a birthday present,” so they could see it as just a cup. A cup that, if broken, isn’t worth losing your good flow of life. When anything presents itself to you, particularly if it seems in some way overwhelming, stop and define it at its most basic. Do not add value judgments. Clear away its mystique so that you can move forward with a clear head.
    
    Marcus Aurelius would use this practice to avoid temptation. For instance, he would use it when he felt himself in danger of making a decision based on lust. 
    
    > &ldquo;and as for sexual intercourse, it is the friction of a piece of gut and, following a sort of convulsion, the expulsion of some mucus.&rdquo;
    > 
    > &#x2013; Marcus Aurelius, Meditations 6.13

9.  This Is Nothing to Me

    When anything begins to take over your mind. Think of the phrase &ldquo;This is nothing to me.&rdquo;
    
    If you are hoping for something to happen, like getting a promotion at work and the anticipation or the outcome turns into negative thoughts. Re-frame it in your mind by saying &ldquo;This promotion is nothing to me&rdquo; It will not affect your ability to live with virtue. It will not stop you from being the best possible version of yourself.
    
    This stops the situation from stealing away the positive emotions you could be experiencing right now.

10. It Was Returned

    > &ldquo;The only possession the wise person has is virtue, and of this they can never be robbed. Of all else they have merely the use on loan.&rdquo;
    > 
    > &#x2013; Seneca, On the Firmness of the Wise Man
    
    The Stoics taught that all indifferents should be viewed as being on loan. All things will come to an end, no person is immortal, and nothing lasts forever.
    
    If you treat everything as if it will be yours forever and you lose it. Then you are building yourself up to feel negative thoughts. Saying &ldquo;It was returned&rdquo; when you lose something will remind you that nothing lasts forever and you will learn to appreciate the present, the time you have with it or the time you had with it after it is gone.


<a id="org0fb98f3"></a>

### Seneca&rsquo;s Description of Stoicism

> &ldquo;No school has more goodness and gentleness; none has more love for human beings, nor more attention to the common good. The goal which it assigns to us is to be useful, to help others, and to take care, not only of ourselves, but of everyone in general and of each one in particular.&rdquo;
> 
> &#x2013; Seneca  

> &ldquo;From time to time, return to Seneca&rsquo;s description. Are you becoming someone who fits this description? If you are, you&rsquo;re doing a great job. Keep it up.&rdquo;
> 
> &#x2013; Matthew J. Van Natta, The Beginner&rsquo;s Guide to Stoicism, Page 79


<a id="org544e2be"></a>

### What DO You Control?

-   Your opinions about life
-   What you pursue in life
-   What you want
-   What you do not want


<a id="orgb023770"></a>

### Before You Sleep

> &ldquo;Let us go to our sleep with joy and gladness; let us say &lsquo;I have lived; the course which Fortune has set for me is finished.&rsquo; And If God is pleased to add another day, we should welcome it with glad hearts. A person is happiest, and is secure in his own possession of himself, who can await the morrow without apprehension. When a man has said: &lsquo;I have lived!&rsquo;, every morning he arises he receives a bonus.&rdquo;
> 
> &#x2013; Seneca, Letters From A Stoic 12

Seneca teaches that this practice can free you from anxiety about the future, allowing you to wake up with each fresh day and rejoice that you have been blessed with the gift of another day, another chance to live the best life you can live.


<a id="orgd1bcc33"></a>

### The Stoic Passions

1.  Negative Passions

    1.  Fear
    
        Fear is the expectation of a coming evil. Some of the feelings that flow from fear are:
        
        -   Terror
        -   Hesitation
        -   Shame
        -   Shock
        -   Panic
        -   Anguish (Regret)
        
        Shame is the fear of being disgraced, but can you be disgraced if you act with virtue? No, no one can disgrace you and no one can cause you to act without virtue.
        
        All fears are built on indifferents. The Dichotomy of Control will help you to see that indifferents are out of your control.
    
    2.  Lust (Appetite)
    
        Lust is the desire for an expected good. It is focused on the future. The future is out of your control so why concern yourself with it?
        
        Some of the feelings associated with Lust include:
        
        -   Want
        -   Hatred
        -   Contention
        -   Anger
        -   Lust
        -   Wrath
        -   Rage
        
        Desire only what you have and you will overcome Lust. Imagine living without dissatisfaction? If you can learn to only desire your own virtuous actions in life, then you won&rsquo;t have an overwhelming appetite for indifferent things.
    
    3.  Delight (Pleasure)
    
        Delight is a mistaken elation over something that seems to be good in the here and now. It includes such things as:
        
        -   Rejoicing at others misfortune
        -   Self-gratification
        -   Extravagant joy
        
        This isn&rsquo;t to say that happiness is a bad thing, far from it. Delight ties your well-being to things you own or a particular moment, all of which can be taken from you. The opposite of Delight is Joy, the state of mind that finds positivity despite the impermanence of things or moments.
    
    4.  Distress
    
        Distress is the irrational drawing away of the mind from something already present. The following emotions come from Distress:
        
        -   Malice
        -   Envy
        -   Jealousy
        -   Pity
        -   Grief
        -   Worry
        -   Sorrow
        -   Annoyance
        -   Vexation
        -   Anguish
        
        To overcome Distress, you must make room for other good passions to flourish. One technique for this is the festival mindset:
        
        > &ldquo;When you&rsquo;re alone you should call this tranquillity and freedom, and think of yourself like the gods; and when you are with many, you shouldn&rsquo;t call it a crowd, or trouble, or uneasiness, but festival and company, and contentedly accept it&rdquo;
        > 
        > &#x2013; Epictetus, Discourses 1:12
        
        When you are at a festival or a concert, you have a great time and the crowds don&rsquo;t annoy you at all. You welcome the crowds, they are all there with you, and you are all sharing the same experience. These are your people. Why not try applying this to any situation?


<a id="orgbaf8d6d"></a>

### What Is So Unbearable About This Moment?

> &ldquo;Do not disturb yourself by picturing your life as a whole; do not assemble in your mind the many and varied troubles which have come to you in the past and will come again in the future, but ask yourself with regard to every present difficulty: ‘What is there in this that is unbearable and beyond endurance?’ You would be ashamed to confess it! And then remind yourself that it is not the future or what has passed that afflicts you, but always the present, and the power of this is much diminished if you take it in isolation and call your mind to task if it thinks that it cannot stand up to it when taken on its own.&rdquo;
> 
> &#x2013; Marcus Aurelius, Meditations 8:36

Most of the emotion that comes from an event comes from your own imagination. Is this thing *really* as bad as you think it to be?


<a id="orgd5608f6"></a>

### It Seemed So to Them

> &ldquo;When any person treats you badly or speaks ill of you, remember that they do this because they think they must. It&rsquo;s not possible for them to do what you think is right, but only what seems right to them&#x2026;If you understand this you will have a milder temper with those who revile you because you can always say, &lsquo;it seemed so to them.&rsquo;&rdquo;
> 
> &#x2013; Epictetus, Enchiridion 42

This points back to Socrates&rsquo;s view that no one desires to do evil. They do not know any better, for if they did, they would act better. You cannot expect people to act with the knowledge you have, because they do not have access to that. This reminds me of a quote that is often falsely attributed to Marcus Aurelius:

> “Everything we hear is an opinion, not a fact. Everything we see is a perspective, not the truth.”


<a id="orgbf1d881"></a>

### Creating Change

> &ldquo;When you do a thing because you have determined it ought to be done, never avoid being seen doing it, even if the opinion of the multitude is going to condemn you. For if your action is wrong, then avoid doing it altogether, but if it is right, why do you fear those who will rebuke you wrongly?&rdquo;
> 
> &#x2013; Epictetus, Enchiridion 35

Do not be afraid to do what you believe is the right thing to do. Some or all may doubt your goals, but if it is truly the correct thing to do, then you should do it anyway, regardless of what people may say.


<a id="org6efa79c"></a>

### Establish A Daily Routine

Choose some of the practices that are talked about in this book to incorporate into your daily life. Creating a Stoic routine is grounding. It will give you the strength to weather any of life&rsquo;s storms.

Start the day with a morning orientation, journal, and think about the events and tasks today brings and how best to prepare for them.

End your day with an evening reflection, such as the Pythagorean practice from page 32:
&ldquo;What have I done wrong? What have I done well? What have I left that must be done tomorrow?&rdquo;

Find the method that works for you. Matthew talks about when he first started he would write a few phrases on scraps of paper and keep them in his pocket. If he felt stressed, he would read a relevant phrase that would help him to centre himself.

If you prepare and deal with bad times properly, they&rsquo;ll be good times. That is to say that if you live according to the virtues in every situation and virtues are the only good, then any situation can be a source of contentment for being the best possible version of yourself. This doesn&rsquo;t mean being joyful if someone has died, but rather acting according to what the situation demands of you and being rewarded for that with deep inner satisfaction.

