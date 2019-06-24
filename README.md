### Instructions

Create a **Spark 2.3.1 notebooks & Jupyterhub with SSL & centos7x** cluster.

Create credentials for logging in to Jupyterhub: https://stackoverflow.com/questions/56725570/how-to-create-credentials-for-jupyterhub

If you are using https://github.com/bluedata-community/bluedata-demo-env-aws-terraform, you can provision a Vault instance on EC2 using:

```
/******************* Instance: Vault ********************/

resource "aws_instance" "vault" {
  ami                    = "${var.ec2_ami}"
  instance_type          = "t2.micro"
  key_name               = "${aws_key_pair.main.key_name}"
  vpc_security_group_ids = [ "${aws_default_security_group.main.id}" ]
  subnet_id              = "${aws_subnet.main.id}"

  root_block_device {
    volume_type = "gp2"
    volume_size = 50
  }

  tags = {
    Name = "${var.project_id}-instance-vault"
    Project = "${var.project_id}"
    user = "${var.user}"
  }

  provisioner "remote-exec" {
    connection {
      type  = "ssh"
      user  = "centos"
      host  = "${aws_instance.vault.public_ip}"
      timeout = "10m"
      private_key = "${file(var.ssh_prv_key_path)}"
      agent = false
    }
    inline = [<<EOF
sudo yum install unzip -y
VAULT_VERSION="1.1.2"
curl -sO https://releases.hashicorp.com/vault/$${VAULT_VERSION}/vault_$${VAULT_VERSION}_linux_amd64.zip
unzip vault_$${VAULT_VERSION}_linux_amd64.zip
sudo mv vault /usr/local/bin/
vault -autocomplete-install
complete -C /usr/local/bin/vault vault
sudo mkdir /etc/vault
sudo mkdir -p /var/lib/vault/data
sudo useradd --system --home /etc/vault --shell /bin/false vault
sudo chown -R vault:vault /etc/vault /var/lib/vault/
cat <<EOF2 | sudo tee /etc/systemd/system/vault.service
[Unit]
Description="HashiCorp Vault - A tool for managing secrets"
Documentation=https://www.vaultproject.io/docs/
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/vault/config.hcl
[Service]
User=vault
Group=vault
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
SecureBits=keep-caps
AmbientCapabilities=CAP_IPC_LOCK
NoNewPrivileges=yes
ExecStart=/usr/local/bin/vault server -config=/etc/vault/config.hcl
ExecReload=/bin/kill --signal HUP 
KillMode=process
KillSignal=SIGINT
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
StartLimitBurst=3
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF2
sudo touch /etc/vault/config.hcl
cat <<EOF2 | sudo tee /etc/vault/config.hcl
disable_cache = true
disable_mlock = true
ui = true
listener "tcp" {
   address          = "0.0.0.0:8200"
   tls_disable      = 1
}
storage "file" {
   path  = "/var/lib/vault/data"
 }
api_addr         = "http://0.0.0.0:8200"
max_lease_ttl         = "10h"
default_lease_ttl    = "10h"
cluster_name         = "vault"
raw_storage_endpoint     = true
disable_sealwrap     = true
disable_printable_check = true
EOF2
sudo systemctl daemon-reload
sudo systemctl enable --now vault
systemctl status vault
export VAULT_ADDR=http://127.0.0.1:8200
echo "export VAULT_ADDR=http://127.0.0.1:8200" >> ~/.bashrc
sudo rm -rf  /var/lib/vault/data/*
vault operator init | sudo tee /etc/vault/init.file
EOF
  	]
  }
}

output "vault_ssh_command" {
  value = "ssh -o StrictHostKeyChecking=no -i ${var.ssh_prv_key_path} centos@${aws_instance.vault.public_ip}"
}

output "vault_private_ip" {
  value = "${aws_instance.vault.private_ip}"
}
output "vault_private_dns" {
  value = "${aws_instance.vault.private_dns}"
}
output "vault_public_ip" {
  value = "${aws_instance.vault.public_ip}"
}
output "vault_public_dns" {
  value = "${aws_instance.vault.public_dns}"
}
```

SSH into the vault server and use the vault CLI to configure the cluster:

```
####
#### first unseal
####

cat /etc/vault/init.file

vault operator unseal <enter unseal key 1>
vault operator unseal <enter unseal key 2>
vault operator unseal <enter unseal key 3>

export VAULT_ROOT_TOKEN=<enter root token>

####
#### now setup
####

vault -version

vault login $VAULT_ROOT_TOKEN

vault auth enable userpass

vault secrets enable -version=2 -path=secret kv

vault policy write my-policy -<<EOF

path "secret/data/*" {
  capabilities = ["create", "update", "read", "delete"]
}
path "secret/data/azure_datalake" {
  capabilities = ["create", "read", "update", "delete"]
}
EOF

vault token create -policy=my-policy

#
# create user
#

vault write auth/userpass/users/chris \
    password=password \
    policies=my-policy,default
    
vault kv put secret/azure_datalake accountName=my_azure_storage
vault kv put secret/azure_datalake accountKey=my_azure_storage_account_key
```

Create a storage container called `mycontainer`

Add the following CSV file to your azure storage container: 

- [boston_house_prices.csv](./boston_house_prices.csv) - note this CSV was adapted from https://github.com/scikit-learn/scikit-learn/blob/master/sklearn/datasets/data/boston_house_prices.csv

Now run the following notebook on the cluster:

- [vault_example.ipynb](https://nbviewer.jupyter.org/github/snowch/bluedata_notebook_with_hashicorp_vault_demo/blob/master/vault_example.ipynb)

### TODO

Setup Vault to run inside BlueData - see:

 - https://hub.docker.com/_/vault
 - https://stackoverflow.com/questions/56731980/how-do-i-pass-docker-parameters-such-as-cap-add-xxx-to-my-docker-instances-r
