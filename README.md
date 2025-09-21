# Kubernetes (k3s) on AWS EC2 - NGINX demo

This project is a tiny k3s cluster running on an EC2 instance in AWS. The app used is NGINX and will display a default web page which will be exposed to the internet over a port. A broken rollout will be simulated and fixed.


**Tools I used for the project**:
- Free tier AWS account
- One t3.small EC2 instance (appears to be the best free-tier instance for this project)
- Kubernetes (k3s)
- PowerShell with awscli v2 installed.
- VS Code (remotely SSH into the instance and make folders and files easier to see and work with)
- Git and Github account


<br>

## Section 1: **Create an ec2 instance and configure SSH access over SSM**
1. Create a t3.small instance in the preferred region. Ensure it has an SSH key attached(or create a new one) and has a public IP.

2. Ensure the instance is accessible via SSH. The instance can just be accessed over SSH via PuTTY with a private key, but I decided to take a different route here to help increase security. Instead of exposing port 22 over the internet in the security group, I opted to SSH to the instance over SSM. Steps I took are below:
    1. Removed port 22 from the inbound rules of the security group that the instance lives in.h
    2. Created IAM role for the EC2 instance and attached the "AmazonSSMManagedInstanceCore" permission to it so that it can be managed by SSM.
    3. Verify that the instance is being managed by SSM by checking that it shows up in Fleet Manager:
    <img width="2401" height="524" alt="image" src="https://github.com/user-attachments/assets/10546bd3-751b-44dc-a871-8a17fed411c2" />







     4. The private SSH key that I used for this instance was one that I had made previously. This key was saved on my machine in .ppk format. I converted it to .pem format using puttygen.

     5. From here I realized I will have to authenticate myself to my AWS account. For authentication, I enabled IAM Identitiy Center, created a user, created a group, put the user into the group, then gave the group permissions to my aws account. From there, via powershell on my machine, I ran the aws                configure sso command and went through the sso profile setup. Now I am able to log in to my aws account via sso:

        <br>

        **aws sso login via powershell**:
        <img width="1226" height="96" alt="image" src="https://github.com/user-attachments/assets/1cda4ace-7197-4b8a-af57-14f02beaf44c" />

        
        **browser opens and says I am authenticated**:
        <img width="753" height="456" alt="image" src="https://github.com/user-attachments/assets/348cc982-ab25-4d5a-bda4-086a54a1a762" />

  
        **Do a quick check if I can see the ec2 instance I created via aws cli**:
        <img width="1635" height="61" alt="image" src="https://github.com/user-attachments/assets/ac2446d5-a58a-4f87-948b-5390841e98de" />

        <br>


      7. Went into the \.ssh\config file on my Windows machine and added the below entry:

         ```Host (custom name for my ssh session)
             HostName (Instance ID) 
             User ubuntu
             IdentityFile C:\Path to my pem file\filename.pem
             IdentitiesOnly yes
             ProxyCommand C:\Windows\System32\cmd.exe /c aws ssm start-session --target %h --region (aws region) --profile (aws sso profile name) --document-name AWS-StartSSHSession --parameters "portNumber=%p"
         ```
    
      
      
      8. Verify that the ssh profile works, does it?

         <img width="548" height="25" alt="image" src="https://github.com/user-attachments/assets/40333c96-35fe-4630-8153-72d2890e3ca0" />



         Success!

         <img width="508" height="105" alt="image" src="https://github.com/user-attachments/assets/ecc855b3-6e6c-4903-bbb0-e26499232e82" />


    



<br><br><br><br>





## Section 2: **Get Kubernetes working on the ec2 instance with NGINX**
<br>
1. Now that I am able to access the instance via SSH over SSM, update the system and then install kubernetes:

```bash
sudo apt update -y
curl -sfL https://get.k3s.iso | sh -
```
<br><br>

Verify it's running with **kubectl get nodes**:

<img width="1094" height="82" alt="image" src="https://github.com/user-attachments/assets/22a24848-fad7-4388-939f-9391e05e4036" />


<br><br><br>
 2. Create manifests. I created a "deployment.yaml" and a "service.yaml". These files are in the repo.
<br><br><br>
 3. Apply the manifests
```bash
kubectl apply -f deployment.yaml
kubectl get pods #check output
kubectl apply -f service.yaml
kubectl get svc #check output
```
<br>
4. Confirm rollout

   ```bash
   kubectl rollout status deploy/<deployment-name>
   ```
<br>
<img width="1198" height="54" alt="image" src="https://github.com/user-attachments/assets/3fa994a5-f232-4a88-9668-be3447f69758" />
<br><br>
5. Check NGINX web page shows via the public dns name of my ec2 instance over port 30080:
<img width="1248" height="293" alt="image" src="https://github.com/user-attachments/assets/568597b5-1b1d-4e69-9e4e-aed61b614a8a" />
<br><br><br><br>


## Section 3: Simulate a broken rollout and fix.
<br>
1. Point the deployment to an invalid container image

   ```bash
   kubectl set image deployment/hello-nginx nginx=nginx:doesnotexist
   ```
<br>
<img width="1401" height="55" alt="image" src="https://github.com/user-attachments/assets/def1e3fb-3fea-43e4-8602-dd76c864e70a" />

<br><br>
Get pods after using an invalid image for the deployment to see status:
<img width="1201" height="106" alt="image" src="https://github.com/user-attachments/assets/8f3914ee-dc7e-45f5-a8d4-9065eab235c6" />


<br><br>
2. Debug using "descibe" and "logs" commands:

  ```bash
  kubectl describe pod <pod-name>
  kubectl logs <pod-name>
  ```
<img width="1996" height="58" alt="image" src="https://github.com/user-attachments/assets/44616269-4027-425e-a4f9-0a7e49db3d81" />

<br><br>
*Note, even though an invalid image got inserted, the website is still up. A new ReplicaSet was spun up using an invalid image while the old pod remained running:
<img width="1313" height="323" alt="image" src="https://github.com/user-attachments/assets/ab20a93d-f6f9-4c09-9d34-551fb4952168" />
<br><br>
3. Fix issue by using a valid image:

```bash
kubectl set image deployment/<name-of-my-deployment> nginx=nginx:1.25
kubectl rollout status deployment/<name-of-my-deployment>
```
<img width="1405" height="53" alt="image" src="https://github.com/user-attachments/assets/6dd33ebf-793c-4133-bef7-b176d4ec4501" />
<br>
<img width="1111" height="59" alt="image" src="https://github.com/user-attachments/assets/e193e9c0-5321-459b-9d22-cb643fe68dc2" />
<br><br><br><br>

## Section 4: Clean up
<br>
1. Delete resources and terminate server:

```bash
kubectl delete -f service.yaml -f deployment.yaml
```
<img width="1146" height="74" alt="image" src="https://github.com/user-attachments/assets/2c11ece6-5384-4a03-9bb1-b14f3278e56b" />
<br><br>
<img width="1251" height="199" alt="image" src="https://github.com/user-attachments/assets/7905ee09-e3af-47ba-9a1f-a391f897ad5d" />











