---
layout: bits
title: Nginx and Amazon S3 CDN
---
One of the projects I've been working on recently required a reliable CDN. There were a number of requirements that the desing had to follow. One of the limitations was the price tag, so going with something like Akamai or Limelight was not an option. This had to be a more or less in-house solution and I was given a task to prepare it.

## Requirements

There were a few requirements the project had to meet.

### Reliability

First and foremost - this had to be a proven technology. Or at least something that would make me (i.e. the system administrator's) sleep safe and sound :)

### Two types of content - public and private

The CDN had to serve two types of content. One was public, that any user could access. The other had to be protected and should be served only for authorized users.

### SSL distribution of the private content

Both the private and public content had to be served over a secure connection. There was no limitation on whether the certificate had to be signed by a recognised authority. Self-signed certificates were "just fine".

## The solution

Given the above requirements I came up with a solution that was basing on the following two components:

* Amazon EC2 AMI with an nginx server acting as a forward-proxy
* Amazon S3 used as the backend storage for both the private and public content

Since the application, that was going to make use of the CDN, was not an ordinary web browser, we could make it work without the HTTP load balancer. Instead, a DNS round-robin was used. This approach eliminated a layer of load balancers that not only cost money to run, but also create an area where something may go wrong.

## Use-case scenarios

There are two use case scenarios.

### Public access

The public access model allows the user to access files directly from Amazon S3.

![Public access path](http://static.onoclea.com/images/bits/nginx-and-amazon-s3-cdn/cdn-public_access.png)

No middleman is necessary as the traffic may go directly to Amazon S3 servers.

### Private access

The private content is proxied (with the help of `X-Accel-Redirect` (see this article: [Nginx-Fu: X-Accel-Redirect From Remote Servers](http://kovyrin.net/2010/07/24/nginx-fu-x-accel-redirect-remote/)) and the [`XSendFile` nginx module](http://wiki.nginx.org/XSendfile)) by nginx, with the original version stored safely on Amazon S3.

![Private access path](http://static.onoclea.com/images/bits/nginx-and-amazon-s3-cdn/cdn-private_access.png)

This way we can have many `api` machines without the trouble of having to keep them synchronized. The most up-to-date content is always available from Amazon S3. No extra costs for the EBS storage or complex DRBD setups.

### Amazon S3

OK, so here goes the fun and technical part.

#### Bucket policy

Firstly - we must set an appropriate Amazon S3 bucket policy. The one I used looked more or less like this:

    {
     "Statement": [
      {
       "Effect": "Allow",
       "Principal": {
        "AWS": "*" 
       },
       "Action": "s3:GetObject",
       "Resource": "arn:aws:s3:::myproject/public/*" 
      },
      {
       "Effect": "Allow",
       "Principal": {
        "AWS": "*" 
       },
       "Action": "s3:*",
       "Resource": "arn:aws:s3:::myproject/private/*",
       "Condition": {
        "IpAddress": {
         "aws:SourceIp": [
          "api_0_ip_address/32",
          "api_1_ip_address/32" 
         ]
        }
       }
      }
     ]
    }

There was a single bucket called `myproject` with two folders - `private` and `public`. The latter is widely available. The former, on the other hand, can be accessed only from the machines, whose IP addresses were explicitly set in the policy conditions.

#### Upload user IAM policy

This, on the other hand, is the IAM policy for a maintenance user. When attached to a user (say... "myproject-cdn-maintainer") it will restrict its operation to uploading/modifying content and prevent any ACL/policy operations, that could leave the system in a inoperable state.

    {
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "s3:GetObject",
            "s3:PutObject",
            "s3:GetObjectAcl",
            "s3:PutObjectAcl",
            "s3:DeleteObject" 
          ],
          "Resource": [
            "arn:aws:s3:::myproject/private/*",
            "arn:aws:s3:::myproject/public/*" 
          ]
        },
        {
          "Effect": "Allow",
          "Action": "s3:ListAllMyBuckets",
          "Resource": "arn:aws:s3:::*" 
        },
        {
          "Effect": "Allow",
          "Action": "s3:ListBucket",
          "Resource": "arn:aws:s3:::myproject" 
        }
      ]
    }

#### Nginx configuration

The nginx configuration (the relevan part) looked the following:

    location ~* ^/private/(.*) {
     internal;
    
     resolver 172.16.0.23;
    
     set $download_uri $1;
     set $download_host myproject.s3.amazonaws.com;
     set $download_prefix private;
    
     proxy_set_header Authorization '';
     proxy_set_header Host $download_host;
    
     proxy_hide_header X-Amz-Id-2;
     proxy_hide_header X-Amz-Request-Id;
     proxy_hide_header ETag;
     proxy_hide_header Last-Modified;
    
     set $download_url http://$download_host/$download_prefix/$download_uri;
    
     error_log /var/log/nginx/api.myproject.onoclea.net_private-error.log debug;
     access_log /var/log/nginx/api.myproject.onoclea.net_private-access.log main;
    
     # retry on Amazon errors
     proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
    
     # pass only GETs
     proxy_method GET;
     proxy_pass_request_body off;
     proxy_set_header Content-Length '';
    
     # disable buffering
     proxy_buffering off;
     proxy_max_temp_file_size 0;
    
     proxy_pass $download_url;
    }

The idea is that whenever the backend application (the one that is going to verify if a user can get an access to the private content) responds with an appropriate `X-Accel-Redirect` header, the internal location will serve a protected content from the Amazon S3. Since the traffic between S3 and EC2 is charged $0 per GiB (i.e. it's free) this imposes no additional costs.

## Caveats

There were some minor quirks I came across.

### Pay attention to default permissions while uploading your data

First of all - remove all ACLs from the data uploaded to S3. According to the IAM (AWS Identity and Access Management) documentation ([IAM Access Policy Language Evaluation Logic](http://docs.amazonwebservices.com/IAM/latest/UserGuide/index.html?AccessPolicyLanguage_EvaluationLogic.html)) object ACLs will override general allow/deny policies. Since [Cyberduck](http://cyberduck.ch/), a tool I used to upload the test content, sets a default 'read all' ACL ([see the doc](http://trac.cyberduck.ch/wiki/help/en/howto/s3#DefaultACLs)) I had to spent some time figuring why the content that should be kept private is widely available. Purging all ACLs fixed that.

### If you want https access you cannot use CNAMEs

The wildcard certificate, issued for the S3 service, is valid only for `*.s3.amazonaws.com`. So if you want to get your CNAME point to the S3 you will get security warning if you use https. So etiher stay with the plain HTTP (i.e. http://cdn.myproject.com) or use a single-level subdomain (http://cdn-myproject-com.s3.amazon.aws.com).
