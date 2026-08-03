# SoftEther vpncmd

Put the Linux `vpncmd` executable and its matching `hamcore.se2` resource here
for the `platform-api` container:

```text
deploy/softether/vpncmd
deploy/softether/hamcore.se2
```

Both files are intentionally ignored by Git. They should come from the same
SoftEther VPN Server release that is running on the test server. For a server
installed under `/usr/local/vpnserver`, copy them with:

```bash
cp /usr/local/vpnserver/vpncmd deploy/softether/vpncmd
cp /usr/local/vpnserver/hamcore.se2 deploy/softether/hamcore.se2
chmod +x deploy/softether/vpncmd
```
