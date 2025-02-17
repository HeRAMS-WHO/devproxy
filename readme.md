# DevProxy V2

This is a default configuration that will:
- Set up Smallstep CA as a local ACME server
- Set up docker-proxy to securely allow access to your docker socket
- Set up traefik to use the docker-proxy and ACME server for serving you local containers

Set up steps:
1. Clone this repo
2. Create the network for devproxy: `docker network create devproxy2`
3. Run `docker-compose run --rm bootstrap`
4. Run `docker-compose up --pull -d traefik` it will automatically be restarted on boot
5. `docker compose cp step-ca:/home/step/certs/root_ca.crt /usr/local/share/ca-certificates/devproxyv2.crt`
6. `sudo update-ca-certificates`
7. Install the generated CA certificate in your browser (it's located in `/usr/local/share/ca-certificates/devproxyv2.crt`)
8. Configure DNS
9. Your CLI should now support the certificate: `curl https://traefik.devproxy.test` should show **Moved permanently**
10. Go to `https://traefik.devproxy.test` and you should see the container working.


## Configuring DNS
For local development the `.test` domain should be used (as it is reserved for that in the relevant RFCs).

### Windows
If you are developing on Windows then you should probably move to Linux.
If that's not an option then your best bet is probably to update your hosts file for every new test project.
Alternatively you could configure your dns server in windows to be `localhost:53535` this will result in a bad experience though since your DNS won't work after login until your docker VM has booted.
You could also install your own DNS server and configure that as defined below...

### Linux
On Linux it is easy to simply reroute your DNS, we include a simple dnsmasq server that is used in internally by the CA to
resolve the domains to the Traefik container. It is also exposed on `localhost:53535`.
By adding this to a dnsmasq config file:
```
server=/.test/127.0.0.1#53535
```
You can do this by running: `echo server=/.test/127.0.0.1#53535 | sudo tee /etc/NetworkManager/dnsmasq.d/test.conf`

All requests for the `.test` tld will be forwarded to our DNS container, which will resolve it to the traefik container.
Because bridge networks are accessible from the host we do not need to expose any ports on the Traefik container. Technically we don't even need to expose the dns server on localhost, were it not for the fact that we don't know its container IP (and the fact that this container IP is not static).

Note that if your system is not using dnsmasq you might have to activate that first. For Ubuntu check out this tutorial: https://anto.online/guides/how-to-enable-and-disable-systemd-resolved-on-ubuntu/
Note that the this can be also done as:

```
We need to enable dnsmasq
cd /etc/NetworkManager/
sudo nano NetworkManager.conf 
add "dns=dnsmasq" to [main]

restart NetworkManager
sudo systemctl restart NetworkManager
check systemd
systemctl status systemd-resolved

verify if the resolv has now dns enabled
cat /etc/resolv.conf
in case it doesn't
systemctl disable systemd-resolved
systemctl stop systemd-resolved
systemctl status systemd-resolved

verify again
cat /etc/resolv.conf

Restart the NetworkManager
sudo systemctl restart NetworkManager
cat /etc/resolv.conf

if it still didnt work
sudo rm /etc/resolv.conf 
create a new simlink
sudo ln -s /run/NetworkManager/resolv.conf /etc/resolv.conf

check if the /etc/resolv has the correct simlink
ls -la

check the networkManager status
systemctl status NetworkManager

if the /run/NetworkManager/resolv.conf still doesnt have the dns enabled
sudo nano /run/NetworkManager/resolv.conf 
sudo rm /etc/resolv.conf 
sudo ln -s /run/NetworkManager/resolv.conf /etc/resolv.conf

you can try the above command
echo server=/.test/127.0.0.1#53535 | sudo tee /etc/NetworkManager/dnsmasq.d/test.conf

after the dns is enabled you can check with
nslookup something.test
nslookup herams.test
you restart the NetworkManager one more time to make sure its still working
sudo systemctl restart NetworkManager
```

## Configuring routing on windows
On Linux most things work out of the box, on Windows additional magic will be needed. You'll probably want to create a `docker-compose.override.yml` that exposes the Traefik container ports on localhost; Docker Desktop should then make that available to you from windows as well.
