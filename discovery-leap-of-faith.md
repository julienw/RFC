
# Discovery using the cloud and leap-of-faith
## NUPNP + plex + LE
 * (less-secure option when QR codes cannot be used)
 * con: you need to trust not only the Box and the client, but also the bridge/DNS-server
 * pro: works well on a laptop or TV which does not support QR codes
 * con: lack of QR-code means trusting that no attacker is on local network during pairing phase

## Flow:
1. At build time, a serial number and a deploy token is put onto the box, so that it can request two services:
 * setting up a DNS entry,
 * signing a CSR.
2. Still at build time, the serial number is put into a string of the form "https://<serial-no>.knilxof.org/". This string is printed onto a sticker.
3. The sticker is put onto the physical Box. The Box is then packaged in such a way that you can't see the serial number sticker in the shop without opening the packaging.
4. The user unpacks the Box, and plugs it into their router using an ethernet cable.
5. The Box obtains a local IP address using DHCP, and generates a CSR.
6. The Box connects to the Bridge on e.g. https://register.knilfox.org and posts the following data:
 * its serial number,
 * the deploy token,
 * its CSR,
 * its local IP address.
7. The Bridge registers a LetsEncrypt cert, using the ACME DNS challenge (currently in staging, see https://community.letsencrypt.org/t/dns-challenge-is-in-staging/8322 ).
8. The Bridge adds an A record for <serial-no>.knilfox.org, pointing to the Box's local IP address.
9. The user can use the normal browser on their client (and no need for a QR code reader here).
10. The Box posts its long URL "https://<serial-no>.knilxof.org/" to https://discover.knilxof.org/I-am-a-Box.
11. The Bridge adds an entry to its database linking the outgoing IP address of the Box to its long URL.
12. The user types https://discover.knilxof.org/I-am-a-client.html into their browser's address bar, and the Bridge does a database lookup;
 * If there is exactly one Box registered from that same outgoing IP address, the user is redirected to https://<serial-no>.knilxof.org/
 * If there are no Boxes registered for that same outgoing IP address, an error is displayed.
 * If there are  >1 Boxes registered for that same outgoing IP address, the user is offered a choice, and told to select the serial-no that matches the one on the sticker on their Box, after which the user is redirected to https://<serial-no>.knilxof.org/ for the serial-no they selected.
13. The Box serves a page (accessible only on its local IP address) that requests the creation of a user on the Box (this has to be a html interface in this case).
14. User creation:
 * If the Box is still in setup mode, it accepts this request and gives the app a token with admin access,
 *  If an admin user already exists, this admin user is asked whether they want to allow the new user creation request.
15. If the Box gets assigned a new local IP address through DHCP, it posts this new IP address to the Bridge and we wait for DNS propagation.




