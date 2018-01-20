# Automating Netflix geoblocking

_DISCLAIMER: This has been written for educational purposes of learning about routing net traffic. I am not responsible for how you use this information. The title is clickbait only._

At the start of last year, I learnt how to launch a browser from another country, using Amazon Web Services (AWS) as a proxy service. 

At the start of this year, I also learnt of a great program called `terraform` (learn about it [here](https://www.terraform.io/intro/index.html), the docs are great!). `Terraform` makes it easy for anyone to use this service. All they need to do is sign up to AWS, create a _Role_ and _Access Key_ and they should be away.

This should work easily for Mac and linux users. Due to `terraform`, the process will be very similar for Windows users, but I have only provided helper `.sh` scripts.

## Plan

There steps we will take are

 - Sign up to AWS
 - Make a User Role with an Access Key
 - Download and install `terraform`
 - Let `terraform` know our credentials
 - Automatically provision the proxy server
 - Launch `google-chrome` via the proxy server 

### Sign up to AWS

Sign up to AWS [here](https://portal.aws.amazon.com/billing/signup). If your first year of usage has already expired and you want to continue using free tier machines, create a new email address to sign up with first.

You will have to provide your payment card details. You will not be charged for the first year, using AWS only for the infrastructure in the provided `*.tf` file. Nonetheless **you do this at your own risk and I am not responsible** if you do get charged for something. Furthermore, if you do get charged, please let me know, as it might mean I'm getting charged, and that's not ideal.