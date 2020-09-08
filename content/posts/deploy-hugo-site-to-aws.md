---
title: "Deploy Hugo site to AWS"
date: 2020-09-06
draft: false
---

[Hugo](https://gohugo.io/) is a fast and modern static site generator written in Go.
It builds pages in under a second. And there are no dependencies to install. It seems like a good idea to use it as a base for a site.

I tryed to use [Jekyll](https://jekyllrb.com/), but I have some issues with it. 
Using Jekyll isn't staringht forward, you need to install Ruby and Gem plugins and enshure right versioning. Themes for a site should be right versioned too.

Hugo sites can be hosted anywhere, including Github Pages, AWS S3, AWS CloudFront or other CDNs.
In this articale, I’d show you how to build Hugo static site and deploy it to S3 and CloudFront.  

## Step 1. Setup Hugo 

1. To install Hugo run `brew install hugo` or `choco install hugo` if you are on Windows.
2. Create a new site with `hugo new site example.com`.  
3. Hugo provides many awesome [themes](https://themes.gohugo.io/). To add a theme run

```bash
$ cd example.com
$ git init
$ git submodule add https://github.com/budparr/gohugo-theme-ananke.git themes/ananke
```
    
4. Then add theme to your `config.yml`:

```bash
$ echo 'theme: "ananke"' >> config.yml
```

5. Add some content with `hugo new posts/my-first-post.md`.  
And edit it like this:

```markdown
---
title: "My First Post"
date: 2019-03-26T08:47:11+01:00
draft: true
---

Hello World!!!
```

5. Start Hugo server `hugo server -D`. http://localhost:1313/.  
Hugo will automatically update any content.

6. Build static pages `hugo -D` then all site will be created under `/public` directory.

## Step 2. Setup AWS

AWS has a top noch tooling for building static websites. But unfortunatelly learing curve is very steep.
I'll give you step by step explaation on how to create static website hosting on AWS.

For hosting a website you need to create an Amazon Route 53 hosted zone. 
At first you need to register a [new domain](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/domain-register.html).  

Then, you can use following CloudFormation template to create a Hosted Zone.

```yaml
AWSTemplateFormatVersion: '2010-09-09'

Description: Creates an Amazon Route 53 hosted zone

Parameters:

  DomainName:
    Type: String
    Description: The DNS name of an Amazon Route 53 hosted zone
    AllowedPattern: (?!-)[a-zA-Z0-9-.]{1,63}(?<!-)
    ConstraintDescription: must be a valid DNS zone name.

Resources:

  DNS:
    Type: AWS::Route53::HostedZone
    Properties:
      HostedZoneConfig:
        Comment: !Join ['', ['Hosted zone for ', !Ref 'DomainName']]
      Name: !Ref 'DomainName'
```

Then we will creates a S3 bucket configured for hosting a static website, 
and a CloudFront distribution with HTTPs.  

For that you will need to provide ACM Certificate ARN in a format `arn:aws:acm:us-east-1:1234567890:certificate/1234567890` hosted in `us-east-1` zone. CloudFront uses `us-east-1` zone and `Z2FDTNDATAQYW2` CloudFront hoster zone id by default.  

Also you will need your domain name, for example `example.com` and a hosted zone Id from the previous step. 

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Creates an S3 bucket configured for hosting a static website, and a Route
  53 DNS record pointing to the bucket
Parameters:

  HostedZoneId:
    Type: String
    Description: The DNS name of an existing Amazon Route 53 hosted zone.
    AllowedPattern: (?!-)[A-Z0-9]{1,32}(?<!-)
    ConstraintDescription: must be a valid Route53 Hosted Zone ID.  

  DomainName:
    Type: String
    Description: The full domain name e.g. example.com
    AllowedPattern: (?!-)[a-zA-Z0-9-.]{1,63}(?<!-)
    ConstraintDescription: must be a valid DNS zone name.

  AcmCertificateArn:
    Type: String
    Description: the Amazon Resource Name (ARN) of an AWS Certificate Manager (ACM) certificate.
    AllowedPattern: "arn:aws:acm:.*"

Resources:

  WebsiteBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref 'DomainName'
      AccessControl: PublicRead
      WebsiteConfiguration:
        IndexDocument: index.html
        ErrorDocument: index.html
      Tags:
        - Key: Application
          Value: Blog

    DeletionPolicy: Retain

  WebsiteBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref 'WebsiteBucket'
      PolicyDocument:
        Statement:
        - Sid: PublicReadForGetBucketObjects
          Effect: Allow
          Principal: '*'
          Action: s3:GetObject
          Resource: !Join ['', ['arn:aws:s3:::', !Ref 'WebsiteBucket', /*]]

  WebsiteCloudfront:
    Type: AWS::CloudFront::Distribution
    DependsOn:
    - WebsiteBucket
    Properties:
      DistributionConfig:
        Comment: Cloudfront Distribution pointing to S3 bucket
        Origins:
        - DomainName: !Select [2, !Split ["/", !GetAtt WebsiteBucket.WebsiteURL]]
          Id: S3Origin
          CustomOriginConfig:
            HTTPPort: '80'
            HTTPSPort: '443'
            OriginProtocolPolicy: http-only
        Enabled: true
        HttpVersion: 'http2'
        DefaultRootObject: index.html
        Aliases:
        - !Ref 'DomainName'
        DefaultCacheBehavior:
          DefaultTTL: 5
          MaxTTL: 30
          AllowedMethods:
          - GET
          - HEAD
          Compress: true
          TargetOriginId: S3Origin
          ForwardedValues:
            QueryString: true
            Cookies:
              Forward: none
          ViewerProtocolPolicy: redirect-to-https
        PriceClass: PriceClass_100
        ViewerCertificate:
          AcmCertificateArn: !Ref AcmCertificateArn
          SslSupportMethod: sni-only
  
      Tags:
        - Key: Application
          Value: Blog


  WebsiteDNSName:
    Type: AWS::Route53::RecordSetGroup
    Properties:
      HostedZoneId: !Ref 'HostedZoneId'
      RecordSets:
      - Name: !Ref 'DomainName'
        Type: A
        AliasTarget:
          HostedZoneId: Z2FDTNDATAQYW2 1️⃣
          DNSName: !GetAtt [WebsiteCloudfront, DomainName]

Outputs:

  BucketName:
    Value: !Ref 'WebsiteBucket'
    Description: Name of S3 bucket to hold website content

  CloudfrontEndpoint:
    Value: !GetAtt [WebsiteCloudfront, DomainName]
    Description: Endpoint for Cloudfront distribution

  DomainName:
    Value: !Ref 'DomainName'
    Description: Domain name
```

>1️⃣ Default Hosted Zone ID for CloudFront

So, that's about all! You've deployed your website to S3 and CloudFront with HTTPs support.

If you want to setup a CI/CD pipeline for a project, just use the following script:

```bash
git sumodules init 1️⃣
git sumodules update 
hugo -D 
aws s3 cp --delete public/ s3://example.com 2️⃣
```

>1️⃣ this will init and clone a theme to your code  
2️⃣ this will deploy your site to S3 and CloudFront. New pages will be available based on CloudFront's `DefaultTTL`, `MaxTTL`
