<!-- # Replicate to a second region --> 
# Configure Active-Active solution using Route53

Now that we set up the Bookstore App in Ireland and Singapore regions, and let's configure Route53 for active-active multi-region solution. 

You don't need to purchase or repurpose your domain name for this workshop as our CDK created subdomain automatically for you. Also the CDK made a request ACM certificates in Ireland and Singapore regions for your convinience. 

Naviagte you have 4 certificates in each region.
![ACM](../images/03-cert-01.png)

## 1. Configure Custom Domains on each API Gateway in each region

<!-- Now that you have a domain name and a valid certificate for it, you can go
ahead and setup your APIs for each region to use your custom domain.  -->
API Gateway Custom domains allow you to access your API using your own domain
name. 
<!-- While you can configure DNS records to point directly to the regular API
Gateway endpoint, an error will be returned unless you have this custom domain
configuration. -->

AWS recently launched new feature in Custom Domains names of Amazon API Gateway. Customers can now create wildcard custom domain names for their Amazon API Gateway EDGE, Regional, and WebSocket APIs. This enhancement extends API Gateway’s existing support for invoking APIs via custom domain names backed by certificates from AWS Certificate Manager (ACM).

Creating an API Gateway wildcard custom domain name makes it easy to provide flexible API invoke URLs to customers, allowing for scalable customer isolation. For example, a common custom domain name use case is to implement customer-specific routing by providing each API customer with a unique, branded invoke URL. Previously this required creating a new API Gateway custom domain name for each customer, but now implementing this strategy will be possible with just one wildcard custom domain name.

You need one domain configurations in each Region with `*.` 

* `eu-west-1` Ireland:
    * `*.arc30901.multi-region.xyz`
* `ap-southeast-1` Singapore:
    * `*.arc30901.multi-region.xyz`

**High-level instructions**

Navigate over to the `API Gateway` in Ireland region, and create `Custom Domain Names`.
* HTTP
* Domain Name: `*.<subdomain>.multi-region.xyz`
* TLS 1.2
* Regional
* ACM Certificate: '<subdomain>.multi-region.xyz`
Edit to add `Base Path Mappings` with `/` for path and `prod` for destination. 
![API Gateway](../images/03-apig-01.png)

Repeat the same process in Singapore region. 

You will use this Custom Domain in Ireland and Singapore to configure health checks and DNS records in Route 53. Copy the `Target Domain Name` from each region.

## 2. Configure Route53 DNS records

### 2.1 Configure DNS records for Health Check

We will creat additional subdomain prefix to configure health checks with `api-ir.` and `api-sg.`. (Please follow the naming.)

**High-level instructions**

Go to the `Route53` and select your `Hosted zones`. Choose your domain name from
the list and you should see a couple of records already configured for
nameservers.

Select `Create Record Set` with a new CNAME record `api-ir.` to point to Target Domain Name for your corresponding API Gateway Custom Domain. You can set the TTL to `1m (60 seconds)` for the
purpose of this workshop. We recommend setting ALL DNS entries to 1m (60 seconds)
as the TTL.
![Route53](../images/03-dns-01.png)

Create one more CNAME record with `api-sg` with `Target Domain Name` of `Custom Domain Name` in Singapore.
![Route53](../images/03-dns-02.png)

At this point you should now be able to visit your subdomain and see your API
working. Navigate to the health check endpoint on your API using your custom
domain in your web browser (e.g. `https://api-ir.arc30901.multi-region.xyz/books`) and
ensure that you see a successful response.

<!-- This endpoint should return the region it is running in so you can also
confirm that this response region matches up with the domain you have
configured. Notice how we're explicitly using HTTPS. It may take a few minutes for your records to
become active so check back later if you do not get a response. -->

### 2.2 Configure Health Check for both regions

In this step you will configure a Route53 health check on both
(Ireland and Singapore) regional endpoints. This health check will be responsible for
triggering a failover between the 2 regions if a problem is detected in the
one region.

<!--Note that if you were configuring an active-active model with something like
Weighted Routing then you would configure a health check on all endpoints, but
only one is necessary in this case since only our primary region will be
handling traffic under normal conditions.-->

**High-level instructions**

Select `Health checks` in Route53, and click `Create health check`.
* Name: identifiable name e.g. `ireland-api`
* What to monitor: `Endpoint`
* Specify endppoint by: `Domain Name`, Protocol: `HTTPS`, Domain name: `api-ir.arc30901.multi-region.xyz`, Path: `books`
* Advanced configuration: change `Request interval` to `Fast (10 seconds)` and set the failure threshold
from *3* down to *1*.  This will greatly speed up the time you need for testing
and failing over (this is not a recommended production configuration but it is
useful for speeding up the remainder of this Workshop).
![Route53](../images/03-dns-03.png)

Do the same for the Singapore region API’s health check.  Once configured, wait a few minutes and you should see your health check go green and say Healthy in the console. Make sure this is green and healthy
before proceeding.
![Route53](../images/03-dns-04.png)

### 2.3 Configure DNS Routing Policy

Now let's configure the zone records for our `api.` subdomain prefix. You will
configure these as CNAME ALIAS records in a weighted pattern using
your health check for Multi-region active-active backend.

***Note.*** To create records for complex routing configurations, you can also use the traffic flow 
visual editor and save the configuration as a traffic policy. However, we use the routing policy
for this workshop, as it is not possible to create an Alias record pointing to a DNS name that is 
being managed by a traffic policy.

**High-level instructions**

Choose `Hosted zones` in `Route53`. Select `Create Record Set` with a new CNAME record `api.` to point to each region API with Alias. Enter `api` as the name and choose
`CNAME` as the type. Now change Alias to `Yes` and select the `api-ir.` prefixed
version of your domain. Since this is an alias, it should appear in the
dropdown list.

Next, choose the `Geolocation` routing policy. Route 53 responds to queries based on the locations from which DNS queries originate. We recommend that you create a `Default` location resource record set. 
Turn on both `Evaluate Target Health` and `Associate with Health Check` then select the `ireland-api` 
health check you created previously. 
![Route53](../images/03-dns-05.png)

Create one more CNAME record with `api-sg` Alias with the same step. Choose the location with `Singapore` instead of `Default`.
![Route53](../images/03-dns-06.png)

With the DNS configured, you should now be able to visit the `api.` prefix of
your domain (remember to use HTTPS). Go to the `/books` path and notice which 
regions is served based on Weighted routing policy.
![Route53](../images/03-dns-07.png)

## 3. Update your UI with new API Gateway Endpoint

Now that we have active-active configuration, you will need to change the API
endpoint in your WebAssets (*bookstore-WebAssets/src/config.js*) to use our newly
created DNS name for our API endpoint.

Go to CodeCommit Repositories, and edit the *config.js* file with `https://api.arc30901.multi-region.xyz` (substituting your own domain) instead of the region specific name.
Also, add in `API_IRELAND_URL` with the FQDN of the Ireland's APIGW CNAME, this is to facilitate our failover testing later.
Commit the changes and wait for Codepipeline/Codebuild to rebuild and update the S3 repository.

![Route53](../images/03-dns-08a.png)

## Completion

Congratulations you have configured a multi-region API and set up a
healthcheck-based Weighted routing policy using Route53. 
![Bookstore](../images/03-complete-01.png)

You can `Sign up` and `Log in` to order books through your domain `https://arc30901.multi-region.xyz/`. Order a book and see if `Order` and `Best Sellers` are working. Also, check the `Order` table in DynamoDB in `Ireland` region with the DynamoDB table in `Singapore` region to see whether your order data is replicated properly. 

***Note*** Sign up requires `Confirmation code` through your valid `Email address`.

In the next module we will intentionally break the primary region (Ireland) and verify that the second region (Singapore) works.

Module 4: [Test Multi-region Failover](../4_TestingFailover/README.md)
