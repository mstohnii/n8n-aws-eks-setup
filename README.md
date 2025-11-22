---
The deployment of the n8n cluster was initially done according to the official instructions, but some important key points were not covered, so here are my personal patched instructions on how to do it the right way. (ps. link to the official instruction, so you can compare https://docs.n8n.io/hosting/installation/server-setups/aws/)

---
## Step 1: (Creating a cluster)
After installing all the prerequisites we need to create a cluster, so we need to open our cli and enter the following command 
```cmd 
eksctl create cluster --name n8n --region <your-aws-region>
```
## Step 2: (Avoiding time waste)
To avoid wasting time while cluster is setting up (the setup process may take more than 15min.), we can prepare everything else:
### Step 2.1:
 In our case, we use the domain `njordops.net` to access n8n. First, let's issue him an SSL certificate. 
- To do this, we need to go to AWS Certificate Manager and click the `request` button.
- Enter the domain details, select DNS verification, and leave the default encryption algorithm.
- The next step is to go to our domain registrar's website and open the control panel (in this case, the domain njordops.net is registered with Cloudflare).
- Go to our domain management page ---> DNS --> Records
- Return to the certificate page and copy the `CNAME name` and `CNAME value`. Then switch back to the DNS settings tab, click Add record, select the CNAME type, and enter the copied values in the appropriate fields. Within a few minutes, our domain will be confirmed and the certificate status will be listed as `Issued`.


### Step 2.2:
The certificate is issued, now we can move on to the manifests. The `kubernetes` folder contains the official manifests that I have modified. You can read the list of changes in the README file. 
## Step 3: (Dealing with problems before they happen)
By the time the **second step** is complete, eksctl will finish creating the cluster. 
Now, we need to patch it up with our hands, because when it was created, it didn't have some of the settings we need. 
### Step 3.1: (EKS Auto Mode)
The auto mode is initially turned off, so let's turn it on.
- Go to the settings of your AWS cluster and enable EKS Auto Mode (during the configuration it might require certain permissions for its role, so copy the names of the policies and add them to its role). Wait for it to turn on.
### Step 3.2: (EBS CSI Driver)
The next problem is that the Amazon EBS CSI Driver add-on is missing, so let's install it. (Link to the official guide https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)
Through trial and error, the easiest way for me was to create a role using eksctl and then install and assign the role by adding add-ons on the cluster page on the AWS website.
- First, let's create a role for our add-on.
``` eksctl
eksctl create iamserviceaccount --name ebs-csi-controller-sa --namespace kube-system --cluster <CLUSTER_NAME> --role-name AmazonEKS_EBS_CSI_DriverRole --role-only --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy --approve
```
- Then on our cluster page, go to the Add-ons tab and click Get more add-ons. Find the EBS CSI Driver add-on and proceed to the next stage. There, select the role we created and click **Create**. Wait for it to turn on.
### Step 3.2: (Setting default Storage Class)
If we leave everything as it is, Storage Class will not be assigned, so let's make it default.
``` cmd
kubectl patch storageclass gp2 -p "{\"metadata\": {\"annotations\": {\"storageclass.kubernetes.io/is-default-class\": \"true\"}}}"
```
## Step 4:
### Step 4.1: 
I almost forgot, before deployment, don't forget to check that the ARN of our certificate matches the one you enter in n8n-service.yaml (you can find it on our certificate page where we took the CNAME data for the domain registrar).
### Step 4.2:
Finally, we have sorted out the problems and can now proceed with the deployment of n8n.
- Enter the `kubernetes` folder via terminal and apply manifests:
```
kubectl apply -f namespace.yaml
```
and than
```
kubectl apply -f .
```
(or you can apply second command twice `;)` BTW)

Using Lens, you can monitor the progress of actions on the cluster (in the events tab). If any issues arise, they will also be displayed there (most of the possible ones).
### Step 4.3: 

- Let's go back to our AWS EKS page. 
- Go to the `Resources`, than open the category `Service and networking` --> `Services` and go straight to the `n8n` service.
- Сheck whether the ports we need are open. If not, open them in the security group.
- Copy `Load balancer URLs` (manually, not via redirect button)  

- Go back to our Cloudflare DNS tab in browser and add a new record. Select CNAME as the type, than enter the `name` of our domain or leave `@` in the *name* field, and enter the **Load balancer URL** in the *value* field. Click **save**.
## Step 5: FINAL
In a few minutes, everything will be up and running and will work via the https protocol.

Notes: The work was completed approximately two weeks prior to the writing of this instruction. Please note that this instruction was written without the use of AI and entirely from memory (with the exception of two commands—creating a role for the EBS driver and setting the default Storage Class, for these two cases, the official guides for them was used)

