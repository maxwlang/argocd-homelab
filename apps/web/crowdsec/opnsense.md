# Configuring opnsense with blacklist
1. Create an LXC container on proxmox host using ubuntu
2. In the LXC container, add the crowdsec apt repository
    `curl -s https://packagecloud.io/install/repositories/crowdsec/crowdsec/script.deb.sh | sudo bash`
3. Install the `crowdsec-blocklist-mirror` bouncer
    `sudo apt install crowdsec-blocklist-mirror`
4. Enter the LAPI pod in kubernetes and add a bouncer entry for this
    `cscli bouncers add crowdsec-blocklist-mirror`
5. Copy the api key from the bouncers add command
6. In the LXC container, edit the config to point to the LAPI pod, add the api key, and allow external access
    `sudo nano /etc/crowdsec/bouncers/crowdsec-blocklist-mirror.yaml`
    ```
    ...
    crowdsec_config:
        lapi_key: "<key here>"
        lapi_url: "http://crowdsec-lapi.k8s.maxwlang.com:80"
    ...
    listen_uri: 0.0.0.0:41412
    ...
7. Re-install the bouncer to fix config
    `sudo apt install crowdsec-blocklist-mirror`
8. Reboot
9. Test bouncer
    `curl http://localhost:41412`
10. Add the bouncer to opnsense as an alias
    `<internal ip>:41412/security/blocklist`