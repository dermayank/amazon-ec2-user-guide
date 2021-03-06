# Bring Your Own IP Addresses \(BYOIP\)<a name="ec2-byoip"></a>

You can bring part or all of your public IPv4 address range from your on\-premises network to your AWS account\. You continue to own the address range, but AWS advertises it on the internet\. After you bring the address range to AWS, it appears in your account as an address pool\. You can create an Elastic IP address from your address pool and use it with your AWS resources, such as EC2 instances, NAT gateways, and Network Load Balancers\.

## Requirements<a name="byoip-requirements"></a>
+ The address range must be registered with your regional internet registry \(RIR\), such as the American Registry for Internet Numbers \(ARIN\) or Réseaux IP Européens Network Coordination Centre \(RIPE\)\. It must be registered to a business or institutional entity and may not be registered to an individual person\.
+ The most specific address range that you can specify is /24\.
+ You can bring each address range to one region at a time\.
+ You can bring 5 address ranges per region to your AWS account\.
+ The addresses in the IP address range must have a clean history\. We may investigate the reputation of the IP address range and reserve the right to reject an IP address range if it contains an IP address that has poor reputation or is associated with malicious behavior\.

## Prepare to Bring Your Address Range to Your AWS Account<a name="prepare-for-byoip"></a>

To ensure that only you can bring your address range to your AWS account, you must authorize Amazon to advertise the address range and provide proof that you own the address range\.

A Route Origin Authorization \(ROA\) is a document that you can create through your RIR\. It contains the address range, the ASNs that are allowed to advertise the address range, and an expiration date\. An ROA authorizes Amazon to advertise an address range under a specific AS number\. However, it does not authorize your AWS account to bring the address range to AWS\. To authorize your AWS account to bring an address range to AWS, you must publish a self\-signed X509 certificate in the RDAP remarks for the address range\. The certificate contains a public key, which AWS uses to verify the authorization\-context signature that you provide\. You should keep your private key secure and use it to sign the authorization\-context message\.

The commands in the following procedure require OpenSSL version 1\.0\.2 or later\.

**To prepare to bring your address range to your AWS account**

1. Create an ROA to authorize Amazon ASNs 16509 and 14618 to advertise your address range\. It might take up to 24 hours for the ROA to become available to Amazon\. For more information, see the following:
   + ARIN — [ROA Requests](https://www.arin.net/resources/rpki/roarequest.html)
   + RIPE — [Managing ROAs](https://www.ripe.net/manage-ips-and-asns/resource-management/certification/resource-certification-roa-management)

1. Generate an RSA 2048\-bit key pair as follows:

   ```
   openssl genrsa -out private.key 2048
   ```

1. Create a public X509 certificate from the key pair using the following command\. In this example, the certificate expires in 365 days, after which time it cannot be trusted\. Therefore, be sure to set the expiration appropriately\. When prompted for information, you can accept the default values\.

   ```
   openssl req -new -x509 -key private.key -days 365 | tr -d "\n" > publickey.cer
   ```

1. Create a signed message\. The format of the message is as follows, where the date is the expiry date of the message:

   ```
   1|aws|account|cidr|YYYYMMDD|SHA256|RSAPSS
   ```

   The following command signs the message using the key pair you created and saves it as `base64_urlsafe_signature`:

   ```
   echo "1|aws|123456789012|198.51.100.0/24|20191201|SHA256|RSAPSS" | tr -d "\n" | openssl dgst -sha256 -sigopt rsa_padding_mode:pss -sigopt rsa_pss_saltlen:-1 -sign private.key -keyform PEM | openssl base64 | tr -- '+=/' '-_~' | tr -d "\n" > base64_urlsafe_signature
   ```

1. Update the RDAP record for your RIR with the X509 certificate\. Be sure to copy the `-----BEGIN CERTIFICATE-----` and `-----END CERTIFICATE-----` from the certificate\. To view your certificate, run the following command:

   ```
   cat publickey.cer
   ```

   For ARIN, add the certificate in the "Public Comments" section for your address range\.

   For RIPE, add the certificate as a new "desc" field for your address range\.

## Provision the Address Range for use with AWS<a name="byoip-provision"></a>

When you provision an address range for use with AWS, you are confirming that you own the address range and authorizing Amazon to advertise it\. We will also verify that you own the address range\.

Use the following [provision\-byoip\-cidr](https://docs.aws.amazon.com/cli/latest/reference/ec2/provision-byoip-cidr.html) command to provision the address range\. The message in the `--cidr-authorization-context` parameter is the signed message that you created in the previous section\.

```
aws ec2 provision-byoip-cidr --cidr address-range --cidr-authorization-context Message="message",Signature="signature"
```

Provisioning an address range is an asynchronous operation, so the call returns immediately, but the address range is not ready to use until its status changes from `pending-provision` to `provisioned`\. To monitor the status of the address range, use the following [describe\-byoip\-cidrs](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-byoip-cidrs.html) command:

```
aws ec2 describe-byoip-cidrs --max-results 5
```

## Advertise the Address Range through AWS<a name="byoip-advertise"></a>

After the address range is provisioned, it is ready to be advertised\. We recommend that you stop advertising the address range from other locations before you advertise it through AWS\. If you keep advertising your IP address range from other locations, we can't reliably support it\. Specifically, we couldn't guarantee that traffic to the address range will enter our network and it would affect our ability to troubleshoot issues\.

To minimize down time, you can configure your AWS resources to use an address from your address pool before it is advertised, and then simultaneously stop advertising it from the current location and start advertising it through AWS\. For more information about allocating an Elastic IP address from your address pool, see [Allocating an Elastic IP Address](elastic-ip-addresses-eip.md#using-instance-addressing-eips-allocating)\.

Use the following [advertise\-byoip\-cidr](https://docs.aws.amazon.com/cli/latest/reference/ec2/advertise-byoip-cidr.html) command to advertise the address range:

```
aws ec2 advertise-byoip-cidr --cidr address-range
```

**Important**  
You can run the advertise\-byoip\-cidr command at most once every 10 seconds, even if you specify different address ranges each time\.

If you want to stop advertising the address range, use the following [withdraw\-byoip\-cidr](https://docs.aws.amazon.com/cli/latest/reference/ec2/withdraw-byoip-cidr.html) command:

```
aws ec2 withdraw-byoip-cidr --cidr address-range
```

**Important**  
You can run the withdraw\-byoip\-cidr command at most once every 10 seconds, even if you specify different address ranges each time\.

## Deprovision the Address Range<a name="byoip-deprovision"></a>

If you no longer want to use your address range with AWS, you can stop advertising the address range and then deprovision it\. You must release all Elastic IP addresses you allocated from the address pool first\.

Use the following [withdraw\-byoip\-cidr](https://docs.aws.amazon.com/cli/latest/reference/ec2/withdraw-byoip-cidr.html) command to stop advertising the address range:

```
aws ec2 withdraw-byoip-cidr --cidr address-range
```

Use the following [deprovision\-byoip\-cidr](https://docs.aws.amazon.com/cli/latest/reference/ec2/deprovision-byoip-cidr.html) command to deprovision the address range:

```
aws ec2 deprovision-byoip-cidr --cidr address-range
```