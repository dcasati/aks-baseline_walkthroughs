# Multi-Region Setup with Azure Front Door

The business unit has deployed their application in two regions as independent application stamps. To improve overall availability of their solution for their end customers and protect from regional data center outages, they want both stamps to back up each other.

After further consultation with their architecture team, they decide to test [Azure Front Door](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-overview#key-benefits) offering further benefits: 
- Traffic optimization using edge locations and split TCP
- SSL offloading
- Automatic failover

## Prerequesites
- Make sure you have two independent deployments of the AKS baseline architecture reference implementation deployed and available.
- These need to be deployed to *different regions* (ideally in distant regions).
- When connecting to backend system, Front Door requires certificates issued by a well-known certificate authority. For the purpose of this walkthrough, only self-signed certificates have been created instead. This is why we will use the unencrypted http protocol (instead of https) to connect to the Application Gateways. *Do not use this setup* in a production environment, as it exposes unprotected public endpoints.
  - Follow [this guide](./app-gateway_add-http.md) to add...
    - ...a listener for port 80 in your Application Gateway configuration and...
    - ...a firewall rule in the VNet hosting your Application Gateway.


##  Walthrough Overview
In this walkthrough, you will...
- create an Azure Front Door instance,
- add your application stamps as _origins_ to your Front Door instance,
- see how Front Door directs http(s) requests to the closest endpoint,
- eee how Front Door will handle a failover in case one origin becomes unavailable,
- restrict access to your origins, only allowing incoming traffic from Azure Front Door,
- see that this does not prevent other Front Door instances to access your Application Gateway and
- learn how to restrict access from only your Front Door.

## Procedure

### Create your Azure Front Door instance and add the first origin as backend.

1. Create a new Resource Group with name `[your prefix]-rg-bu0001a0008-multiregion`

   ![](img/001_new-rg.png)

1. Create a new Azure Front Door instance (search for _Front Door and CDN profiles_); select _Azure Front Door_ and _Custom Create_.

   ![](img/011_new-fd_1.png)

   Use `[your prefix]-rg-bu0001a0008-frontdoor` as name for your Front Door instance.

   ![](img/012_new-fd_2.png)

   Do not configure any secrets, but define an endpoint named `[your prefix]-rg-bu0001a0008` 

   ![](img/013_new-fd_3.png)

   ![](img/017_new-fd_endpoint-configuration.png)

   Add a route:

   (Make sure to only _accept_ HTTPS, but use HTTP for _forwarding_.)

   ![](img/016_new-fd_route.png)
   
   Create a new origin group. Remember to ...
   - specify `GET` as probe method,
   - (for this demo setup) reduce the probe interval to 15 seconds.

   ![](img/015_new-fd_origin-group.png)

   And, to this origin group, add a new origin pointing to your Application Gateway as Backend. Please make sure to...
   - ...use the backend region as _Name_ (recommended),
   - ...select _Application Gateway_ as _Origin Type_  and select your instance from the _Host name_ drop down,
   - ...overwrite the _Origin host header_ with your domain name `bicycle.[your domain]` and 
   - ...disable _Certificate subject name validation_.

   ![](img/014_new-fd_origin.png)

   Skip the Security Policy for now.

   If the validation passed, Hit _Create_ and wait for your Front Door to get deployed.

   ![](img/018_new-fd_validation.png)

   ![](img/019_new-fd_deployment-succeeded.png)

1. Get the FQDN of your Front Door...
 
   ![](img/20_test-fd_endpoint-url.png)

   ...and open `https://<your endpoint fqdn>`:

   ![](img/21_test-fd_browser.png)


### Add a second origin as backend.

1. Now add a second origin and use the Application Gateway of an application stamp that one of your colleagues deployed.

   ![](img/030_fd-origin_add-origin.png)

1. Deploy a Test-VM in the region of that second origin. 

   ![](img/031_fd-origin_deploy-test-vm.png)

   Note that it might take some minutes until the configuration update is effective.

1. Connect to that VM, oben the browser and navigate to the Front Door endpoint. Compare the value of _Host Name_ with the value when accessing via your own browser.

   (In the setup used for this tutoruial), these were the container names running in Central US: 
   ```bash
   $ kubectl get pods -n a0008 --context shco-aks-pbl5hvxsyw5yy  | grep aspnet
   aspnetapp-deployment-56b77c4f79-cl8k2       1/1     Running   0          38h
   aspnetapp-deployment-56b77c4f79-cv4hp       1/1     Running   0          38h
   ```
   
   ...and these were running in West Europe:
   ```bash
   $ kubectl get pods -n a0008 --context leho-aks-rio6zecikhluy  | grep aspnet
   aspnetapp-deployment-56b77c4f79-f5bvk       1/1     Running   0          17h
   aspnetapp-deployment-56b77c4f79-l2rpb       1/1     Running   0          17h
   ```

   Azure VM running in Central US:
   ![](img/032_fd-origin_test-access-cus.png)
   
   Author's development VM:
   
   ![](img/033_fd-origin_test-access-author.png)


1. You have seen how Front Door directs user requests to the region closest to the user location.


### Simulate a failover 

1. Stop the Application Gateway in one of your regions.

   ```bash
   az network application-gateway stop --id /subscriptions/[subscription id]/resourceGroups/[resource group name]/providers/Microsoft.Network/applicationGateways/[application gateway name]
   ```

1. Go to the _Metrics_ blade of your Front Door and select _Origin Health Percentage_. The number should have decreased.

   ![](img/041_fd-failover_metrics.png)

1. Refersh the page on the machine running in the region in which you shut down the Application Gateway. It now returns responses from the other region.
   
   ![](img/042_fd-failover_page-refresh.png)

   
1. Start the Application Gateway.

   ```bash
   az network application-gateway start --id /subscriptions/[subscription id]/resourceGroups/[resource group name]/providers/Microsoft.Network/applicationGateways/[application gateway name]
   ```

1. Refresh the page again and see how Front Door now dispatches requests to the nearest region again.
   

## Only allow Front Door(s) to access your backends

1. Note that so far, the public endpoint of your Applcation Gateway is still open to serve any direct request (i.e., not passing Front Door). 

   ![](img/050_restrict-appgw_open-direct-connection.png)

1. To restrict access from Front Door only, go back to the Network Security Groups applied to the subnet hosting your Application Gateway. Change the inbound rule for port 80 to only allow requests originating from Service Tag _AzureFrontDoor.Backend_.

   Note that it might take some minutes until this change is effective.
   
   ![](img/050_restrict-appgw_inbound-from-front-door.png)

   (Of course, as long port 443 remains open on the Network Security Group, the service remains directly accessible via the Application Gateway. This is for demo purposes only.)

1. Direct requests to the Application Gateway from  your browser will now time out, while the service remains accessible via Front Door:

    ![](img/052_restrict-appgw_direct-connection-timeout.png)

1. Get access to a second Front Door (either by deploying another instance or joining forces with another workshop participant.)

1. Ensure both Front Door instances have the same origins set up. And verify, both Front Door instances can well reach all origins. This shows, that the rules in the Network Security Group allow incoming requests from arbitrary Front Door instances to reach the Application Gateway.

   ![](img/053_restrict-appgw_access-from-two-fds.png)

## Ensure only _your_ Front Door can access your Application Gateway using WAF.

1. Take a look at [How do I lock down the access to my backend to only Azure Front Door?
](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-faq#how-do-i-lock-down-the-access-to-my-backend-to-only-azure-front-door-); the last paragraph of this section shows how to use the http header `X-Azure-FDID` to apply a Web Application Firewall (WAF) rule to only accept requests originating from specific Front Doors on the Application Gateway level. In the referred _Example 7_ of [Create and use Web Application Firewall v2 custom rules on Application Gateway](https://docs.microsoft.com/en-us/azure/web-application-firewall/ag/create-custom-waf-rules#example-7), 

1. To only allow _your specific_ Front Door to access the Application Gateway, get the _Front Door ID_ from the resource overview. 

   ![](img/061_waf-policy_front-door-id.png)

1. Browse to the _Application Gateway WAF policy_ in the Resource Group `[your prefix]-rg-bu0001a0008` and navigate to _Custom Rules_.

   ![](img/062_waf-policy_custom-rules.png)

1. Add a new custom rule, denying all traffic not originating from your individual Front Door instance.

   ![](img/2022-05-09-11-49-01.png)

   Do not forget to save the WAF policy configuration.

1. Once the new custom rule is effective, requests dispatched from a Front Door instance with an ID other than `e2fc8833...` will be denied by the Web Application Firewall.

   ![](img/064_waf-policy_access-restricted.png)

   You have now restricted access to your backend to one specific Front Door only.


# Resources
- [End-to-end TLS with Azure Front Door](https://docs.microsoft.com/en-us/azure/frontdoor/end-to-end-tls#backend-tls-connection-azure-front-door-to-backend):