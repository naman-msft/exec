# SpringBootDemo
Spring Boot application that we will deploy to Kubernetes clusters in all 3 clouds

# Building and deploying the docker image: 
```bash
sudo apt-get install -y default-jdk maven
```
```bash
mvn clean install
```
```bash
mvn spring-boot:run  
```
```bash
docker build -t spring-boot-demo.jar .
docker run -p 9091:8080 spring-boot-demo.jar
```

# Deploying to VM

## Azure 

### Create and connect to the VM 
Set up parameter variables: 

```bash
export SUBSCRIPTION=ad70ac39-7cb2-4ed2-8678-f192bc4272b6 # customize this
export RESOURCE_GROUP=SpringBoot # customize this
export REGION=westus2 # customize this
export VM_NAME=springboot-vm
export VM_IMAGE=UbuntuLTS
export ADMIN_USERNAME=vm-admin-name # customize this
```

Log in and create VM: 

```bash
az group create --name ${RESOURCE_GROUP} --location ${REGION}
az vm create \
  --resource-group ${RESOURCE_GROUP} \
  --name ${VM_NAME} \
  --image ${VM_IMAGE} \
  --admin-username ${ADMIN_USERNAME} \
  --generate-ssh-keys \
  --public-ip-sku Standard --size standard_d4s_v3
```

Store the VM IP address for later: 

```bash
VM_IP_ADDRESS=`az vm show -d -g ${RESOURCE_GROUP} -n ${VM_NAME} --query publicIps -o tsv` 
```

Run the following to open port 8080 on the vm since SpringBoot uses it

```bash
az vm open-port --port 8080 --resource-group ${RESOURCE_GROUP} --name ${VM_NAME} --priority 1100
```

Connect to the VM: 

```bash
ssh ${ADMIN_USERNAME}@${VM_IP_ADDRESS}
```

### Deploy the application

Install Java and maven needed for application

```bash
sudo apt update
sudo apt install default-jdk
sudo apt install maven
```

Now it's time to clone the project into the vm and give it proper permissions: 

```bash
cd /opt
sudo git clone https://github.com/dasha91/SpringBootDemo
cd SpringBootDemo
sudo chmod -R 777 /opt/SpringBootDemo/
```

Run and deploy the app
```bash
mvn clean install
mvn spring-boot:run  
```

Finally go to http://[$VM_IP_ADDRESS]:8080 to confirm that it's working :D :D :D 