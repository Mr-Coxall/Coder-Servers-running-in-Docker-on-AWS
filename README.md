# Coder Servers running in Docker on AWS

- These are the steps to get multiple Coder IDE Servers up and running on an AWS EC2 machine, using docker and Nginx Proxy Manager and your own domain names.

## AWS EC2 Security Group
- create a new security group with SSH, HTTP, HTTPS & 9443 (for portainer) access
- ![AWS Security Group](./images/AWS_security_group.png)

## AWS EC2 Instance
- create an EC2 instance
  -  Debian
  -  Architecture - 64-bit (Arm) <- I pick this because it is cheaper
  -  Instance type - t4g.medium (could be smaller or larger)
  -  use your existing or create a key pair (to SSH into machine)
  -  Network settings - Select existing security group and pick the one you created above
  -  Configure storage - the amount you want
 
## AWS EC2 Elastic IPs
- create a new elastic IP, so you have a static IP address
  - Allocate Elastic IP address
  - Associate you your instance
  - ![AWS Elastic_IP](./images/AWS_Elastic_IP.png)

## Connect to EC2 Instance
- ssh into your instance
  - ![AWS Elastic_IP](./images/AWS_EC2_Instance_SSH.png)
- ``` BASH
  sudo apt update
  sudo apt upgrade -y
  ```

## Install Docker
- install docker onto the instance
  - see: https://docs.docker.com/engine/install/debian/
- ``` BASH
  sudo apt-get update
  sudo apt-get install ca-certificates curl
  sudo install -m 0755 -d /etc/apt/keyrings
  sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
  sudo chmod a+r /etc/apt/keyrings/docker.asc

  echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  sudo apt-get update

  sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

  sudo docker run hello-world
  ```

## Install Portainer CE (Community Edition)
- install portainer
  - https://docs.portainer.io/start/install-ce/server/docker/linux
- ``` BASH
  sudo docker volume create portainer_data
  sudo docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:2.21.4
  ```
- login to portainer (https://xx.xx.xx.xx:9443)
  - where xx.xx.xx.xx = Elastic IP
  - accept that the connection has a "self-signed certificate"
  - create a user with password

## Add a Coder Stact
- install a Coder docker instance
  - https://coder.com/docs/install/docker
  - ``` BASH
    version: "3.9"
    services:
      coder:
        # This MUST be stable for our documentation and
        # other automations.
        image: ghcr.io/coder/coder:${CODER_VERSION:-latest}
        # restart automatically
        #???
        ports:
          - "7080:7080"
        environment:
          CODER_PG_CONNECTION_URL: "postgresql://${POSTGRES_USER:-username}:${POSTGRES_PASSWORD:-password}@database/${POSTGRES_DB:-coder}?sslmode=disable"
          CODER_HTTP_ADDRESS: "0.0.0.0:7080"
          # You'll need to set CODER_ACCESS_URL to an IP or domain
          # that workspaces can reach. This cannot be localhost
          # or 127.0.0.1 for non-Docker templates!
          CODER_ACCESS_URL: "${CODER_ACCESS_URL}"
        # If the coder user does not have write permissions on
        # the docker socket, you can uncomment the following
        # lines and set the group ID to one that has write
        # permissions on the docker socket.
        #group_add:
        #  - "998" # docker group on host
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
        depends_on:
          database:
            condition: service_healthy
      database:
        # Minimum supported version is 13.
        # More versions here: https://hub.docker.com/_/postgres
        image: "postgres:16"
        # restart automatically
        #????
        ports:
          - "5432:5432"
        environment:
          POSTGRES_USER: ${POSTGRES_USER:-username} # The PostgreSQL user (useful to connect to the database)
          POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-password} # The PostgreSQL password (useful to connect to the database)
          POSTGRES_DB: ${POSTGRES_DB:-coder} # The PostgreSQL default database (automatically created at first launch)
        volumes:
          - coder_data:/var/lib/postgresql/data # Use "docker volume rm coder_coder_data" to reset Coder
        healthcheck:
          test:
            [
              "CMD-SHELL",
              "pg_isready -U ${POSTGRES_USER:-username} -d ${POSTGRES_DB:-coder}",
            ]
          interval: 5s
          timeout: 5s
          retries: 5
    volumes:
      coder_data:
    ```
