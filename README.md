# EC2 NAT

Turn a regular old AWS EC2 instance into a NAT instance for one or more instances in your private subnet.

1. **Launch an instance**

   - Use Amazon Linux 2023 for the sake of this tutorial.
   - When selecting your VPC, ensure you place the instance in a public subnet. You will SSH into the instance in a later step.

3. **Alter Network Interface Settings**

    From the EC2 console, navigate to the instance's Elastic Network Interface details. This can be done from the sidebar or from the instance details in the networking tab.

    &#8594; Disable Source/Destination Check

4. **Install iptables**

    - SSH into your instance.
    ```sh
    sudo yum install -y iptables-services
    sudo iptables -t nat -A POSTROUTING -o ens5 -j MASQUERADE
    sudo bash -c 'iptables-save > /etc/sysconfig/iptables'
    ```
5. **Alter sysctl file**

    ```sh
    sudo vim /etc/sysctl.conf
    ```

    Append to `/etc/sysctl.conf`:

    ```ini
    net.ipv4.ip_forward = 1
    net.ipv4.conf.all.send_redirects = 0
    ```

    Run:
    ```sh
    sudo sysctl -p /etc/sysctl.conf
    ```

    Exit the instance.

6. **Alter Security Group**

    - From instance details, navigate to its security group in the Security tab.

    - Inbound rules:
      - Only allow traffic from security groups of the instance(s) in your private subnet. 
      - You can also do this by private IP of the private subnet instance(s), technically. 
      - Unless you're certain what type of traffic you will need to allow outbound from your private subnet instance(s), allow all traffic for now.

    - Outbound rules:
      - Allow all traffic outbound (to the internet).

7. **Update Route Table**

    - Navigate to the route table for your VPC.
      - VPC &#8594; Route tables &#8594; rtb-[your-private-subnet]
      - Edit routes &#8594; Add route
      - Destination: `0.0.0.0/0`
      - Target: `the elastic network interface id of the NAT instance`

8. **Test your private instance(s)**

    This step assumes you have set up your private instance in a way that you can SSH into it from an instance in one of your public subnets.

    For example:
    - SSH into your NAT instance, then SSH into your private instance.

    Assuming you're able to access your private instance, you can test its connection to the internet (via the NAT instance) in various ways. Here are two examples:

    - Install new packages on it. For example:
        ```sh
        sudo yum install golang -y
        ```
    - Test with `curl`:
        ```sh
        curl -I http://www.google.com
        ```

    If you're able to do either of these, you're all set. 
    
    Now, you can:
    - setup a custom CI/CD pipeline with GitHub and bash scripting.
    - setup a new systemd service to run your Go [or other compiled] application, which will be able to reach the internet / make calls to other services outside your VPC.
    - save a ton of money by not using a managed AWS NAT Gateway.
