# autoca

This repository serves as a primer for setting up an internal certificate authority (CA) that issues x.509 certificates in an automated way. Luckily we can dispense with most of the complexity of the ACME protocol that is now the de-facto standard for certificate issuance in the internet. However the degree to which the configuration examples from this repository cover your use case and PKI security requirements must be carefully reviewed. The following assumptions apply:

- you use a systemd-enabled Linux operating system
- **openssl** and **rsync** are installable on both the server and all clients
- you control all clients and as such clients can be trusted

## Idea

Clients check on a regular basis if they possess a certificate in a certain place. Unless such a certificate exists and it is still valid for the foreseeable future, a new private key and certificate signing requests (CSR) is created. The CSR is sent via `ssh` to a server that controls the CA.

To that end the server saves the client's public key to an `authorized_keys` file, effectively authorizing the client to submit CSRs and receive a certificate. The server is configured to sign incoming CSRs immediately and write the newly issued certificate to the same place, removing the CSR in the process.

Then the client can reconnect to the server and fetch the certificate. Once the certificate is available on the client it can be copied to the proper place, e.g. where it is read by a webserver to serve content via TLS.

## Configuration

Assumptions:

- the server that controls the CA is called `server.internal`
- the example client is called `client.internal`

### Server

First and foremost the server needs a x.509 certificate authority that can be used to issue certificates. While this may sound ominous a certificate authority is nothing more than a certificate with some special attribute and is trusted by all parts of your infrastructure in order to encrypt traffic using certificates signed by this authority.

Run the following command to create a CA:

```sh
sudo mkdir -p /root/x509
sudo openssl req -outform pem -out /root/x509/ca.crt -newkey ec -pkeyopt ec_paramgen_curve:secp384r1 -keyform pem -keyout /root/x509/ca.key -subj "/CN=Internal Certificate Authority" -addext "keyUsage=critical,digitalSignature,keyEncipherment,nonRepudiation,keyCertSign" -addext "basicConstraints=CA:TRUE,pathlen:0" -x509 -days 1095
```

Save the passphrase for the private key to a file in the same directory called `/root/x509/passphrase`.

Refer to `man x509v3_config` for details on key usages and constraints.

Create a system user that clients use to connect to the server and supply CSRs:

```sh
sudo adduser --system --group --home /home/requests --shell /bin/bash requests
sudo mkdir /home/requests/x509
sudo chown requests: /home/requests/x509
```

Create the file `home/requests/.ssh/authorized_keys`. It must contain the public key of every client and restrict the client to a specific action. The following row serves as an example and must be repeated for every client:

```
restrict,command="rrsync -no-del /home/requests/x509" ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKqs1j90cLP/Fy6Nt8ciKT3M/E19Egmk5+71zpBA7K4i
```

Now clients are able to submit CSRs by copying them to the server and into the specified directory. The server uses the following two systemd template units to listen for incoming requests and sign them subsequently:

```
# /etc/systemd/system/sign-certificate-request@.path
[Unit]
Description=Watch for incoming x.509 certificate request for %i

[Path]
PathExists=/home/requests/x509/%i.csr

[Install]
WantedBy=multi-user.target
```

This is a path unit that triggers a service unit by the same name (except the suffix) when a file is created at the given path. This is the corresponding service unit:

```
# /etc/systemd/system/sign-certificate-request@.service
[Unit]
Description=Sign x.509 certificate request for %i
ConditionFileIsExecutable=/usr/bin/openssl
ConditionPathExists=/home/requests/x509/%i.csr

[Service]
Type=oneshot
WorkingDirectory=/home/requests/x509
ExecStart=/usr/bin/echo "Creating new certificate for %i"
ExecStart=/usr/bin/openssl req -outform pem -out %i.crt -in %i.csr -x509 -days 90 -CA /root/x509/ca.crt -CAkey /root/x509/ca.key -passin file:/root/x509/passphrase -copy_extensions copy -addext "basicConstraints=CA:FALSE"
ExecStart=/usr/bin/chown requests: %i.crt
ExecStart=/usr/bin/echo "Removing certificate request"
ExecStart=/usr/bin/rm %i.csr
```

The service unit uses the CA to sign the request, thereby issuing a new certificate, and remove the CSR. The service unit is a *static* unit, meaning it can only be triggered by other units or manually started. It is supposed to be started by the path unit only. The path unit needs to be instantiated for every client. For example (for a client called *client.internal*):

```sh
sudo systemctl enable sign-certificate-request@client.internal.path
sudo systemctl start sign-certificate-request@client.internal.path
```

The path unit is now triggered whenever a client called *client.internal* submits a CSR at the given path.

### Client

The example client is called *client.internal*. The steps below must be repeated for every client.

For the sake of simplicity we use the **root** user to manage the certificate lifecycle. This could of course be any other user in production. Create a new directory for private keys, CSRs and certificates. Also generate a new ssh key pair:

```sh
sudo mkdir ~/x509
sudo ssh-keygen -t ed25519
```

The ssh public key must be copied to the server as part of a new entry in the file `/home/requests/.ssh/authorized_keys` as mentioned in the section above.

The certificate issuance and lifecycle logic is implemented with systemd template units:

- a timer and service unit that check if a certificate for the specified host exists
- a timer and service unit that check if the host certificate is still valid for the foreseeable future
- a service unit to create a new private key and CSR and submit it to the server via `rsync`
- a service unit to fetch the new certificate shortly after via `rsync`

Create the systemd units:

```
# /etc/systemd/system/check-certificate-existence@.timer
[Unit]
Description=Check x.509 certificate existence

[Timer]
OnActiveSec=1
OnCalendar=daily

[Install]
WantedBy=multi-user.target
```

```
# /etc/systemd/system/check-certificate-existence@.service
[Unit]
Description=Check x.509 certificate existence
ConditionPathExists=!/root/x509/%i.crt
OnSuccess=create-certificate-request@%i.service

[Service]
Type=oneshot
ExecStart=/usr/bin/echo "certificate does not exist, triggering renewal"
```

```
[Unit]
Description=Check x.509 certificate validity

[Timer]
OnActiveSec=1
OnCalendar=daily

[Install]
WantedBy=multi-user.target
```

```
# /etc/systemd/system/check-certificate-validity@.service
[Unit]
Description=Check x.509 certificate validity
ConditionPathExists=/root/x509/%i.crt
ConditionFileIsExecutable=/usr/bin/openssl
OnFailure=create-certificate-request@%i.service

[Service]
Type=oneshot
ExecStart=/usr/bin/openssl x509 -in /root/x509/%i.crt -checkend 604800
```

```
# /etc/systemd/system/create-certificate-request@.service
[Unit]
Description=Create and submit new x.509 certificate request
ConditionFileIsExecutable=/usr/bin/openssl
ConditionFileIsExecutable=/usr/bin/rsync
OnSuccess=fetch-certificate@%i.service

[Service]
Type=oneshot
WorkingDirectory=/root/x509
ExecStart=/usr/bin/openssl req -outform pem -out %i.csr \
  -newkey ec -pkeyopt ec_paramgen_curve:secp384r1 -keyform pem -keyout %i.key -noenc \
  -subj "/CN=%i" -addext "subjectAltName=DNS:client.internal" \
  -addext "keyUsage=critical,digitalSignature,keyEncipherment,nonRepudiation" -addext "extendedKeyUsage=critical,serverAuth" -addext "basicConstraints=CA:FALSE" 
ExecStart=/usr/bin/openssl req -in %i.csr -noout -text
ExecStart=/usr/bin/rsync -pvz %i.csr requests@server.internal:%i.csr
```

```
# /etc/systemd/system/fetch-certificate@.service
[Unit]
Description=Fetch signed x.509 certificate
ConditionFileIsExecutable=/usr/bin/rsync

[Service]
Type=oneshot
WorkingDirectory=/root/x509
ExecStart=/usr/bin/sleep 1
ExecStart=/usr/bin/rsync -pvz requests@server.internal:%i.crt .
```

The service units are all static units, meaning they are supposed to be triggered by the timer units, other service units or manually via `systemctl start`. Enable and start the timer units:

```sh
FQDN=$(hostname --fqdn)

sudo systemctl enable check-certificate-existence@$FQDN.timer
sudo systemctl start check-certificate-existence@$FQDN.timer

sudo systemctl enable check-certificate-validity@$FQDN.timer
sudo systemctl start check-certificate-validity@$FQDN.timer
```

Trigger the service unit that creates and submits the CSR manually to retrieve the first certificate:

```
sudo systemctl start create-certificate-request@$FQDN.service
```
