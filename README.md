## How to setup docker-openvpn using docker-compose on Amazon linux 2

1. Create a AWS EC2 instance (t2.nano) with the user-data that installs git, docker, docker-compose and aws ssm agent.  
```shell
#!/bin/bash
sudo yum update -y

# install git
sudo yum install -y git

# install docker
sudo amazon-linux-extras install -y docker
sudo service docker start
sudo usermod -a -G docker ec2-user
sudo chkconfig docker on

# install docker-compose
sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

# install aws ssm agent
cd /tmp
sudo yum install -y https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
```

2. SSM into the EC2 instance (ssm:StartSession permission is required).
```shell
aws ssm start-session --target [ec2_instance_id] --profile [aws_profile]
```

3. Verify git, docker and docker-compose are installed.
```shell
git --version
docker --version
docker-compose --version
```

4. Create a docker-compose.yml under app directory.
```shell
mkdir app
cd app
vi docker-compose.yml
```

docker-compose.yml
```yml
version: '2'
services:
  openvpn:
    cap_add:
     - NET_ADMIN
    image: kylemanna/openvpn
    container_name: openvpn
    ports:
     - "1194:1194/udp"
    restart: always
    volumes:
     - ./openvpn-data/conf:/etc/openvpn
```

5. Initialize the configuration files and certificates.  
Make sure that your EC2 instance security group allows UDP 1194 port
```shell
docker-compose run --rm openvpn ovpn_genconfig -u udp://YOUR_EC2_PUBLIC_IP:1194
docker-compose run --rm openvpn ovpn_initpki
```


6. Fix ownership (depending on how to handle your backups, this may not be needed).  
```shell
sudo chown -R $(whoami): ./openvpn-data
```

7. Start OpenVPN server process
```shell
docker-compose up -d openvpn
```

8. Generate a client certificate
```shell
# with a passphrase (recommended)
docker-compose run --rm openvpn easyrsa build-client-full $CLIENTNAME
# without a passphrase (not recommended)
docker-compose run --rm openvpn easyrsa build-client-full $CLIENTNAME nopass
```

9. Retrieve the client configuration with embedded certificates
```shell
docker-compose run --rm openvpn ovpn_getclient $CLIENTNAME > $CLIENTNAME.ovpn
```

10. Revoke a client certificate.  
```shell
# Keep the corresponding crt, key and req files.
docker-compose run --rm openvpn ovpn_revokeclient $CLIENTNAME
# Remove the corresponding crt, key and req files.
docker-compose run --rm openvpn ovpn_revokeclient $CLIENTNAME remove
```

11. Install `tunnelblick` (Mac OS)
```shell
brew cask install tunnelblick
```

12. Drag and drop the downloaded `$CLIENTNAME.ovpn`
13. Connect VPN

### Tips
1. Multiple client connections.  
```shell
docker-compose run --rm openvpn vi /etc/openvpn/openvpn.conf
```
```shell
duplicate-cn
```

### Issues
1. block-outside-dns.  
https://github.com/kylemanna/docker-openvpn/issues/330#issuecomment-350983156

### References
- https://github.com/kylemanna/docker-openvpn
- https://elegantcoder.com/aws-openvpn-begins/
- https://medium.com/@gurayy/set-up-a-vpn-server-with-docker-in-5-minutes-a66184882c45
