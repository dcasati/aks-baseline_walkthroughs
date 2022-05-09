# Add NSG rule to allow incoming traffic on port 80.

1. Browse to NSG attached to the subnet hosting the Application Gateway.
   ![](img/z01_app-gw_add-inbound-nsg-rule.png)

1. Configure a new inbound traffic rule.

   ![](img/z02_app-gw-nsg_add-inbound-from-any.png)

#  Add Application Gateway Listener for port 80.

1. Browse to your Application Gateway.

1. Add a new Listener for port 80. 

   ![](img/z03_app-gw_listener-port-80.png)

1. Add a new Request Routing Rule

   ![](img/z04_app-gw_request-routing-rule-1.png)

   ![](img/z05_app-gw_request-routing-rule-2.png)
   
1. Open `http://bicycle.leho.com/` to verify the Application Gateway well serves responses on http/80.

   ![](img/z06_app-gw_test-request.png)